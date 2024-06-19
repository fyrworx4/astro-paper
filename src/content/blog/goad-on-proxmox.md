---
title: Installing GOAD on Proxmox
author: taylor
pubDatetime: 2024-06-19T22:30:09Z
slug: goad-on-proxmox
featured: false
draft: first
tags:
  - proxmox
  - goad
  - infrastructure
description: How to install GOAD on Proxmox.
---

## Table of Contents

## Introduction

Mayfly has already made a [guide](https://mayfly277.github.io/posts/GOAD-on-proxmox-part1-install/) on installing GOAD on Proxmox. However, reading through the blog it seemed a bit too over complicated for my situation. For one, I have Proxmox installed in my homelab, so any NAT forwarding rules will be unnecessary to reach any of the internal boxes. I don't mind if some of my VMs are directly connected to my home network, and I also don't really care too much about proper firewall rules implementation and just want things to "work".

Therefore, many of the steps will be similar to Mayfly's guide, but with my own twist to suit my own situation.

## Network diagram

![](https://i.postimg.cc/4NbcwNKQ/SCR-20240619-kqnv.png)

The pfSense will allow us to create two segmented networks - one specifically for GOAD and the other for any other VMs we want to have access into GOAD, like attacker VMs and the provisioning container.

The provisioning container will contain the GitHub repo for GOAD and will create templates with Packer, create GOAD VMs with Terraform, and configure the GOAD VMs with Ansible. Therefore it will need to interact with the Proxmox's API as well as the VMs in the GOAD network.

## Install Proxmox

I won't cover the exact steps on installing Proxmox onto a bare metal machine, but you can refer to the [documentation](https://pve.proxmox.com/pve-docs/chapter-pve-installation.html) on the steps.

I left most settings as default, such as the node's name being `pve`. In this blog I'll refer to my node name as `pve`, but if you choose to customize your node's name then be sure to replace `pve` with your own value.

## Setting up Proxmox

Once logged into Proxmox, on "Server View", navigate to Datacenter -> pve -> System -> Network and add the 3 following devices:

**Linux Bridge for VLANs**

- Create -> Linux Bridge
  - Ensure that the name is `vmbr1`
  - VLAN aware: yes
  - Comment: "VLANs"

![](https://i.postimg.cc/cCsG0HXc/SCR-20240619-jkvp.png)

**VLAN 10**

- Create -> Linux VLAN
  - Name: `vlan10`
  - IP/CIDR: `10.0.0.5/24`
  - VLAN raw device: `vmbr1`
  - Comment: "Connection to GOAD"

![](https://i.postimg.cc/52VYDHYR/SCR-20240619-lmdu.png)

**VLAN 20**

- Create -> Linux VLAN
  - Name: `vlan20`
  - VLAN raw device: `vmbr1`
  - Comment: "GOAD"

![](https://i.postimg.cc/pV9bDp63/SCR-20240619-kryh.png)

Click on "Apply Configuration".

## Downloading required ISOs

We will need to get the following ISOs onto Proxmox:

- pfSense
- Windows Server 2019
- Windows Server 2016

We can use the "Download from URL" option to tell Proxmox to download the ISOs directly.

**pfSense**

For pfSense, use the following info:

- URL: `https://repo.ialab.dsu.edu/pfsense/pfSense-CE-2.7.2-RELEASE-amd64.iso.gz`
- File name: `pfSense-CE-2.7.2-RELEASE-amd64.iso.gz`

You can use the "Query URL" button to automatically populate the file name for this ISO.

![](https://i.postimg.cc/ZRy648wr/SCR-20240619-jznm.png)

**Windows Server 2016 and 2019**

For Windows Server 2019, use the below info:

- URL: `https://software-download.microsoft.com/download/pr/17763.737.190906-2324.rs5_release_svc_refresh_SERVER_EVAL_x64FRE_en-us_1.iso`
- File name: `windows_server_2019_17763.737_eval_x64.iso`

IMPORTANT! Make sure that the name of this file is exactly as shown. Don't click on the Query URL button!

![](https://i.postimg.cc/DZF7wFgB/SCR-20240619-kbyz.png)

Repeat the steps for Windows Server 2016:

- URL: `https://software-download.microsoft.com/download/pr/Windows_Server_2016_Datacenter_EVAL_en-us_14393_refresh.ISO`
- File name: `windows_server_2016_14393.0_eval_x64.iso`

After a couple of minutes, you should see the three ISOs.

## Setting up pools

Go to Datacenter -> Permissions -> Pools and create the following pools:

- VMs
- GOAD
- Templates

![](https://i.postimg.cc/RC24WcXW/SCR-20240619-kdog.png)

## Setting up pfSense VM

### Creating the VM

Create a new pfSense VM with the following settings:

- Name: pfSense
- Resource Pool: VMs
- ISO image: Select the pfSense ISO
- Disk size: 32 GB
- Cores: 2
- Memory (RAM): 1 GB
- Network: `vmbr0`
- Confirm

Once VM is completed, add a network:

![](https://i.postimg.cc/8cQXSPmz/SCR-20240619-kgfa.png)

We should see something like this:

![](https://i.postimg.cc/sDym3L7K/SCR-20240619-ksgn.png)

Select the bridge as `vmbr1` and leave the VLAN tag blank. We will configure VLANs through pfSense.

![](https://i.postimg.cc/nrKx8W7N/SCR-20240619-klci.png)

Start the VM. You can access the console through the "Console" tab, or by double-clicking the VM name.

Accept the first menu and then select "Install" to install pfSense.

![](https://i.postimg.cc/j5VBsXsT/SCR-20240619-klux.png)

I selected "Auto (ZFS)".

On the ZFS configuration page, keep hitting Enter until you see this:

![](https://i.postimg.cc/CMs9gk9y/SCR-20240619-kmlm.png)

Hit Space to select the disk, and then hit Enter again. Select "YES" to confirm, then hit Enter. Reboot the system once installation is complete.

When the console asks "Should VLANs be set up now?" hit Y.

For the parent interface name, type `vtnet1`.

For the VLAN tag, type in 10.

![](https://i.postimg.cc/Cxs4ZsHY/SCR-20240619-kpat.png)

Repeat the same process, but set the VLAN tag to 20.

![](https://i.postimg.cc/yNfXj6bb/SCR-20240619-kpev.png)

Then hit Enter to finish VLAN configuration.

Configure the interfaces as follows:

- WAN interface name: `vtnet0`
- LAN interface name: `vtnet1.10`
- OPT1 interface name: `vtnet1.20`

Then hit Y to confirm.

Once the pfSense reboots, we are greeted with the console menu.

![](https://i.postimg.cc/HntQJpXz/SCR-20240619-ktew.png)

### Configuring pfSense interfaces

Let's perform some more configuration on the interfaces.

**LAN**

- Select "2" to configure interface IP addresses.
- Select "2" to configure LAN's IP address.
- Select "n" on the DHCP question so that we don't use DHCP.
- For the LAN IPv4 address, use `10.0.0.1/24`.
- Hit Enter to not use an upstream gateway.
- Select "n" to not use DHCPv6.
- Leave IPv6 blank.
- Select "y" to set up DHCP server on LAN.
- Start range: `10.0.0.100`
- End range: `10.0.0.254`
- Hit "n" to not revert to HTTP.

**OPT1**

- Select "2" to configure interface IP addresses.
- Select "3" to configure OPT1's IP address.
- Select "n" on the DHCP question so that we don't use DHCP.
- For the LAN IPv4 address, use `192.168.10.1/24`.
- Hit Enter to not use an upstream gateway.
- Select "n" to not use DHCPv6.
- Leave IPv6 blank.
- Select "y" to set up DHCP server on LAN.
- Start range: `192.168.10.100`
- End range: `192.168.10.254`
- Hit "n" to not revert to HTTP.

We should see something like this now:

![](https://i.postimg.cc/zvMqFhCG/SCR-20240619-kvgs.png)

### Configuring firewall rules

To access the pfSense's web GUI, we can use SSH tunneling to access pfSense's port 443 from the Proxmox host.

```bash
ssh -L 8080:10.0.0.1:443 root@PROXMOX-IP
```

Once SSH'd in, navigate to https://localhost:8080 on your browser.

![](https://i.postimg.cc/j2yqhXXd/SCR-20240619-lneq.png)

Log in with default credentials `admin:pfsense`.

Keep hitting "Next" until you get to the "Configure WAN Interface" step. Uncheck both "Block RFC 1918 Private Networks" and "Block bogon networks".

![](https://i.postimg.cc/yYPsYWp5/SCR-20240619-lnqs.png)

Keep hitting next until you get to the "Set Admin WebGUI Password". Change the password to something more secure.

Then at the last step, click "Reload".

Go to Firewall -> Rules -> OPT1 and add a new rule:

![](https://i.postimg.cc/hGFB2xXX/SCR-20240619-lpdf.png)

Ensure that the following settings are applied to this rule:

- Action: Pass
- Interface: OPT1
- Protocol: Any
- Source: OPT1 subnets
- Destination: Any

### Configuring DNS

Under System -> General Setup, add a public DNS server like `1.1.1.1`:

![](https://i.postimg.cc/W463Rjq6/SCR-20240619-nnuy.png)

## Set up provisioning container

First, we download the CT template from Proxmox by going to the "local" storage -> CT Templates -> Templates -> Search "ubuntu" -> Select Ubuntu 22.04.

![](https://i.postimg.cc/6Q9XsKhK/SCR-20240619-lquf.png)

Create a new CT. Set the hostname to `provisioning` and set up an SSH password and key if you want.

![](https://i.postimg.cc/FHFYZMJf/SCR-20240619-lvsq.png)

Set the template to the Ubuntu 22.04 image you just downloaded.

![](https://i.postimg.cc/FK77t0pN/SCR-20240619-lvts.png)

I set my disk size to 20 GB, CPU size to 2 cores, and RAM to 1 GB.

My network settings are as follows:

- Name: `eth0`
- Bridge: `vmbr1`
- VLAN Tag: 10
- IPv4: Static
- IPv4/CIDR: `10.0.0.2/24`
- Gateway: `10.0.0.1`

![](https://i.postimg.cc/T3hRhfDc/SCR-20240619-lxgi.png)

And then confirm.

Post-creation, I added a new network device so that I can directly SSH to it from my home network:

![](https://i.postimg.cc/65bXkgm4/SCR-20240619-lxpq.png)

Let's start this container. We can double click the container to access the terminal or SSH into it.

Run the following command:

```bash
apt update && apt upgrade
apt install git vim gpg tmux curl gnupg software-properties-common mkisofs vim
```

If you can't hit the Internet, you can edit your DNS settings on the container.

![](https://i.postimg.cc/T1CKM0z2/SCR-20240619-mbsb.png)

### Installing Packer

Packer is a tool to create templates from configuration files. We will be using Packer to create Windows Server 2016 and 2019 templates.

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install packer
```

Run the following to confirm that Packer has been installed:

```bash
packer -v
```

![](https://i.postimg.cc/tCmg32Fy/SCR-20240619-mple.png)

### Installing Terraform

Terraform is a tool to create infrastructure (such as VMs) from configuration files. We will be using Terraform to deploy the Windows Server VMs.

```bash
# Install the HashiCorp GPG key.
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Verify the key's fingerprint.
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint

# add terraform sourcelist
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
tee /etc/apt/sources.list.d/hashicorp.list

# update apt and install terraform
apt update && apt install terraform
```

Run the following to confirm that Terraform has been installed:

```bash
terraform -v
```

![](https://i.postimg.cc/7YHB3LXS/SCR-20240619-mejv.png)

### Installing Ansible

Ansible is an automation tool used to configure batches of machines from a centralized server. We will be using Ansible to configure the GOAD vulnerable environment.

```bash
apt install python3-pip
python3 -m pip install --upgrade pip
python3 -m pip install ansible-core==2.12.6
python3 -m pip install pywinrm
```

Run the following commands to confirm that Ansible and Ansible Galaxy have been installed:

```bash
ansible-galaxy --version
ansible --version
```

![](https://i.postimg.cc/MHWYms0z/SCR-20240619-mekr.png)

### Cloning the GOAD project

Next, clone GOAD from GitHub.

```bash
cd /root
git clone https://github.com/Orange-Cyberdefense/GOAD.git
```

## Creating templates with Packer

First, we download cloudbase-init on the provisioning container:

```bash
cd /root/GOAD/packer/proxmox/scripts/sysprep
wget https://cloudbase.it/downloads/CloudbaseInitSetup_Stable_x64.msi
```

We then create a new user on Proxmox dedicated for GOAD deployment:

```bash
# create new user
pveum useradd infra_as_code@pve
pveum passwd infra_as_code@pve

# assign Administrator role to new user lol
pveum acl modify / -user 'infra_as_code@pve' -role Administrator
```

### Editing HCL configuration files

On the provisioning container, copy the config file.

```bash
cd /root/GOAD/packer/proxmox/
cp config.auto.pkrvars.hcl.template config.auto.pkrvars.hcl
```

Edit the configuration file with the password of the `infra_as_code` user, as well as the other variables if necessary.

![](https://i.postimg.cc/VkYdsxff/SCR-20240619-miiy.png)

### Preparing the ISO files

```bash
cd /root/GOAD/packer/proxmox/
./build_proxmox_iso.sh
```

This will create the ISO files that will be used by Packer to build our templates. The ISO files contain the `unattend.xml` files that will perform auto configuration of the VMs to be used as templates.

On Proxmox, use `scp` to copy over the ISO files to Proxmox.

```bash
scp root@10.0.0.2:/root/GOAD/packer/proxmox/iso/scripts_withcloudinit.iso /var/lib/vz/template/iso/scripts_withcloudinit.iso
```

While on Proxmox, download virtio-win.iso, which contain the Windows drivers for KVM/QEMU.

```bash
cd /var/lib/vz/template/iso
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

Afterwards, we can check that the ISOs are downloaded properly:

![](https://i.postimg.cc/8PvY4C65/SCR-20240619-mkzh.png)

### Modifying Packer configuration files

Modify `packer.json.pkr.hcl`. On line 43, change `vmbr3` to `vmbr1`, as well as `vlan_tag` to `20`:

![](https://i.postimg.cc/rs1Sd7D8/SCR-20240619-mmhy.png)

Modify `windows_server2019_proxmox_cloudinit.pkvars.hcl` so that `vm_disk_format` is set to `raw`, and ensure that the ISO name matches with the one downloaded onto Proxmox:

![](https://i.postimg.cc/zBfMFSCZ/SCR-20240619-mqzz.png)

Do the same for `windows_server2016_proxmox_cloudinit.pkvars.hcl`.

### Running Packer

Run Packer against the Windows Server 2019 image:

```bash
packer init .
packer validate -var-file=windows_server2019_proxmox_cloudinit.pkvars.hcl .
packer build -var-file=windows_server2019_proxmox_cloudinit.pkvars.hcl .
```

It takes a while for the VM to be set up.

Afterwards, run Packer for Windows Server 2016:

```bash
packer validate -var-file=windows_server2016_proxmox_cloudinit.pkvars.hcl .
packer build -var-file=windows_server2016_proxmox_cloudinit.pkvars.hcl .
```

_Note: The template will be configured to use the French keyboard by default_

## Creating GOAD VMs with Terraform

Now, let's use Terraform to build out the VMs for our environment.

On the provisioning box, make a copy of the `variables.tf`:

```bash
cd /root/GOAD/ad/GOAD/providers/proxmox/terraform
cp variables.tf.template variables.tf
```

We change the following variables:

- `pm_api_url`: `https://10.0.0.5:8006/api2/json`
- `pm_password`: the password to the `infra_as_code` user
- `pm_node`: name of node
- `pm_pool`: leave as GOAD
- `storage`: `local-lvm`
- `network_bridge`: `vmbr1`
- `network_vlan`: 20

Also be sure to change the IDs of the templates (may be different depending on how many VMs you had before):

- `WinServer2019_x64`: 104
- `WinServer2016_x64`: 105

![](https://i.postimg.cc/RFKWZfYf/SCR-20240619-nfdh.png)

### Running Terraform

Run Terraform with the following commands. This will create 5 new Windows VMs for GOAD.

```bash
terraform init
terraform plan -out goad.plan
terraform apply "goad.plan"
```

## Configuring GOAD with Ansible

Now, what is left to do is to run Ansible.

```bash
cd /root/GOAD/ansible
ansible-galaxy install -r requirements.yml
export ANSIBLE_COMMAND="ansible-playbook -i ../ad/GOAD/data/inventory -i ../ad/GOAD/providers/proxmox/inventory"
../scripts/provisionning.sh # yes the file name is misspelled...
```

This can take a couple hours.

---
author: taylor
slug: spf-dkim-dmarc
featured: false
draft: false
tags:
  - email
  - spf
  - dkim
  - dmarc
description: DMARC is weird.
title: The Holy Trinity of Email Security - SPF, DKIM, and DMARC
pubDatetime: 2024-12-08T05:18:57Z
---

If you search up "DMARC" on Google, you get pretty generic descriptions from email security companies that DMARC "enhances security" by implementing "anti-spoofing" measures. These sites also attempt to pitch their own DMARC analyzer/security tool to provide "advanced malware protection" against attackers.

Yes I'm looking at you, Mimecast.

While you can get a generic idea of how it works/why it is important, there is one flaw with these posts - they don't explain what's really going on behind the scenes. What email security headers are being analyzed to verify DMARC? How does it work with SPF and DKIM? Sometimes, why does DMARC fail even when SPF and DKIM pass?

Let's dive into it, shall we?

[I have a TLDR at the bottom](#to-summarizetldr)

## Table of Contents

## What is DMARC?

DMARC stands for Domain-based Message Authentication, Reporting, and Conformance. DMARC uses SPF and DKIM to perform validation on incoming email messages. Depending on the output of SPF and DKIM checks, DMARC will either accept or reject incoming emails.

DMARC records can be found in a TXT record for the `_dmarc` subdomain of a given domain. For example, if I wanted to get the DMARC record for `steampowered.com`, I would query for a TXT record at `_dmarc.steampowered.com`:

```
v=DMARC1; p=reject; rua=mailto:e8b9a9244a5c591@rep.dmarcanalyzer.com; ruf=mailto:e8b9a9244a5c591@for.dmarcanalyzer.com; pct=100; sp=reject; fo=1;
```

Here's what each of the items mean in the above DMARC record:

- `v=DMARC1` - included in every DMARC record
- `p=reject` - the DMARC policy (in this case, reject the email)
- `rua=mailto:` - the email to send aggregate (summary) reports of DMARC checks.
- `ruf=mailto:` - the email to send forensic (detailed) reports of DMARC checks
- `pct=100` - the percentage of emails to go through a DMARC check
- `sp=reject` - the DMARC policy for subdomains (in this case, reject)
- `fo=1` - generate a report to the RUA whenever either SPF or DKIM do not produce an "aligned pass" result. (We'll go over this later too)

The most important item here is the `p` field, which stands for "policy". It's the action to take whenever DMARC fails. The possible values for `p` are:

- `p=none` - don't perform any action against email that failed DMARC checks
- `p=quarantine` - quarantine email that failed DMARC checks by placing it into recipient's junk folder
- `p=reject` - don't deliver email that failed DMARC checks

A missing DMARC record or a DMARC policy of `none` can be abused to allow attackers to perform email spoofing attacks. However, if the `none` policy is applied, email messages that fail DMARC validation can still be tracked, and most email clients will flag the unverified email message as malicious.

Before we dive a bit deeper into DMARC, let's first discuss about SPF and DKIM.

## What is SPF?

SPF stands for Sender Policy Framework. It lays out the guidelines on who is allowed to send email from a specific domain. You can think of it like a firewall ACL for email messages of sorts.

SPF is configured in the form of a TXT record of a given domain. Here's what the SPF record looks like for `steampowered.com`:

```
v=spf1 mx ip4:72.165.61.134/31 ip4:208.64.202.32/27 -all
```

And each of the items:

- `v=spf1` - included in every SPF record
- `mx` - refers to the IP address of sender is associated with the MX record of the domain.
- `ip4:72.165.61.134/31` - refers to the IP range of `72.165.61.134/31`
- `ip4:208.64.202.32/27` - refers to the IP range of `208.64.202.32/27`
- `-all` - allows emails to be delivered if the sender IP address matches any of the IP addresses listed in the record, and rejects emails sent from non-listed IP addresses.

Each of the items in the SPF record are called "mechanisms". So you would refer to `mx` as the `mx` mechanism, `ip4` as the `ip4` mechanism, and so on. In general, they're referred to as "tags" but in SPF they're specifically "mechanisms".

You may see `~all` or `+all` or `?all` in other SPF records. The character behind the `all` are referred to as qualifiers or prefixes. Here's what they mean:

- `-all` - also known as a hardfail, emails that don't match the allowed IP addresses are rejected
- `~all` - also known as a softfail, emails that don't match the allowed IP addresses are still delivered but are flagged with an SPF fail check
- `+all` - emails that don't match the allowed IP addresses still get delivered anyways
- `?all` - don't perform SPF verification at all and deliver the email anyways

This is how SPF works in a nutshell:

1. An email server receives an incoming email message.
2. The email server looks at the domain of the `Return-Path` header of the email message.
3. The email server retrieves the SPF record of the `Return-Path` domain.
4. If the source IP of the email message matches the IP addresses specified within the SPF record, then the email message is said to be **SPF authenticated**.
5. The email server will then decide how to deliver the email based on the SPF record's `all` mechanism.

A key step to point out is that SPF uses the `Return-Path` header, which is the email used for things like bounced messages. Why does SPF use the `Return-Path` header? I don't really know.

## What is DKIM?

DKIM stands for DomainKeys Identified Mail. It ensures that the sender of an email address is who they actually are, as well as that the email message has not been modified in transit. This is done with digital signatures and asymmetric key encryption to achieve this.

When an email server is configured to use DKIM, it will sign the email message with a private key stored somewhere locally on that server. The signed emails also contain a `DKIM-Signature` header, which contains encrypted hashes of some of the email headers (configured on the email server) and the body of the message.

This is the DKIM record associated with `steampowered.com`:

```
v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDESbiiHOv8WD5RPZ8UAYkS+rmxWyWV7l+g5jtfkZEKsOokamC9RnRHBlZiKucmA1o0ffmg+Z8KAFjf4Yy9OL1OGEO20F3UfMACffbJNsun67J3V7XWBsJwczEsJ21rmAhF0c9ntyg6yGAxwiYfzONhi0WKuN1QLslFjBRcMLoIrQIDAQAB
```

- `v=DKIM1` - included in every DKIM record
- `k=rsa` - the type of key being used in the `p` field
- `p=MIGf...` - the DKIM public key used to decrypt digital signatures

In order to query the DKIM record, you would need to know the selector that is being used. The `selector` is a value that is included in the DKIM signature to specify the location of the DKIM record.

Therefore, the DKIM record would be located at `[selector]._domainkey.<domain>`.

The `selector` can be any arbitrary string of text. For example, if I want my `selector` to be `deeznuts` then my DKIM TXT record will be located at `deeznuts._domainkey.tsnguyen.com`. The sending email server then must be configured to use `deeznuts` as the `selector` of the DKIM signature.

This is how DKIM works:

1. The sending email server computes the hashes of email headers and body.
2. The sending email server encrypts the hash with a DKIM private key (stored locally on the server).
3. Sending email server includes the encrypted hash into a DKIM signature, along with other information such as the hashing algorithm used and the domain and selector to use to query DKIM records.
4. Email is sent to client, along with the DKIM signature (that contains the encrypted hash).
5. The client email server sees the DKIM signature and obtains the DKIM public key from querying the DKIM record
   - The client email server knows that selector and domain to use when querying the DKIM record, via the `s=` and `d=` fields in the `DKIM-Signature` header, respectively.
6. The client email server decrypts the hash from DKIM signature using the DKIM public key.
7. The client email server generates its own hash of the email headers and body.
8. If the hashes are the same, the client email server verifies that the email and sender are valid, and the email is said to be **DKIM authenticated**.

There's a lot more that happens underneath the hood within DKIM! But that can be saved for a separate post.

### Extra DKIM notes

There's some terminology when it comes to defining DKIM architecture, including _DKIM signers_ and _DKIM verifiers_.

- DKIM signers usually refer to the systems within the email chain that are responsible for performing the hashing and encrypting of email messages.
- DKIM verifiers usually refer to the systems on the receiving end of the email message that decrypt and compare the hashes. They are used more often describe the DKIM process rather than describing individual entities or systems.

## DMARC Alignment

We've talked about how email messages can be SPF and DKIM authenticated. But DMARC throws in another term called **alignment**, which is super important because it's actually the **alignment** that determines whether DMARC passes or fails.

According to RFC 7489, an authenticated identifier is a domain that has been validated using either SPF or DKIM. That would be the `Return-Path` value for SPF and the `d=` value for DKIM.

DMARC identifier alignment occurs when the domain authenticated by SPF and/or DKIM matches the `From:` header of the email message.

Why "and/or"? Because DMARC only requires **one** of the two authentication mechanisms to pass and align with the sending domain.

That means if SPF passes but DKIM fails, as long as the domain used by SPF matches with the domain in the `From:` field, then DMARC passes as well. Now if SPF and DKIM both pass but DMARC fails, then that means that the domain in the `From:` field may not match either of the domains evaluated by SPF and DKIM.

In other words, DMARC checks are validated when:

- The domain passes authentication - the message passes at least one authentication method, being SPF or DKIM
- The domain passes alignment - the domain authenticated from SPF and/or DKIM matches the domain in the `From:` field of the message

## Example of SPF and DKIM passing but DMARC failing

Let's look at the following email message:

![](https://i.postimg.cc/fyp8GjKQ/headers-1.png)

It seems that SPF and DKIM both pass but DMARC fails. Let's take a closer look at the email headers.

![](https://i.postimg.cc/VsVZ0mcZ/Email-Verification-Headers.png)

If we only look at the important headers, we can extract the following domains used for each header:

- `Return-Path` header (Used for SPF) - `amazonses.com`
- `d=` value from `DKIM-Signature` header (Used for DKIM) - `amazonses.com`
- `From:` header - `gmail.com`

SPF uses the domain of the `Return-Path` header to perform the SPF check. If we look at the `Return-Path` of this email message, we can see that it is `amazonses.com`. Similarly, if we look at the domain used to validate DKIM, it is also `amazonses.com`, listed as the `d=` field in the `DKIM-Signature` header.

However, our `From` address is a `gmail.com` domain. Because `gmail.com` is different then the SPF- and DKIM-validated domain of `amazonses.com`, DMARC authentication fails.

## To summarize/TLDR

- SPF specifies which IP addresses are authorized to send email on behalf of a domain
- DKIM signs emails using public and private key pairs to verify message authenticity
- DMARC validates emails by checking:
  - That either SPF, DKIM, or both checks pass
  - That the authenticated domain (from SPF's `Return-Path` or the `d=` field in DKIM's signature) matches the `From:` header

## Other Observations

### DKIM-signed emails may still be marked as illegitimate

Let's say in this case, SPF and DMARC is valid, and the email message is DKIM-signed. So a signed email = good email. Sometimes not. Why might the email be unable to deliver?

Possible reasons:

- The private key does not match with the public key, and the signature cannot be properly obtained due to incorrect keys
- Different selectors/domains are used, and the email server cannot obtain the public key to verify message authenticity
- Email security solutions can perform additional inspection on email headers that go beyond SPF/DMARC checks

## References

Unlike most blogs that discuss about DMARC, these are actually pretty good:

- [Dmarcian - What is SPF?](https://dmarcian.com/what-is-spf/#:~:text=With%20an%20SPF%20record%20in,on%20behalf%20of%20your%20domain.)
- [Dmarcian - What is DKIM?](https://dmarcian.com/what-is-dkim/)
- [Dmarcian - The Difference in DMARC Reports: RUA and RUF](https://dmarcian.com/rua-vs-ruf/)
- [Dmarcly - What is DMARC Identifier Alignment (domain alignment)?](https://dmarcly.com/blog/what-is-dmarc-identifier-alignment-domain-alignment)

It's also always interesting to read the RFCs as well! I'd highly recommend it since they are the original "source" of the technologies:

- [RFC 7208 - SPF](https://www.rfc-editor.org/rfc/rfc7208.html)
- [RFC 6376 - DKIM](https://www.rfc-editor.org/rfc/rfc6376.html)
- [RFC 7489 - DMARC](https://www.rfc-editor.org/rfc/rfc7489.html)

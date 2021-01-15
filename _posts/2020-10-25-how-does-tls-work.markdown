---
layout: post
title:  "How does TLS work?"
date:   2020-10-25
category: mutual-tls
tags: tls ssl https pki
---

## Intro
Hello, world! I've been doing some reading around zero-trust networks and 
thought it would be fun/educational to implement mutual TLS between 
microservices, as it's not something I've come across before. I've also been 
trying to familiarise myself with Hashicorp's suite of tools (outside of  
Terraform, which I use a lot!). A colleague mentioned having used Vault to issue
TLS certificates, so I thought it would be a good project to combine the two!

This post will (hopefully) be the first in a series detailing setting up mutual 
TLS authentication between RESTful web services, using [Hashicorp Vault](https://www.vaultproject.io)
as a privately hosted certificate authority.

First, however, I thought it would be best to go through the basics of how TLS
works (as I wasn't 100% sure myself!).

## How does TLS work?

### Assumed Background
Throughout this article, I'll assume a basic understanding of the concepts
behind symmetric and asymmetric key cryptography, cryptographic signatures, and
networking.

### What is TLS?
TLS (or SSL<sup>*</sup>) is a network protocol used to encrypt traffic, whilst 
also allowing the client to verify the identity of the server (and also vice 
versa in the case of mutual TLS - more on that later!).

<sup>*</sup>older versions of the TLS protocol were initially called SSL. The 
terms are often used interchangeably.

### Certificates
Certificates are key to making TLS work. Certificates are files associated with a 
public/private keypair, which contain the following information:
- The domain name the certificate was issued to
- Or the person/organisation/device/etc it was issued to
- The certificate authority it was issued by
- Any associated subdomains the certificate is also valid for
- An issue date and expiration date
- The public key of the keypair (used by the client to encrypt data during [the handshake](#the-handshake))
- A cryptographic signature, signed using the private key of another certificate

And some more metadata. To have a look at the information contained in the TLS 
certificate used to secure your browser's communication with this website, click 
the padlock in the browser's address bar.

Certificates are typically hosted on a server, and will be sent to the client 
when the connection is first initiated. The client can then use the certificate 
to verify the identity of the server, using the help of a certificate authority.

### Certificate Authorities
A certificate authority (CA) is essentially a repository of TLS certificates. Every 
SSL certificate belongs to (is issued by) a specific CA. When a client is 
presented with a certificate by a server (which we'll call certificate A), it does
the following:
- Download the certificate that was used to sign certificate A (call this 
certificate B) from it's issuing CA
- Use certificate B's public key to verify that certificate A was in fact signed 
by certificate B
- Download the certificate that was used to sign certificate B (which may have 
been issued by a different CA), and repeat

![Certificate Authorities Diagram](/assets/tls/CertificateAuthorities.png)

This process continues until we reach a certificate that hasn't been signed by 
any other certificate; the root certificate.

If the root certificate has been issued by a trusted root certificate authority, 
the whole chain of certificates is trusted, and hence we have validated the 
identity of certificate A.

Root certificate authorities are those which are explicitly trusted by the
"root store" of the environment the client is running in. For example, 
Microsoft own a root store that comes pre-installed on every windows machine.

If the certificate authority that issued the root certificate is not in the 
client's root store, then none of the certificates in the chain are trusted, and 
the client is unable to verify the server's identity.

Certificate authorities are generally ran by large companies that undergo 
extensive security audits, hence why they are trusted to validate identities.

### The Handshake
TLS uses symmetric encryption to encrypt session traffic, meaning a shared secret
key (the session key) is required. That's where the handshake comes in. It is 
initiated by the client, and is the process by which the client and server 
securely agree on the session key.

There are various different key exchange methods supported by TLS; the most 
commonly used is RSA. It goes something like:

![TLS flow diagram](/assets/tls/TLS.png)

- The client sends a "hello" request to the server, including a list of key
exchange methods (cryptographic suites) it supports, as well as a random string 
of bytes called the client random
- The server responds with its chosen suite (RSA in this case), its TLS 
certificate, and another random string of bytes called the server random
- The client then verifies the identity of the server with a certificate authority
- Provided the certificate checks out, the client generates another random string
of bytes (the premaster key), encrypts it using the server's public key from the
certificate, then sends it to the server
- The server can then decrypt the premaster key using it's private key
- Both the client and server now calculate the session key using the client 
random, the server random, and the premaster key. They should arrive at the same
result, so the session key can now be used for symmetric key encryption
- To verify that the process worked, the client encrypts a "ready" message using
the session key, and sends it to the server
- The server checks that is can decrypt the "ready" message, and sends back a 
"finished" message (also encrypted using the session key)
- The handshake is now finished, and the session key can be used to
encrypt/decrypt all traffic between the client and server

### Mutual TLS
Mutual TLS (mTLS) is a variant of TLS in which the client also presents the 
server with a certificate after initiating the handshake. This allows the server 
to verify the identity of the client, as well as the other way around, adding an
extra layer of security. This is particularly useful in zero-trust networks, 
where clients must prove their identity to servers.

Mutual TLS is usually used in corporate networks for application-to-application
communication, where the certificates used are issued by a private CA that lives 
within the network.

## Roadmap
The [next post](/mutual-tls/2020/11/07/vault-as-a-ca.html) in this series will 
look at setting up Vault to act as a private CA. Then we'll look at using that 
CA to issue certificates to microservices, which will authenticate with one 
another using mTLS.

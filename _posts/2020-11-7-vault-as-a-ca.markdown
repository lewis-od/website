---
layout: post
title:  "Setting up Vault as a Certificate Authority"
date:   2020-11-07
category: mutual-tls
tags: hashicorp vault certificate-authority tls pki
---

Following on from my last post about TLS, in this post we'll look at setting up 
HashiCorp Vault as a certificate authority. The following post will then focus 
on setting up mTLS between 2 microservices using said CA.

### Prerequisites
To follow along with this article you'll require [Terraform](https://www.terraform.io) 
to be installed on your machine, as well as a basic understanding of what 
Terraform does and how it works.

If you aren't familiar with Terraform, there are some great introductory 
tutorials on the [HashiCorp Learn](https://learn.hashicorp.com/terraform) 
website.

I'm also going to assume you know how TLS works. If you don't, then take a look 
at my [previous post](/mutual-tls/2020/10/25/how-does-tls-work.html) 😉.

### Installing Vault
The first step is obviously to install Vault; you can do that by following the 
instructions [here](https://learn.hashicorp.com/tutorials/vault/getting-started-install).
I'd also recommend following the rest of that tutorial if you aren't already 
familiar with the basic concepts behind Vault.

Start the dev server in a terminal by running:
```sh
vault server -dev
```
and take note of the root token.

Then open a new terminal window, and authenticate with vault by running:
```sh
export VAULT_ADDR=http://localhost:8200
vault login <root-token>
```

### Configuring Vault with Terraform
Vault has a [Terraform provider](https://registry.terraform.io/providers/hashicorp/vault/latest/docs)
that lets you specify your configuration in a declarative manner, and makes
automating the configuration of your server pretty simple. This is what we'll use
to set up our certificate authority.

Firstly create a directory to hold your terraform code, and create a file with 
the following contents:
```terraform
provider "vault" {
  version = "~> 2.15"
  address = "http://localhost:8200"
}
```
The provider will connect to the server at the specified address using the token 
in the file at `~/.vault-token`. This file is created for you by the Vault CLI 
when you run `vault login`, so you should already have one that's populated with
the correct token.

Run `terraform init` to download the provider, and you should be ready to go!

### Setting up a CA
The first thing to do is mount the [PKI secrets engine](https://www.vaultproject.io/docs/secrets/pki),
which is what will be generating our certs for us. We can mount it at `/pki` 
with the Terraform:
```terraform
resource "vault_mount" "pki" {
  path                  = "/pki"
  type                  = "pki"
  max_lease_ttl_seconds = 315360000 // 10 years
}
```

We now need to use this to issue a root certificate to use for our CA:
```terraform
variable "base_domain" {
    type        = string
    description = "The domain name the CA will issue certificates for"
    default     = "localhost"
}

resource "vault_pki_secret_backend_root_cert" "root" {
  backend     = vault_mount.pki.path
  type        = "internal"
  common_name = "${var.base_domain} Certificate Authority"
  ttl         = "87600h" // 10 years
}
```
Here I've used `localhost` for the domain name, since in the next post we'll 
look at setting up mTLS for microservices running on your local machine. The 
common name of the root certificate actually doesn't matter, but I've declared 
the `base_domain` variable here as we'll need it later.

When Vault issues certificates, they contain 2 special URLS:
- The URL of the CA itself, used by clients to download certificates
- The URL of the certificate revocation list (CRL), used by clients to download
a list of certificates that have been manually revoked before their expiration 
date

We need to configure both of these URLs, and can do so with:
```terraform
resource "vault_pki_secret_backend_config_urls" "config_urls" {
  backend                 = vault_mount.pki.path
  issuing_certificates    = ["http://localhost:8200/v1/pki/ca"]
  crl_distribution_points = ["http://localhost:8200/v1/pki/crl"]
}
```
Again I've used `http://localhost:8200` as this is where the client and server 
apps we write in the next post will be able to access Vault. In the real world,
this would be wherever your vault server is deployed (e.g. 
`https://vault.yourcompany.com`).

And that's it for our root CA! Run `terraform apply` and everything should be 
happy.

### Roles
Now our CA is set up, we're almost ready to issue some certificates. First 
however, we need to declare some roles that are able to issue certificates. We'll
create 2 roles: one for issuing certificates for servers, and one for issuing 
certificates to users.

For the server role, add the following:
```terraform
resource "vault_pki_secret_backend_role" "server_role" {
  backend = vault_mount.pki.path
  name    = "server"
  max_ttl = "72h"

  allowed_domains    = [var.base_domain]
  allow_any_name     = false
  allow_glob_domains = true
  allow_subdomains   = true
  enforce_hostnames  = true

  client_flag = false
  server_flag = true
}
```
You can tweak the block of settings in the middle if you wish. Finally the 
client role:
```terraform
resource "vault_pki_secret_backend_role" "client_role" {
  backend = vault_mount.pki.path
  name    = "client"
  max_ttl = "72h"

  allowed_domains    = [var.base_domain]
  allow_any_name     = false
  allow_bare_domains = true // Required for email addresses
  allow_glob_domains = false
  allow_ip_sans      = true
  allow_subdomains   = false
  enforce_hostnames  = true

  client_flag = true
  server_flag = false
}
```
This can be used to issue certs to users, where the common name is the user's
email address. Note that the domain name of the user's email must be in the 
`allowed_domains` list, so in this case we can only issue certificates with names 
like `username@localhost`.

Run one final `terraform apply` to create the roles, and we're done with the 
Terraform. You now have a fully functioning certificate authority!

### Issuing our first certificates
To issue certificates we'll be using the Vault CLI. However it's worth noting 
that this can also be done using Vault's REST API, or even some more Terraform.

To issue a certificate for a server, run:
```sh
vault write /pki/issue/server common_name=localhost
```
This should issue the certificate, and print the contents of the certificate along
with its associated private key to your terminal.

Similarly, to issue a client certificate, we can run:
```sh
vault write /pki/issue/client common_name=lewis@localhost
```

**Tip**: You can add `-format=json` to the end of the issue command, then use
something like [jq](https://stedolan.github.io/jq/) to parse it:
```sh
vault write /pki/issue/server \
    common_name="localhost" \
    -format=json > certificate.json
jq -r .data.private_key certificate.json > key.pem
jq -r .data.certificate certificate.json > cert.pem
rm certificates.json
```

### Caveats
With this setup, all of the certificates issued by our CA are signed by the root
certificate. This is generally a bad idea; if the certificate authority somehow 
became compromised, we'd have to revoke all certificates issued by the CA, set up
the CA using a new root cert, then re-distribute this root cert to all of our 
clients.

To get around this, it's common to set up another CA called an intermediate 
authority. The root CA signs the intermediate CA's certificate, then that is 
used to sign all of our client and server certificates. This way, if the 
intermediate authority becomes compromised, we just set up another one using a 
new certificate that's also signed by the root. We avoid having to re-distribute 
a new root certificate, which could be a pain.

Vault natively supports running as an intermediate authority, where the root CA
is either: the same Vault server (bad idea!); another Vault server; or an 
external, non-Vault, CA. For more info, see this 
[article](https://learn.hashicorp.com/tutorials/vault/pki-engine?in=vault/secrets-management) 
on the HashiCorp website.

It's also worth nothing that for the purposes of running locally, we've been
running the Vault server in dev mode, and haven't set up any proper 
authentication, RBAC, etc. Definitely don't do that in production!

### Conclusion
That's all folks!

The Terraform code for configuring Vault written for this post can be found on
Github [here](https://github.com/lewis-od/vault-mtls/tree/master/terraform).
Note that this has been extended to include an intermediate authority, as 
described above.

In the next post we'll take a look at using our newly 
configured CA to perform mTLS between a client and server application.

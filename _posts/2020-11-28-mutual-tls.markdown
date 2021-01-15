---
layout: post
title:  "Mutual TLS between microservices"
date:   2020-11-28
category: mutual-tls
tags: hashicorp vault go microservices tls pki
---

This post is the 3rd in a series on mutual TLS. You can find part one 
[here](/mutual-tls/2020/10/25/how-does-tls-work.html), and part 2 
[here](/mutual-tls/2020/11/7/vault-as-a-ca.html).

In this post, we'll be looking at using our newly configures certificate authority
to set up mutual TLS authentication between two apps written in Go. These will 
be a basic HTTPS server with a hello world endpoint, and a client to call that 
endpoint.

The code accompanying all 3 blog posts can be found on Github
[here](https://github.com/lewis-od/vault-mtls).

### Prerequisites
- Basic Golang knowledge, and the Go CLI installed
- Vault running and configured as a CA (as detailed in the previous post)

### Simple HTTP server and client
We'll first start with basic HTTP client/server apps without any TLS. We can then
add one-way TLS, and finally upgrade to mutual TLS.

First create a directory this project, and within that a directory for the 
client, and a directory for the server. Within the server directory, create a 
file called `server.go` with the following contents:
```go
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/hello", helloHandler)
	log.Println("Starting server on port 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "Hello, world!\n")
}
```
This will start a HTTP server on port 8080, with a single `/hello` endpoint. 
From the `server` directory, run `go run server.go` to start the server. In a 
new terminal window, running
```
curl http://localhost:8080/hello
```
should print out `Hello, world!`. That's the server sorted, now for the client.

Create the following file at `client/client.go`:
```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"time"
)

func main() {
	client := &http.Client{}
	for {
		body := fetchEndpoint("http://localhost:8080/hello", client)
		fmt.Println(string(body))
		time.Sleep(1 * time.Second)
	}
}

func fetchEndpoint(url string, client *http.Client) []byte {
	r, err := client.Get(url)
	if err != nil {
		log.Fatal(err)
	}

	defer r.Body.Close()
	body, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Fatal(err)
	}

	return body
}
```
With the server still running, run `go run client.go` from the `client` 
directory, and you should see `Hello, world!` printed out once every second.
Sorted!

### Adding one-way TLS
Now for the fun stuff: adding TLS. Firstly we need to issue a certificate and 
private key for our server to use. We can do this by running the command:
```
vault write /pki/issue/server common_name=localhost
```
This will print out a bunch of stuff, including the certificate and its 
corresponding private key. Copy the certificate (including the
`-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` parts) to a new 
file at `server/certs/cert.pem`, and the private key (again, including the 
`-----BEGIN RSA PRIVATE KEY-----` bits) to `server/certs/key.pem`.

Now we need to tell the server to use this certificate and private key for TLS.
This is as simple as changing the line in `server.go`:
```go
log.Fatal(http.ListenAndServe(":8080", nil))
```
to
```go
log.Fatal(http.ListenAndServeTLS(":8080", "certs/cert.pem", "certs/key.pem", nil))
```
Restart the server to pick up the changes, then in `client.go` change 
`http://localhost:8080/hello` to `https://localhost:8080/hello`. Now run the
client and... it fails:
```
Get "https://localhost:8080/hello": x509: certificate signed by unknown authority
exit status 1
```
This is because our CA is using a self-signed cert to sign the certs it issues, 
which isn't in our machine's built-in trust-store. Therefore the client app is 
unable to verify the validity of the certificate being presented by the server, 
and throws an error.

To fix this, we need to tell the client to trust the CA's signing certificate. It 
will then be able to validate any certificates that our CA issues. Vault hosts 
the CA's signing cert at the `/v1/pki/ca/pem` endpoint, so download it with:
```
mkdir client/certs
curl http://localhost:8200/v1/pki/ca/pem > client/certs/root.pem
```
Your directory structure should now look like this:
```
.
├── client
│   ├── certs
│   │   └── root.pem
│   └── client.go
└── server
    ├── certs
    │   ├── cert.pem
    │   └── key.pem
    └── server.go
```

Add the following function to `client.go`:
```go
func createCertPool(certPath string) *x509.CertPool {
	rootCert, err := ioutil.ReadFile(certPath)
	if err != nil {
		log.Fatal(err)
	}
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(rootCert)

	return certPool
}
```
This will create a `CertPool`, essentially just a set of certificates, which we 
can then tell the client to trust. We can use this in the `main` function when 
creating the HTTP client:
```go
func main() {
	certPool := createCertPool("certs/root.pem")
	client := &http.Client{
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{
				RootCAs: certPool,
			},
		},
	}
	for {
		body := fetchEndpoint("https://localhost:8080/hello", client)
		fmt.Printf("%s\n", body)
		time.Sleep(1 * time.Second)
	}
}
```
Now `go run client.go` should again print out `Hello, world!` every second. 
We've successfully set up one-way TLS; no eavesdroppers will be able to see the 
contents of our top secret hello world messages!

### Adding mutual TLS
And now time for the bit we're all here for: mutual TLS. For this, we'll need to
make the following changes:

1. Tell the server to require that clients also present TLS certificates
1. Tell the server to trust the CA's signing cert, the same way we did for the
client
1. Issue a certificate for the client, and tell it to present this to the server
when initiating a connection

For part 1, change the `main()` function in `server.go` to:
```go
func main() {
	tlsConfig := &tls.Config{
		ClientAuth: tls.RequireAndVerifyClientCert,
	}

	server := &http.Server{
		Addr:      ":8080",
		TLSConfig: tlsConfig,
	}

	http.HandleFunc("/hello", helloHandler)
	log.Println("Starting server on port 8080")
	log.Fatal(server.ListenAndServeTLS("certs/cert.pem", "certs/key.pem"))
}
```
Most of this is just boilerplate code to allow us to specify custom TLS config,
the important part is: `ClientAuth: tls.RequireAndVerifyClientCert,`, which 
requires all clients to present a valid TLS certificate.

Now for part 2. This works much the same way that it did for the client; we 
load the root certificate into a `CertPool`, then tell the server to trust all 
certs in the pool. Copy the `createCertPool` function from `client.go` into 
`server.go`, then at the start of `main()` in `server.go`, add to the TLS config:
```go
certPool := createCertPool("certs/root.pem")
tlsConfig := &tls.Config{
	ClientAuth: tls.RequireAndVerifyClientCert,
	ClientCAs:  certPool,
}
```
You'll also need to copy `root.pem` from `client/certs` to `server/certs`. Now 
the server will accept all client certificates issued by our Vault CA!

Finally part 3; telling the client to present a certificate. We first issue a 
certificate for the client:
```
vault write /pki/issue/client common_name=user@localhost
```
Copy the certificate to `client/certs/cert.pem`, and the corresponding private 
key to `client/certs/key.pem`. Your final directory structure should look like 
this:
```
├── client
│   ├── certs
│   │   ├── cert.pem
│   │   ├── key.pem
│   │   └── root.pem
│   └── client.go
└── server
    ├── certs
    │   ├── cert.pem
    │   ├── key.pem
    │   └── root.pem
    └── server.go
```

Now to use this certificate in the client. Add the following function to 
`client.go`:
```go
func readCertificate(certPath, keyPath string) tls.Certificate {
	cert, err := tls.LoadX509KeyPair(certPath, keyPath)
	if err != nil {
		log.Fatal("Error reading certificate and private key")
	}
	return cert
}
```
This will read both the certificate and private key into a `tls.Certificate`
struct. We then tell the client to use this `Certificate` for TLS by adding it to
the http client's config in `main()`:
```go
clientCertificate := readCertificate("./certs/cert.pem", "./certs/key.pem")
client := &http.Client{
	Transport: &http.Transport{
		TLSClientConfig: &tls.Config{
			RootCAs:      certPool,
			Certificates: []tls.Certificate{clientCertificate},
		},
	},
}
```

And that should be it! Overall the code should look something like:

client.go
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"time"
)

func main() {
	certPool := createCertPool("certs/root.pem")
	clientCertificate := readCertificate("./certs/cert.pem", "./certs/key.pem")
	client := &http.Client{
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{
				RootCAs:      certPool,
				Certificates: []tls.Certificate{clientCertificate},
			},
		},
	}
	for {
		body := fetchEndpoint("https://localhost:8080/hello", client)
		fmt.Println(string(body))
		time.Sleep(1 * time.Second)
	}
}

func createCertPool(certPath string) *x509.CertPool {
	rootCert, err := ioutil.ReadFile(certPath)
	if err != nil {
		log.Fatal(err)
	}
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(rootCert)

	return certPool
}

func readCertificate(certPath, keyPath string) tls.Certificate {
	cert, err := tls.LoadX509KeyPair(certPath, keyPath)
	if err != nil {
		log.Fatal("Error reading certificate and private key")
	}
	return cert
}

func fetchEndpoint(url string, client *http.Client) []byte {
	r, err := client.Get(url)
	if err != nil {
		log.Fatal(err)
	}

	defer r.Body.Close()
	body, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Fatal(err)
	}

	return body
}
```

server.go
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"io"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {
	certPool := createCertPool("certs/root.pem")
	tlsConfig := &tls.Config{
		ClientAuth: tls.RequireAndVerifyClientCert,
		ClientCAs:  certPool,
	}

	server := &http.Server{
		Addr:      ":8080",
		TLSConfig: tlsConfig,
	}

	http.HandleFunc("/hello", helloHandler)
	log.Println("Starting server on port 8080")
	log.Fatal(server.ListenAndServeTLS("certs/cert.pem", "certs/key.pem"))
}

func createCertPool(certPath string) *x509.CertPool {
	rootCert, err := ioutil.ReadFile(certPath)
	if err != nil {
		log.Fatal(err)
	}
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(rootCert)

	return certPool
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "Hello, world!\n")
}
```

Now for the moment of truth. Start the server with then in another terminal 
window start the client, and you should see the familiar `Hello, world!` message
being printed out.

### Conclusion
And that concludes this 3 part series on using Vault as a certificate authority 
for mutual TLS. A fully functional demo including dockerised apps and an 
intermediate CA can be found on Github [here](https://github.com/lewis-od/vault-mtls).



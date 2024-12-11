+++
title = 'Mutual TLS (mTLS) with macOS keychain'
description = "How to configure mTLS using Apple's macOS keychain"
image = "mtls-safari.png"
date = 2024-01-31
categories = ["Security"]
tags = ["mTLS", "TLS", "macOS", "Cyber Security"]
draft = false
+++

_This article is part of a series on [mTLS](../mtls). Check out the previous article: [mTLS Hello World](../mtls-hello-world)._

## Securing mTLS certificates and keys

In the [mTLS Hello World](../mtls-hello-world) article, we generated mTLS certificates and keys for the client and the server. We also created two certificate authorities (CAs) and signed the client and server certificates with their respective CAs. We ended up with the following files:

- server CA: `certs/server-ca.crt`
- server CA private key: `certs/server-ca.key`
- TLS certificate for localhost server: `certs/server.crt`
- server TLS certificate private key: `certs/server.key`
- client CA: `certs/client-ca.crt`
- client CA private key: `certs/client-ca.key`
- TLS certificate for client: `certs/client.crt`
- client TLS certificate private key: `certs/client.key`

In a real-world scenario, we would need to secure these files. The server CA private key and the client CA private key are the most important files to secure. If an attacker gets access to these files, they can create new certificates and impersonate the server or the client. These two files should be secured in a dedicated secure storage.

The server will need access to the client CA, the server TLS certificate, and the server TLS certificate private key. The server TLS certificate private key is the most important to secure out of these three files.

The client will need access to the server CA, the client TLS certificate, and the client TLS certificate private key. We can use the macOS keychain to secure these files. In a future article, we will show how to secure these on Windows with certificate stores.

## Apple's macOS keychain

As I've written in [inspecting keychain files on macOS](../inspecting-keychain-files-on-macos), keychains are the macOS's method to track and protect secure information such as passwords, private keys, and certificates.

The system keychain is located at `/Library/Keychains/System.keychain`. It contains the root certificates and other certificates. The login keychain is located at `/Users/<username>/Library/Keychains/login.keychain-db`. It contains the user's certificates and private keys. In this example, we will use the system keychain, which all users on the system can access.

## Generating mTLS certificates and keys

We will use the following script to generate the mTLS certificates and keys. It resembles the script from the [mTLS Hello World](../mtls-hello-world) article.

```shell
#!/bin/bash

# This script generates certificates and keys needed for mTLS.

mkdir -p certs

# Private keys for CAs
openssl genrsa -out certs/server-ca.key 2048
openssl genrsa -out certs/client-ca.key 2048

# Generate CA certificates
openssl req -new -x509 -nodes -days 1000 -key certs/server-ca.key -out certs/server-ca.crt -subj "/C=US/ST=Texas/L=Austin/O=Your Organization/OU=Your Unit/CN=testServerCA"
openssl req -new -x509 -nodes -days 1000 -key certs/client-ca.key -out certs/client-ca.crt -subj "/C=US/ST=Texas/L=Austin/O=Your Organization/OU=Your Unit/CN=testClientCA"

# Generate a certificate signing request
openssl req -newkey rsa:2048 -nodes -keyout certs/server.key -out certs/server.req -subj "/C=US/ST=Texas/L=Austin/O=Your Organization/OU=Your Unit/CN=testServerTLS"
openssl req -newkey rsa:2048 -nodes -keyout certs/client.key -out certs/client.req -subj "/C=US/ST=Texas/L=Austin/O=Your Organization/OU=Your Unit/CN=testClientTLS"

# Have the CA sign the certificate requests and output the certificates.
openssl x509 -req -in certs/server.req -days 398 -CA certs/server-ca.crt -CAkey certs/server-ca.key -set_serial 01 -out certs/server.crt -extfile localhost.ext
openssl x509 -req -in certs/client.req -days 398 -CA certs/client-ca.crt -CAkey certs/client-ca.key -set_serial 01 -out certs/client.crt

# Clean up
rm certs/server.req
rm certs/client.req
```

The maximum validity period for a TLS certificate is 398 days. Apple will reject certificates with a more extended validity period.

## Importing client mTLS certificates and keys into the macOS keychain

We will import the client mTLS certificates and keys into the macOS keychain using the following script. The script uses the [security](https://ss64.com/mac/security.html) command line tool. Accessing the system keychain must be run as root (`sudo`).

```shell
#!/bin/bash

# This script imports mTLS certificates and keys into the Apple Keychain.

# Import the server CA
security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain certs/server-ca.crt

# Import the client CA so that client TLS certificates can be verified
security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain certs/client-ca.crt
# Import the client TLS certificate and key
security import certs/client.crt -k /Library/Keychains/System.keychain
security import certs/client.key -k /Library/Keychains/System.keychain -x -T /usr/bin/curl -T /Applications/Safari.app -T '/Applications/Google Chrome.app'
```

The `-x` option marks the imported key as non-extractable. No application or user can view the private key once it is imported. The private key can only be used indirectly via Apple's APIs.

The `-T` option specifies the applications that can access the key. Additional applications may be added later to the access control list.

## Verifying imported certificates and keys

As an extra step, we can verify the client and server certificates before using them in an application.

We can verify the server certificate by running the following command:

```shell
security verify-cert -c certs/server.crt -p ssl -s localhost -k /Library/Keychains/System.keychain
```

The output should include:

```
...certificate verification successful.
```

The Apple keychain automatically combines the certificate and the private key into an identity. We can verify the client identity by running the following command:

```shell
security find-identity -p ssl-client /Library/Keychains/System.keychain
```

The list of identities should include:

```
Policy: SSL (client)
  Matching identities
  1) B307B90CCD374080E74F1B15AF602B35A75D8401 "testClientTLS"
     1 identities found

  Valid identities only
  1) B307B90CCD374080E74F1B15AF602B35A75D8401 "testClientTLS"
     1 valid identities found
```

macOS can validate the identity because we also imported the client CA into the system keychain.

## Running the mTLS server

As in the [mTLS Hello World](../mtls-hello-world) article, we will use `docker compose up` to start two nginx servers:

- https://localhost:8888 for TLS
- https://localhost:8889 for mTLS

## Connecting to the TLS and mTLS servers with clients

Because the server CA was added to the system keychain, curl can now access the TLS server without any additional flags:

```shell
curl https://localhost:8888/hello-world.txt
```

However, the built-in curl client cannot access the mTLS server. We use the `-v` option for additional information:

```shell
curl -v https://localhost:8889/hello-world.txt
```

The output:

```
*   Trying [::1]:8889...
* Connected to localhost (::1) port 8889
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Request CERT (13):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Certificate (11):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* ALPN: server accepted http/1.1
* Server certificate:
*  subject: C=US; ST=Texas; L=Austin; O=Your Organization; OU=Your Unit; CN=testServerTLS
*  start date: Jan 28 17:08:10 2024 GMT
*  expire date: Mar  1 17:08:10 2025 GMT
*  subjectAltName: host "localhost" matched cert's "localhost"
*  issuer: C=US; ST=Texas; L=Austin; O=Your Organization; OU=Your Unit; CN=testServerCA
*  SSL certificate verify ok.
* using HTTP/1.1
> GET /hello-world.txt HTTP/1.1
> Host: localhost:8889
> User-Agent: curl/8.4.0
> Accept: */*
>
< HTTP/1.1 400 Bad Request
< Server: nginx/1.25.3
< Date: Sun, 28 Jan 2024 18:28:20 GMT
< Content-Type: text/html
< Content-Length: 237
< Connection: close
<
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.25.3</center>
</body>
</html>
* Closing connection
```

The client attempted the TLS handshake, but the server rejected the connection because the client did not provide a certificate. Our built-in curl client does not currently support mTLS using the macOS keychain. The client used for this example is:

```
curl 8.4.0 (x86_64-apple-darwin23.0) libcurl/8.4.0 (SecureTransport) LibreSSL/3.3.6 zlib/1.2.12 nghttp2/1.55.1
Release-Date: 2023-10-11
Protocols: dict file ftp ftps gopher gophers http https imap imaps ldap ldaps mqtt pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
Features: alt-svc AsynchDNS GSS-API HSTS HTTP2 HTTPS-proxy IPv6 Kerberos Largefile libz MultiSSL NTLM NTLM_WB SPNEGO SSL threadsafe UnixSockets
```

On the other hand, Safari can access the mTLS server. We can verify this by opening the following URL in Safari:

```
https://localhost:8889/hello-world.txt
```

We see the following popup:

{{< figure src="mtls-safari.png" alt="Safari mTLS popup" caption="Safari mTLS popup" >}}

We can click **Continue** to connect to the mTLS server. Future connections will not show the popup and will automatically use the client certificate.

Google Chrome's behavior is similar.

**Note:** If we did not add Safari as an application that can access the client key, Safari would ask for a username and password to connect to the system keychain.

## Creating our own mTLS client

In the following article, we will [create our own mTLS client with the Go programming language](../mtls-go-client). This is the first step toward [creating an mTLS client integrated with the macOS keychain](../mtls-go-client-using-apple-keychain).

Later, we will [use mTLS with the Windows certificate store](../mtls-with-windows) and [create an mTLS client integrated with the Windows certificate store](../mtls-go-client-windows-certificate-store).

## Further reading

- Recently, we explained [agents and daemons and plists on macOS](../macos-launchd-agents-and-daemons/).
- We also showed [how to convert a script into a macOS install package](../script-only-macos-install-package/).

## Example code on GitHub

The example code is available on GitHub at https://github.com/getvictor/mtls/tree/master/mtls-with-apple-keychain

## mTLS with macOS keychain video

{{< youtube Y0y6-cCzz8w >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.

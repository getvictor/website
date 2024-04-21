+++
title = 'Mutual TLS (mTLS) Go client with custom certificate signer'
description = "How to build an mTLS Go client that sends a custom CertificateVerify message"
image = "signer.png"
date = 2024-02-14
tags = ["TLS", "Golang", "CyberSecurity"]
categories = ["mTLS"]
draft = false
+++

_This article is part of a series on [mTLS](/categories/mtls). Check out the previous articles:_
- [mTLS Hello World](../mtls-hello-world)
- [mTLS with macOS keychain](../mtls-with-apple-keychain)
- [mTLS Go client](../mtls-go-client)

## Why a custom certificate signer?

In the [mTLS Go client](../mtls-go-client) article, we built a simple Go client that uses mTLS. Our client used Go standard library methods and loaded the client certificate and private key from the filesystem. However, keeping the private key on the filesystem is insecure and not recommended. We aim to build an mTLS client fully integrated with the operating system's keystore.

The first step toward that goal is to extract the functionality of the mTLS handshake that requires the private key. Luckily, the client's private key is only needed to sign the `CertificateVerify` message. The `CertificateVerify` message is the last in the mTLS handshake. It proves to the server that the client has the private key associated with the client certificate.

{{< figure src="../mtls-hello-world/mtls-handshake.png" alt="Mutual TLS (mTLS) handshake diagram" >}}

From [Wikipedia entry on TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake):

> The client sends a **CertificateVerify** message, which is a signature over the previous handshake messages using the client's certificate's private key. This signature can be verified by using the client's certificate's public key. This lets the server know that the client has access to the private key of the certificate and thus owns the certificate.

## Setting up the environment

We will use the same certificates and keys script from the [mTLS with macOS keychain](../mtls-with-apple-keychain) article. If you still need to generate the certificates and keys, please follow the instructions in that article.

In addition, we will import the generated certificates and keys into the macOS keychain. (In a future article, we will use the Windows Certificate Store instead.)

Finally, as in the [mTLS Hello World](../mtls-hello-world) article, we will use `docker compose up` to start two nginx servers:
- https://localhost:8888 for TLS
- https://localhost:8889 for mTLS

## Building our crypto.Signer

We will build a custom `crypto.Signer` that signs the `CertificateVerify` message. The `crypto.Signer` interface is defined in the Go standard library's `crypto` package. It is used to sign messages with a private key.

```go
// CustomSigner is a crypto.Signer that uses the client certificate and key to sign
type CustomSigner struct {
    x509Cert       *x509.Certificate
    clientCertPath string
    clientKeyPath  string
}

func (k *CustomSigner) Public() crypto.PublicKey {
    fmt.Printf("crypto.Signer.Public\n")
    return k.x509Cert.PublicKey
}
func (k *CustomSigner) Sign(rand io.Reader, digest []byte, opts crypto.SignerOpts) (
    signature []byte, err error) {
    fmt.Printf("crypto.Signer.Sign\n")
    tlsCert, err := tls.LoadX509KeyPair(k.clientCertPath, k.clientKeyPath)
    if err != nil {
       log.Fatalf("error loading client certificate: %v", err)
    }
    fmt.Printf("Sign using %T\n", tlsCert.PrivateKey)
    return tlsCert.PrivateKey.(crypto.Signer).Sign(rand, digest, opts)
}
```

Although we still use the filesystem to load the client certificate and private key, we now use the `crypto.Signer` interface to sign the `CertificateVerify` message. In the future, we will replace this code by calls to the operating system's keystore. The vital thing to note is that we only load the private key when we need to sign the digest and do not load the key during the client configuration.

## Getting the client certificate

Besides building a custom `crypto.Signer`, we will implement a custom `GetClientCertificate` function. This function will be called during the TLS handshake when the server requests a certificate from the client. The function will load the client certificate and create a `CustomSigner` instance. It will not load the private key at this time. Once again, the client certificate is only loaded when needed and not during the client's configuration.

We set `Certificate: [][]byte{cert.Raw},` because the Go implementation of the TLS handshake requires the client certificate here to validate it against the server's CA.

```go
func GetClientCertificate(clientCertPath string, clientKeyPath string) (*tls.Certificate, error) {
    fmt.Printf("Server requested certificate\n")
    if clientCertPath == "" || clientKeyPath == "" {
       return nil, errors.New("client certificate and key are required")
    }
    clientBytes, err := os.ReadFile(clientCertPath)
    if err != nil {
       return nil, fmt.Errorf("error reading client certificate: %w", err)
    }
    var cert *x509.Certificate
    for block, rest := pem.Decode(clientBytes); block != nil; block, rest = pem.Decode(rest) {
       if block.Type == "CERTIFICATE" {
          cert, err = x509.ParseCertificate(block.Bytes)
          if err != nil {
             return nil, fmt.Errorf("error parsing client certificate: %v", err)
          }
       }
    }

    certificate := tls.Certificate{
       Certificate: [][]byte{cert.Raw},
       PrivateKey: &CustomSigner{
          x509Cert:       cert,
          clientCertPath: clientCertPath,
          clientKeyPath:  clientKeyPath,
       },
    }
    return &certificate, nil
}
```

## Putting it all together

With the above customizations, we create our new Go mTLS client:

```go
package main

import (
    "crypto/tls"
    "flag"
    "fmt"
    "github.com/getvictor/mtls/signer"
    "io"
    "log"
    "net/http"
)

func main() {

    urlPath := flag.String("url", "", "URL to make request to")
    clientCert := flag.String("cert", "", "Client certificate file")
    clientKey := flag.String("key", "", "Client key file")
    flag.Parse()
    if *urlPath == "" {
       log.Fatalf("URL to make request to is required")
    }

    client := http.Client{
       Transport: &http.Transport{
          TLSClientConfig: &tls.Config{
             GetClientCertificate: func(info *tls.CertificateRequestInfo) (
                 *tls.Certificate, error) {
                return signer.GetClientCertificate(*clientCert, *clientKey)
             },
          },
       },
    }

    // Make a GET request to the URL
    rsp, err := client.Get(*urlPath)
    if err != nil {
       log.Fatalf("error making get request: %v", err)
    }
    defer func() { _ = rsp.Body.Close() }()

    // Read the response body
    rspBytes, err := io.ReadAll(rsp.Body)
    if err != nil {
       log.Fatalf("error reading response: %v", err)
    }

    // Print the response body
    fmt.Printf("%s\n", string(rspBytes))
}
```

Trying to hit the mTLS server with:

```shell
go run client-signer.go --url https://localhost:8889/hello-world.txt --cert certs/client.crt --key certs/client.key
```

Returns the expected result:

```
mTLS Hello World!
```

## Using certificate and key from the macOS keychain

In the following article, we will [use the macOS keychain to load the client certificate and generate the `CertificateVerify` message without extracting the private key](../mtls-go-client-using-apple-keychain).

## Example code on GitHub

The example code is available on GitHub at https://github.com/getvictor/mtls/tree/master/mtls-go-custom-signer

## mTLS Go client with custom certificate signer video

{{< youtube FsKMAEfn21w >}}

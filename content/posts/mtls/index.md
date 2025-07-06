+++
title = 'Mutual TLS (mTLS): building a client using the system keystore'
description = "An overview of our series on mTLS"
image = "mtls-handshake.png"
date = 2024-04-01
categories = ["Security", "Software Development"]
tags = ["mTLS", "TLS", "Cyber Security"]
draft = false
+++

We recently completed a series of articles on mutual TLS (mTLS). In this series, we covered the basics of mTLS, how to
use macOS keychain and Windows certificate store, and how to build an mTLS Go client. Our goal was to show you how to
use mTLS in your applications and securely store your mTLS certificates and keys without exposing them to the
filesystem.

Here is a summary of the articles in the series:

### [Mutual TLS intro and hands-on example](../mtls-hello-world)

An introduction to mTLS and a hands-on example of using an mTLS client to connect to an mTLS server.

### [Using mTLS with the macOS keychain](../mtls-with-apple-keychain)

A guide on how to use the macOS system keystore to store your mTLS certificates and keys. We connect to an mTLS server
with applications that use the macOS system keychain to find the mTLS certificates.

### [Create an mTLS Go client](../mtls-go-client)

We create a standard mTLS client in Go using the `crypto/tls` library. This client loads the client certificate and
private key from the filesystem.

### [Add a custom certificate signer to the mTLS Go client](../mtls-go-custom-signer)

We implement a custom `crypto.Signer` to sign a client certificate during the mTLS handshake. Thus, we are a step closer
to removing our client certificate and private key from the filesystem.

### [A complete mTLS Go client using the macOS keychain](../mtls-go-client-using-apple-keychain)

In this article, we continue the previous article by connecting our custom signer to the macOS keychain using CGO and
Apple APIs.

### [Using mTLS with the Windows certificate store](../mtls-with-windows)

Switching to Windows, we learn how to use the Windows system keystore to store your mTLS certificates and keys. We
connect to an mTLS server with applications that use the Windows certificate store to find the mTLS certificates.

### [Create an mTLS Go client using the Windows certificate store](../mtls-go-client-windows-certificate-store)

Using the software pattern from the previous articles on the macOS keychain, we build an mTLS client in Go integrated
with the Windows certificate store to store the mTLS certificates and keys.

### [mTLS vs HTTP signature faceoff: securing your APIs](../mtls-vs-http-signature/)

Where do mTLS and HTTP message signatures fit, and how to choose the right one for your architecture.

### Mutual TLS (mTLS): building a client using the system keystore video playlist

{{< youtubeplaylist PLr-TrdMhEklRF4lQ_bIH0WJTUiY8ldc0W >}}

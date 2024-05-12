+++
title = 'Mutual TLS (mTLS) Go client using macOS keychain'
description = "How to build an mTLS Go client that uses Apple's macOS keychain"
image = "mtls-go-apple-keychain.png"
date = 2024-02-22
categories = ["Security", "Software Development"]
tags = ["mTLS", "TLS", "Golang", "macOS", "Application Security"]
draft = false
+++

_This article is part of a series on [mTLS](../mtls). Check out the previous articles:_
- [mTLS Hello World](../mtls-hello-world)
- [mTLS with macOS keychain](../mtls-with-apple-keychain)
- [mTLS Go client](../mtls-go-client)
- [mTLS Go client with custom certificate signer](../mtls-go-custom-signer)

## Why use macOS keychain?

In the [mTLS Go client](../mtls-go-client) article, we built a simple Go client that uses mTLS. Our client used Go standard library methods and loaded the client certificate and private key from the filesystem. However, keeping the private key on the filesystem is insecure and not recommended. We aim to build an mTLS client fully integrated with the operating system's keystore.

The macOS keychain is a secure storage system for passwords and other confidential information. It is used by many Apple applications, such as Safari, Mail, and iCloud, to store the user's passwords and additional sensitive information.

## Building a custom tls.Certificate for macOS keychain

This work builds on the [mTLS Go client with custom certificate signer](../mtls-go-custom-signer) article. We will use the `CustomSigner` from that article to build a custom `tls.Certificate` that uses the macOS keychain.

However, before the application uses the `Public` and `Sign` methods of the `CustomSigner,` we need to retrieve the certificate from the keychain using Apple's API.

### Retrieving certificate from macOS keychain with CGO

We will use CGO to call the macOS keychain API to retrieve the client certificate. To set up CGO, we include the following code above our imports:

```go
/*
   #cgo LDFLAGS: -framework CoreFoundation -framework Security
   #include <CoreFoundation/CoreFoundation.h>
   #include <Security/Security.h>
*/
import "C"
```

To find the identities from the keychain, we use [SecItemCopyMatching](https://developer.apple.com/documentation/security/1398306-secitemcopymatching). An identity is a certificate and its associated private key.

```go
identitySearch := C.CFDictionaryCreateMutable(
    C.kCFAllocatorDefault, maxCertificatesNum, &C.kCFTypeDictionaryKeyCallBacks,
     &C.kCFTypeDictionaryValueCallBacks,
)
defer C.CFRelease(C.CFTypeRef(unsafe.Pointer(identitySearch)))
const commonName = "testClientTLS"
var commonNameCFString = stringToCFString(commonName)
defer C.CFRelease(C.CFTypeRef(commonNameCFString))
C.CFDictionaryAddValue(identitySearch, unsafe.Pointer(C.kSecClass),
    unsafe.Pointer(C.kSecClassIdentity))
C.CFDictionaryAddValue(identitySearch, unsafe.Pointer(C.kSecAttrCanSign),
    unsafe.Pointer(C.kCFBooleanTrue))
C.CFDictionaryAddValue(identitySearch, unsafe.Pointer(C.kSecMatchSubjectWholeString),
    unsafe.Pointer(commonNameCFString))
// To filter by issuers, we must provide a CFDataRef array of DER-encoded ASN.1 items.
// C.CFDictionaryAddValue(identitySearch, unsafe.Pointer(C.kSecMatchIssuers), unsafe.Pointer(issuerCFArray))
C.CFDictionaryAddValue(identitySearch, unsafe.Pointer(C.kSecReturnRef),
    unsafe.Pointer(C.kCFBooleanTrue))
C.CFDictionaryAddValue(identitySearch, unsafe.Pointer(C.kSecMatchLimit),
    unsafe.Pointer(C.kSecMatchLimitAll))
var identityMatches C.CFTypeRef
if status := C.SecItemCopyMatching(C.CFDictionaryRef(identitySearch), &identityMatches);
    status != C.errSecSuccess {
    return nil, fmt.Errorf("failed to find client certificate: %v", status)
}
defer C.CFRelease(identityMatches)
```

In our example, we find the identities by a common name, which we hardcode for demonstration purposes. We can filter by the certificate issuer, as shown in the commented-out code. Filtering by issuer requires an array of DER-encoded ASN.1 items, which can be created from the `tls.CertificateRequestInfo` object. Another approach to finding the proper certificate is to retrieve all the keychain certificates and filter them in Go code.

### Converting the Apple identity to a Go `x509.Certificate`

After we retrieve the array of identities from the keychain, we convert them to Go `x509.Certificate` objects and pick the first one that is not expired.

```go
var foundCert *x509.Certificate
var foundIdentity C.SecIdentityRef
identityMatchesArrayRef := C.CFArrayRef(identityMatches)
numIdentities := int(C.CFArrayGetCount(identityMatchesArrayRef))
fmt.Printf("Found %d identities\n", numIdentities)
for i := 0; i < numIdentities; i++ {
    identityMatch := C.CFArrayGetValueAtIndex(identityMatchesArrayRef, C.CFIndex(i))
    x509Cert, err := identityRefToCert(C.SecIdentityRef(identityMatch))
    if err != nil {
        continue
    }
    // Make sure certificate is not expired
    if x509Cert.NotAfter.After(time.Now()) {
        foundCert = x509Cert
        foundIdentity = C.SecIdentityRef(identityMatch)
        fmt.Printf("Found certificate from issuer %s with public key type %T\n",
            x509Cert.Issuer.String(), x509Cert.PublicKey)
        break
    }
}
```

The `identityRefToCert` function converts the `SecIdentityRef` to a Go `x509.Certificate` object. It exports the certificate to PEM format using [SecItemExport](https://developer.apple.com/documentation/security/1394828-secitemexport) and then parses the PEM to get the `x509.Certificate` object.

```go
func identityRefToCert(identityRef C.SecIdentityRef) (*x509.Certificate, error) {
    // Convert the identity to a certificate
    var certificateRef C.SecCertificateRef
    if status := C.SecIdentityCopyCertificate(identityRef, &certificateRef);
        status != 0 {
       return nil, fmt.Errorf("failed to get certificate from identity: %v", status)
    }
    defer C.CFRelease(C.CFTypeRef(certificateRef))

    // Export the certificate to PEM
    // SecItemExport: https://developer.apple.com/documentation/security/1394828-secitemexport
    var pemDataRef C.CFDataRef
    if status := C.SecItemExport(
       C.CFTypeRef(certificateRef), C.kSecFormatPEMSequence, C.kSecItemPemArmour, nil,
           &pemDataRef,
    ); status != 0 {
       return nil, fmt.Errorf("failed to export certificate to PEM: %v", status)
    }
    defer C.CFRelease(C.CFTypeRef(pemDataRef))
    certPEM := C.GoBytes(unsafe.Pointer(C.CFDataGetBytePtr(pemDataRef)),
        C.int(C.CFDataGetLength(pemDataRef)))

    var x509Cert *x509.Certificate
    for block, rest := pem.Decode(certPEM); block != nil; block, rest = pem.Decode(rest) {
       if block.Type == "CERTIFICATE" {
          var err error
          x509Cert, err = x509.ParseCertificate(block.Bytes)
          if err != nil {
             return nil, fmt.Errorf("error parsing client certificate: %v", err)
          }
       }
    }
    return x509Cert, nil
}
```

### Retrieve the private key reference from the keychain

At this point, we also retrieve the private key reference from the keychain. We will use the private key reference to sign the `CertificateVerify` message during the TLS handshake. The reference does not contain the private key. When importing private keys to the keychain, they should be marked as non-exportable so that no one can retrieve the private key cleartext from the keychain.

```go
var privateKey C.SecKeyRef
if status := C.SecIdentityCopyPrivateKey(C.SecIdentityRef(foundIdentity), &privateKey);
    status != 0 {
    return nil, fmt.Errorf("failed to copy private key ref from identity: %v", status)
}
```

### Building the custom `tls.Certificate`

Finally, we put together the custom `tls.Certificate` using the `x509.Certificate` and the private key reference.

```go
customSigner := &CustomSigner{
    x509Cert:   foundCert,
    privateKey: privateKey,
}
certificate := tls.Certificate{
    Certificate:                  [][]byte{foundCert.Raw},
    PrivateKey:                   customSigner,
    SupportedSignatureAlgorithms: []tls.SignatureScheme{supportedAlgorithm},
}
```

Our example only supports the `tls.PSSWithSHA256` signature algorithm to keep the code simple. Adding additional algorithm support is easy since it only requires passing the right parameter to the `SecKeyCreateSignature` function, which we will review next.

## Signing the mTLS digest with Apple's keychain

As discussed in the previous [mTLS Go client with custom certificate signer](../mtls-go-custom-signer) article, we need to sign the `CertificateVerify` message during the TLS handshake. We will use the `CustomSigner` to sign the digest, which implements the `crypto.Signer` interface as defined in the Go standard library's `crypto` package.

```go
type CustomSigner struct {
    x509Cert   *x509.Certificate
    privateKey C.SecKeyRef
}

func (k *CustomSigner) Public() crypto.PublicKey {
    fmt.Printf("crypto.Signer.Public\n")
    return k.x509Cert.PublicKey
}

func (k *CustomSigner) Sign(_ io.Reader, digest []byte, opts crypto.SignerOpts) (
    signature []byte, err error) {
    fmt.Printf("crypto.Signer.Sign with key type %T, opts type %T, hash %s\n",
        k.Public(), opts, opts.HashFunc().String())

    // Convert the digest to a CFDataRef
    digestCFData := C.CFDataCreate(C.kCFAllocatorDefault,
        (*C.UInt8)(unsafe.Pointer(&digest[0])), C.CFIndex(len(digest)))
    defer C.CFRelease(C.CFTypeRef(digestCFData))

    // SecKeyAlgorithm: https://developer.apple.com/documentation/security/seckeyalgorithm
    // SecKeyCreateSignature: https://developer.apple.com/documentation/security/1643916-seckeycreatesignature
    var cfErrorRef C.CFErrorRef
    signCFData := C.SecKeyCreateSignature(
       k.privateKey, C.kSecKeyAlgorithmRSASignatureDigestPSSSHA256,
       C.CFDataRef(digestCFData), &cfErrorRef,
    )
    if cfErrorRef != 0 {
       return nil, fmt.Errorf("failed to sign data: %v", cfErrorRef)
    }
    defer C.CFRelease(C.CFTypeRef(signCFData))

    // Convert CFDataRef to Go byte slice
    return C.GoBytes(unsafe.Pointer(C.CFDataGetBytePtr(signCFData)),
        C.int(C.CFDataGetLength(signCFData))), nil
}
```

We use the [SecKeyCreateSignature](https://developer.apple.com/documentation/security/1643916-seckeycreatesignature) function to sign the digest. The function takes the private key reference, the algorithm, the digest, and a pointer to a `CFErrorRef.` The function returns a CFDataRef, which we convert to a Go byte slice. Additional algorithms can be supported by passing the proper parameter to the `SecKeyCreateSignature` function.

## Putting it all together

With the above code, we can create our new Go mTLS client that uses the macOS keychain.

```go
func main() {

    urlPath := flag.String("url", "", "URL to make request to")
    flag.Parse()
    if *urlPath == "" {
       log.Fatalf("URL to make request to is required")
    }

    client := http.Client{
       Transport: &http.Transport{
          TLSClientConfig: &tls.Config{
             GetClientCertificate: signer.GetClientCertificate,
             MinVersion:           tls.VersionTLS13,
             MaxVersion:           tls.VersionTLS13,
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

We limit the scope of this example to TLS 1.3

## Build the mTLS client

With `go build client-signer.go`, we generate the `client-signer` executable.

## Setting up the environment

The next step is to use the macOS keychain to store the client certificate and private key. We will use the same certificates and keys script from the [mTLS with macOS keychain](../mtls-with-apple-keychain) article. If you still need to generate the certificates and keys, please follow the instructions in that article.

We must also import the generated certificates and keys into the macOS keychain.

```shell
# Import the server CA
security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain certs/server-ca.crt

# Import the client CA so that client TLS certificates can be verified
security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain certs/client-ca.crt
# Import the client TLS certificate and key
security import certs/client.crt -k /Library/Keychains/System.keychain
security import certs/client.key -k /Library/Keychains/System.keychain -x -T $PWD/client-signer -T /usr/bin/curl -T /Applications/Safari.app -T '/Applications/Google Chrome.app'
```

We specify our application `$PWD/client-signer` as one of the trusted applications that can access the private key. If we do not select the trusted application, we will get a security pop-up whenever our app tries to access the private key.

Finally, as in the [mTLS Hello World](../mtls-hello-world) article, we will use `docker compose up` to start two nginx servers:
- https://localhost:8888 for TLS
- https://localhost:8889 for mTLS

## Running the Go mTLS client using the macOS keychain

We can now run our mTLS client without pointing to certificate and key files. Hitting the ordinary TLS server:

```shell
./client-signer --url https://localhost:8888/hello-world.txt
```

Returns the expected:

```
TLS Hello World!
```

While hitting the mTLS server:

```shell
./client-signer --url https://localhost:8889/hello-world.txt
```

Returns a more detailed message, including the print statements in our custom code:

```
Server requested certificate
Found 1 identities
Found certificate from issuer CN=testClientCA,OU=Your Unit,O=Your Organization,L=Austin,ST=Texas,C=US with public key type *rsa.PublicKey
crypto.Signer.Public
crypto.Signer.Public
crypto.Signer.Sign with key type *rsa.PublicKey, opts type *rsa.PSSOptions, hash SHA-256
mTLS Hello World!
```

## Using certificate and key from the Windows certificate store

The following article will explore [using the Windows certificate store to hold the mTLS client certificate and private key](../mtls-with-windows).

## Example code on GitHub

The example code is available on GitHub at https://github.com/getvictor/mtls/tree/master/mtls-go-apple-keychain

## mTLS Go client using macOS keychain video

{{< youtube iYWPrL4sR5U >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.

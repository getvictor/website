+++
title = 'Mutual TLS (mTLS) Go client using Windows certificate store'
description = "How to build an mTLS Go client that uses the Windows certificate store"
image = "mtls-go-windows.png"
date = 2024-03-20
categories = ["Security", "Software Development"]
tags = ["mTLS", "TLS", "Golang", "Windows", "Application Security"]
draft = false
+++

_This article is part of a series on [mTLS](../mtls). Check out the previous articles:_
- [mTLS Hello World](../mtls-hello-world)
- [mTLS with macOS keychain](../mtls-with-apple-keychain)
- [mTLS Go client](../mtls-go-client)
- [mTLS Go client with custom certificate signer](../mtls-go-custom-signer)
- [mTLS Go client using macOS keychain](../mtls-go-client-using-apple-keychain)
- [mTLS with Windows certificate store](../mtls-with-windows)

## Why use the Windows certificate store?

Keeping the mTLS client private key on the filesystem is insecure and not recommended. In the [mTLS Go client using macOS keychain](../mtls-go-client-using-apple-keychain), we demonstrated achieving greater mTLS security with macOS keychain. In this article, we reach a similar level of protection with the Windows certificate store.

The Windows certificate store is a secure location where certificates and keys can be stored. Many applications, such as Edge and Powershell, use it. The Windows certificate store is an excellent place to store mTLS client certificates and keys.

## Building a custom tls.Certificate for the Windows certificate store

This work builds on the [mTLS Go client with custom certificate signer](../mtls-go-custom-signer) article. We will use the `CustomSigner` from that article to build a custom `tls.Certificate` that uses the Windows certificate store.

However, before the application uses the `Public` and `Sign` methods of the `CustomSigner,` we must retrieve the client certificate using Windows APIs.

### Retrieving mTLS client certificate from Windows certificate store using Go

We will use the [golang.org/x/sys/windows](https://pkg.go.dev/golang.org/x/sys/windows) package to access the Windows APIs. We use the `windows` package to call the [CertOpenStore](https://learn.microsoft.com/en-us/windows/win32/api/wincrypt/nf-wincrypt-certopenstore), [CertFindCertificateInStore](https://learn.microsoft.com/en-us/windows/win32/api/wincrypt/nf-wincrypt-certfindcertificateinstore), and [CryptAcquireCertificatePrivateKey](https://learn.microsoft.com/en-us/windows/win32/api/wincrypt/nf-wincrypt-cryptacquirecertificateprivatekey) functions from the `crypt32` DLL (dynamic link library).

First, we open the `MY` store, which is the personal store for the current user. This store contains our client mTLS certificate.

```go
// Open the certificate store
storePtr, err := windows.UTF16PtrFromString(windowsStoreName)
if err != nil {
    return nil, err
}
store, err := windows.CertOpenStore(
    windows.CERT_STORE_PROV_SYSTEM,
    0,
    uintptr(0),
    windows.CERT_SYSTEM_STORE_CURRENT_USER,
    uintptr(unsafe.Pointer(storePtr)),
)
if err != nil {
    return nil, err
}
```

Next, we find the certificate by the common name.

```go
// Find the certificate
var pPrevCertContext *windows.CertContext
var certContext *windows.CertContext
commonNamePtr, err := windows.UTF16PtrFromString(commonName)
for {
    certContext, err = windows.CertFindCertificateInStore(
        store,
        windows.X509_ASN_ENCODING,
        0,
        windows.CERT_FIND_SUBJECT_STR,
        unsafe.Pointer(commonNamePtr),
        pPrevCertContext,
    )
    if err != nil {
        return nil, err
    }
    // We can extract the certificate chain and further filter the certificate
    // we want here.
    break
}
```

### Converting the Windows certificate to a Go `x509.Certificate`

After retrieving the certificate from the Windows certificate store, we convert it to a Go `x509.Certificate`.

```go
// Copy the certificate data so that we have our own copy outside the windows context
encodedCert := unsafe.Slice(certContext.EncodedCert, certContext.Length)
buf := bytes.Clone(encodedCert)
foundCert, err := x509.ParseCertificate(buf)
if err != nil {
    return nil, err
}
```

### Building the custom `tls.Certificate`

Finally, we put together the custom `tls.Certificate` using the `x509.Certificate`. We hold on to the `certContext` pointer to get the private key later.

```go
customSigner := &CustomSigner{
    store:              store,
    windowsCertContext: certContext,
}

customSigner.x509Cert = foundCert

certificate := tls.Certificate{
    Certificate:                  [][]byte{foundCert.Raw},
    PrivateKey:                   customSigner,
    SupportedSignatureAlgorithms: []tls.SignatureScheme{supportedAlgorithm},
}
```

Our example only supports the `tls.PSSWithSHA256` signature algorithm to keep the code simple.

## Signing the mTLS digest with the Windows certificate store

As discussed in the previous [mTLS Go client with custom certificate signer](../mtls-go-custom-signer) article, we must sign the `CertificateVerify` message during the TLS handshake. We will use the `CustomSigner` to sign the digest, which implements the `crypto.Signer` interface as defined in the Go standard library's `crypto` package.

```go
// CustomSigner is a crypto.Signer that uses the client certificate and key to sign
type CustomSigner struct {
    store              windows.Handle
    windowsCertContext *windows.CertContext
    x509Cert           *x509.Certificate
}

func (k *CustomSigner) Public() crypto.PublicKey {
    fmt.Printf("crypto.Signer.Public\n")
    return k.x509Cert.PublicKey
}

func (k *CustomSigner) Sign(_ io.Reader, digest []byte, opts crypto.SignerOpts
    ) (signature []byte, err error) {
    ...
```

#### Retrieve the private key reference from the Windows certificate store

We retrieve the private key reference from the Windows certificate store using the `CryptAcquireCertificatePrivateKey` function.

```go
// Get private key
var (
    privateKey                  windows.Handle
    pdwKeySpec                  uint32
    pfCallerFreeProvOrNCryptKey bool
)
err = windows.CryptAcquireCertificatePrivateKey(
    k.windowsCertContext,
    windows.CRYPT_ACQUIRE_CACHE_FLAG|windows.CRYPT_ACQUIRE_SILENT_FLAG|
        windows.CRYPT_ACQUIRE_ONLY_NCRYPT_KEY_FLAG,
    nil,
    &privateKey,
    &pdwKeySpec,
    &pfCallerFreeProvOrNCryptKey,
)
if err != nil {
    return nil, err
}
```

#### Signing the mTLS digest

We will use the [NCryptSignHash](https://learn.microsoft.com/en-us/windows/win32/api/ncrypt/nf-ncrypt-ncryptsignhash) function from `ncrypt.dll` to sign the digest.

```go
var (
    nCrypt         = windows.MustLoadDLL("ncrypt.dll")
    nCryptSignHash = nCrypt.MustFindProc("NCryptSignHash")
)
```

But before we do that, we must create a `BCRYPT_PSS_PADDING_INFO` structure for our supported RSA-PSS algorithm.

```go
flags := nCryptSilentFlag | bCryptPadPss
pPaddingInfo, err := getRsaPssPadding(opts)
if err != nil {
    return nil, err
}
```

Where `getRsaPssPadding` is a helper function:

```go
func getRsaPssPadding(opts crypto.SignerOpts) (unsafe.Pointer, error) {
    pssOpts, ok := opts.(*rsa.PSSOptions)
    if !ok || pssOpts.Hash != crypto.SHA256 {
       return nil, fmt.Errorf("unsupported hash function %s", opts.HashFunc().String())
    }
    if pssOpts.SaltLength != rsa.PSSSaltLengthEqualsHash {
       return nil, fmt.Errorf("unsupported salt length %d", pssOpts.SaltLength)
    }
    sha256, _ := windows.UTF16PtrFromString("SHA256")
    // Create BCRYPT_PSS_PADDING_INFO structure:
    // typedef struct _BCRYPT_PSS_PADDING_INFO {
    //     LPCWSTR pszAlgId;
    //     ULONG   cbSalt;
    // } BCRYPT_PSS_PADDING_INFO;
    return unsafe.Pointer(
       &struct {
          pszAlgId *uint16
          cbSalt   uint32
       }{
          pszAlgId: sha256,
          cbSalt:   uint32(pssOpts.HashFunc().Size()),
       },
    ), nil
}
```

Finally, we sign the digest using the `NCryptSignHash` function.

```go
    // Sign the digest
    // The first call to NCryptSignHash retrieves the size of the signature
    var size uint32
    success, _, _ := nCryptSignHash.Call(
       uintptr(privateKey),
       uintptr(pPaddingInfo),
       uintptr(unsafe.Pointer(&digest[0])),
       uintptr(len(digest)),
       uintptr(0),
       uintptr(0),
       uintptr(unsafe.Pointer(&size)),
       uintptr(flags),
    )
    if success != 0 {
       return nil, fmt.Errorf("NCryptSignHash: failed to get signature length: %#x", success)
    }

    // The second call to NCryptSignHash retrieves the signature
    signature = make([]byte, size)
    success, _, _ = nCryptSignHash.Call(
       uintptr(privateKey),
       uintptr(pPaddingInfo),
       uintptr(unsafe.Pointer(&digest[0])),
       uintptr(len(digest)),
       uintptr(unsafe.Pointer(&signature[0])),
       uintptr(size),
       uintptr(unsafe.Pointer(&size)),
       uintptr(flags),
    )
    if success != 0 {
       return nil, fmt.Errorf("NCryptSignHash: failed to generate signature: %#x", success)
    }
    return signature, nil
```

## Putting it all together

With the above code, we can create our new Go mTLS client that uses the Windows certificate store.

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

## Setting up the environment
The next step is to use the Windows certificate store to store the client certificate and private key. We will use the certificates and keys scripts from the previous [mTLS with Windows certificate store](../mtls-with-windows) article. If you still need to generate the certificates and keys, please follow the instructions in that article.

Finally, as in the [mTLS with Windows certificate store](../mtls-with-windows) article, we start two nginx servers:
- https://<your_host>:8888 for TLS
- https://<your_host>:8889 for mTLS

## Running the Go mTLS client using the Windows certificate store

We can run our mTLS client without pointing to certificate/key files and retrieving everything from the Windows certificate store. Hitting the ordinary TLS server:

```powershell
go run .\client-signer.go --url https://myhost:8888/hello-world.txt
```

Returns the expected:

```
TLS Hello World!
```

While hitting the mTLS server:

```powershell
go run .\client-signer.go --url https://myhost:8889/hello-world.txt
```

Returns a more detailed message, including the print statements in our custom code:

```
Server requested certificate
Found certificate with common name testClientTLS
crypto.Signer.Public
crypto.Signer.Public
crypto.Signer.Sign with key type *rsa.PublicKey, opts type *rsa.PSSOptions, hash SHA-256
mTLS Hello World!
```

## Example code on GitHub

The example code is available on GitHub at https://github.com/getvictor/mtls/tree/master/mtls-go-windows

## mTLS Go client using Windows certificate store video

{{< youtube L4uk43i3kyY >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.

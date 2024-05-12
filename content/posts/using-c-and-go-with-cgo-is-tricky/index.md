+++
title = 'Using C and Go with CGO is tricky'
description = "CGO Hello World fail!"
image = "cgo-hello-world-fail.png"
date = 2024-01-18
tags = ["Golang", "CGO", "Hello World"]
draft = false
+++

## Simple CGO examples

CGO is a way to call C code from Go. It helps call existing C libraries or for performance reasons. CGO is enabled by default but can be disabled with the `-cgo` build flag.

Below is a simple example of calling a C function from Go.

```go
package main

/*
double add(double a, double b) {
    return a + b;
}
*/
import "C"
import "fmt"

func main() {
    fmt.Println(C.add(1, 2))
}
```

The C code is embedded in the Go code as a comment above `import "C"`. The comment must start with `/*` and end with `*/`. The C code must be valid.
The Go compiler compiles the C code and links the resulting object file with the Go code.

Here is an example of using an existing C library.

```go
package main

/*
#include "math.h"
double add(double a, double b) {
    return a + b;
}
*/
import "C"
import "fmt"

func main() {
    fmt.Println(C.floor(C.add(1, 2.1)))
}
```

We call the `floor` function from the `math.h` library. The `math.h` library is included with the C compiler, so we don't need to do anything special to use it.

## CGO Hello World fail

Here is another example where we print "Hello World" from C.

```go
package main

/*
#include "stdio.h"
*/
import "C"

func main() {
    C.printf(C.CString("Hello World\n"))
}
```

However, the above seemingly straightforward example will fail to compile with the following enigmatic error:

```shell
cgo: ./exmaple.go:9:2: unexpected type: ...
```

The problem is that `printf` is a variadic function that can take a variable number of arguments. CGO does not support variadic functions. Even using Go variadic syntax will not work:

```go
    args := []interface{}{}
    C.printf(C.CString("Hello World\n"), args...)
```

The workaround for this is to use another non-variadic function, such as `vprintf`, or to wrap the variadic C function in a non-variadic C function.

```go
package main

/*
    #include "stdio.h"
    void wrapPrintf(const char *s) {
       printf("%s", s);
    }
*/
import "C"

func main() {
    C.wrapPrintf(C.CString("Hello, World\n"))
}
```

## C++ Hello World fail

Another issue with CGO is only C code can be called from Go. C++ code cannot be called from Go. The following code will fail to compile:

```go
package main

/*
    #include <iostream>
    void helloWorld() {
       std::cout << "Hello, World" << std::endl;
    }
*/
import "C"

func main() {
    C.helloWorld()
}
```

However, C++ code can be called from C, so we can write a C wrapper for the C++ code.

## CGO real-world example

The following is an example of real-world usage of CGO, which uses Apple's APIs to add a secret to the keychain.

```go
package keystore

/*
   #cgo LDFLAGS: -framework CoreFoundation -framework Security
   #include <CoreFoundation/CoreFoundation.h>
   #include <Security/Security.h>
*/
import "C"
import (
    "fmt"
    "unsafe"
)

const service = "com.fleetdm.fleetd.enroll.secret"

var serviceStringRef = stringToCFString(service)

// AddSecret will add a secret to the keychain. This application can retrieve this
// secret without any user authorization.
func AddSecret(secret string) error {

    query := C.CFDictionaryCreateMutable(
       C.kCFAllocatorDefault,
       0,
       &C.kCFTypeDictionaryKeyCallBacks,
       &C.kCFTypeDictionaryValueCallBacks,
    )
    defer C.CFRelease(C.CFTypeRef(query))

    data := C.CFDataCreate(C.kCFAllocatorDefault,
                           (*C.UInt8)(unsafe.Pointer(C.CString(secret))),
                           C.CFIndex(len(secret)))
    defer C.CFRelease(C.CFTypeRef(data))

    C.CFDictionaryAddValue(query, unsafe.Pointer(C.kSecClass),
                           unsafe.Pointer(C.kSecClassGenericPassword))
    C.CFDictionaryAddValue(query, unsafe.Pointer(C.kSecAttrService),
                           unsafe.Pointer(serviceStringRef))
    C.CFDictionaryAddValue(query, unsafe.Pointer(C.kSecValueData),
                           unsafe.Pointer(data))

    status := C.SecItemAdd(C.CFDictionaryRef(query), nil)
    if status != C.errSecSuccess {
       return fmt.Errorf("failed to add %v to keychain: %v", service, status)
    }
    return nil
}

// stringToCFString will return a CFStringRef
func stringToCFString(s string) C.CFStringRef {
    bytes := []byte(s)
    ptr := (*C.UInt8)(&bytes[0])
    return C.CFStringCreateWithBytes(C.kCFAllocatorDefault, ptr, C.CFIndex(len(bytes)),
                                     C.kCFStringEncodingUTF8, C.false)
}
```

The C linker flags are specified with the `#cgo LDFLAGS` directive.

The CGO code uses a lot of casting and data conversion. Let's break down the following segment:

```go
(*C.UInt8)(unsafe.Pointer(C.CString(secret)))
```

`C.CString` converts a Go string to a C string. It is one of the CGO special functions to convert between Go and C types. See [cgo documentation](https://pkg.go.dev/cmd/cgo) for more information.

`unsafe.Pointer` converts a C pointer to a generic Go pointer. And `(*C.UInt8)` casts the Go pointer back to a C pointer.

Unfortunately, CGO cannot cast a C string to a `(*C.UInt8)` directly. The following will fail to compile:

```go
(*C.UInt8)(C.CString(secret))
```

We must go through an intermediate cast to `unsafe.Pointer`, representing a void C pointer.

## Additional topics

Our custom C and Go code was always in the same file in the above examples. However, the C code can be in a separate file and linked to our Go executable.

## CGO Hello World fail video

{{< youtube C9h8YO1NwPM >}}

+++
title = "How to override methods in Go"
description = "Go supports method overriding through embedded structs"
authors = ["Victor Lyuboslavsky"]
image = "method-overriding-headline.png"
date = 2025-01-01
categories = ["Software Development"]
tags = ["Golang"]
draft = false
+++

- [Example of method overriding with embedded structs](#method-overriding-with-embedded-structs)

## What is method overriding?

[Method overriding](https://en.wikipedia.org/wiki/Method_overriding) is a feature of object-oriented programming
languages that allows a subclass to provide a specific implementation of a method already defined in its superclass.
When a subclass overrides a method, the subclass's method is called instead of the superclass's method when the object
is of the subclass type. Method overriding is an example of polymorphism, where the same method name can have different
implementations depending on the object's type.

Go does not have classes, but it has structs and interfaces that can be used to achieve similar functionality.

## Embedded structs

Go favors composition over inheritance. Multiple smaller types can be combined to create a larger type. A clean way to
do this is through [embedded structs](https://golang.org/doc/effective_go#embedding). When a struct embeds another
struct, it inherits the embedded struct's fields and methods. For example, consider the following code:

```go
package main

import "fmt"

type MyPrint struct {
    i int
}

func (m* MyPrint) Do() {
    fmt.Printf("Hello, World! %d\n", m.i)
    m.i++
}

type MyComposedPrint struct {
    MyPrint
}

func main() {
    fmt.Println("Base:")
    var m MyPrint
    m.Do()
    m.Do()

    fmt.Println("Composed:")
    var s MyComposedPrint
    s.Do()
    s.Do()
}
```

This code outputs the following when run:

```
Base:
Hello, World! 0
Hello, World! 1
Composed:
Hello, World! 0
Hello, World! 1
```

The composed struct forwards the method call to the embedded struct.

{{< figure src="embedded-struct.png" alt="MyPrint struct inside the MyComposedPrint struct.">}}

## Method overriding with embedded structs

To override a method in Go, you can define a method with the same name and parameters in the composed struct. For
example, consider the following code:

```go
package main

import "fmt"

type MyPrint struct {
    i int
}

func (m* MyPrint) Do() {
    fmt.Printf("Hello, World! %d\n", m.i)
    m.i++
}

type MyComposedPrint struct {
    MyPrint
}

func (m*MyComposedPrint) Do() {
    // call the parent method, similar to super.Do() in other languages
    m.MyPrint.Do()
    fmt.Printf("Hello, Composed World! %d\n", m.i)
    m.i++
}

func main() {
    fmt.Println("Base:")
    var m MyPrint
    m.Do()
    m.Do()

    fmt.Println("Composed:")
    var s MyComposedPrint
    s.Do()
    s.Do()
}
```

This code outputs the following when run:

```
Base:
Hello, World! 0
Hello, World! 1
Composed:
Hello, World! 0
Hello, Composed World! 1
Hello, World! 2
Hello, Composed World! 3
```

`MyComposedPrint.Do` overrides `MyPrint.Do`. When deciding which method to call, Go first looks for the method in the
type itself. If the method is not found, Go looks for the method in the embedded types.

## Method overriding and interfaces

Method overriding is often used in conjunction with interfaces. An interface defines a set of methods a type must
implement to satisfy the interface. When a type implements an interface, it can be used wherever the interface is
expected. An interface is another example of polymorphism in Go.

One typical pattern is overriding a method in a 3rd party library. For example, you can embed a type from a library and
override a method to add additional functionality. However, you will still pass the new type in your code using the
library's original interface.

## Further reading

- We recently described [the differences between Go modules and packages](../go-modules-and-packages/).
- Previously, we explained [how to properly unmarshal JSON null, set, and missing fields in Go](../go-json-unmarshal/).
- We also explained [the difference between nil slice and empty slice in Go](../nil-slice-versus-empty-slice-in-go/).

## Watch method overriding in Go video

{{< youtube OW42Lv8HjUk >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.

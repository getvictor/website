+++
title = 'Nil slice versus empty slice in Go'
description = "Don't fear the nil slice!"
image = "cover.png"
date = 2023-12-28
tags = ["Golang", "null", "GoStyleGuide"]
draft = false
+++

{{< youtube q0B4q_0u4XI >}}

When starting to code in Go, I encountered the following situation. I needed to create an empty slice, so I did:

```go
slice := []string{}
```

However, my IDE flagged it as a warning, and pointed me to [this Go style guide passage](https://go.dev/wiki/CodeReviewComments#declaring-empty-slices), which recommended using a nil slice instead:

```go
var slice []string
```

This recommendation didn't seem right to me. How can a nil variable be better? Won’t I run into issues like null pointer exceptions and other annoyances? Well, as it turns out, that’s not how slices work in Go. When declaring a nil slice, it is not the dreaded null pointer. It is still a slice. This slice includes a slice header, but its value just happens to be nil.

The main difference between a nil slice and an empty slice is the following. A nil slice compared to nil will return true. That’s pretty much it.

```go
if slice == nil {
	fmt.Println("Slice is nil.")
} else {
	fmt.Println("Slice is NOT nil.")
}
```

When printing a nil slice, it will print like an empty slice:

```go
fmt.Printf("Slice is: %v\n", slice)
```

```
Slice is: []
```

You can append to a nil slice:

```go
slice = append(slice, "bozo")
```

You can loop over a nil slice, and the code will not enter the for loop:

```go
for range slice {
	fmt.Println("We are in a for loop.")
}
```

The length of a nil slice is 0:

```go
fmt.Printf("len: %#v\n", len(slice))
```

```
len: 0
```

And, of course, you can pass a nil slice by pointer. That’s right -- pass a nil slice by pointer.

```go
func passByPointer(slice *[]string) {
    fmt.Printf("passByPointer len: %#v\n", len(*slice))
    *slice = append(*slice, "bozo")
}
```

You will get the updated slice if the underlying slice is reassigned.

```go
passByPointer(&slice)
fmt.Printf("len after passByPointer: %#v\n", len(slice))
```

```
len after passByPointer: 1
```

The code above demonstrates that a nil slice is not a nil pointer. On the other hand, you cannot dereference a nil pointer like you can a nil slice. This code causes a crash:

```go
var nullSlice *[]string
fmt.Printf("Crash: %#v\n", len(*nullSlice))
```

Here's the full gist:

{{< gist getvictor bff0fa45185630e264a40476207d8e4d >}}

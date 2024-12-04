+++
title = "Use Go to unmarshal JSON null, set, and missing fields"
description = "How to detect null, set, or missing JSON fields when unmarshalling into a Go struct"
authors = ["Victor Lyuboslavsky"]
image = "go-json-unmarshal-headline.png"
date = 2024-10-09
categories = ["Software Development"]
tags = ["Golang", "JSON"]
draft = false
+++

## JSON unmarshalling use cases

When passing a JSON payload to a Go application, you may encounter situations where you must tell the difference between
set, missing, or null fields.

For example, consider the following JSON payload:

```json
{
  "name": "Alice",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "Springfield"
  }
}
```

We can unmarshal this JSON payload using JSON tags and the following Go structs:

```go
type Person struct {
    Name    string `json:"name"`
    Age     int    `json:"age"`
    Address Address `json:"address"`
}
type Address struct {
    Street string `json:"street"`
    City   string `json:"city"`
}
```

However, we will not be able to tell the difference between these two JSON payloads:

- `{ "name": null }`
- `{ "name": "" }`

Go's zero values are not distinguishable from missing fields when unmarshalling JSON.

We can change the above struct to use pointers to identify null fields:

```go
type Person struct {
    Name    *string `json:"name"`
    Age     *int    `json:"age"`
    Address *Address `json:"address"`
}
```

However, we will still not be able to tell the difference between these two JSON payloads:

- `{ "name": null }`
- `{ }`

Both of these payloads will unmarshal into a `Person` struct with all fields set to `nil`, and we cannot distinguish
between a missing field and a field set to `null`.

One reason to distinguish between missing and null fields is to avoid overwriting existing values with `null` values.
For example, when `name` is not specified in the JSON payload, we may want to keep the existing name value in the
`Person` struct. But we may want to clear the name when `name` is defined as `null`.

## Detecting null, set, and missing JSON fields with Go

We can use custom unmarshalling logic by implementing the [Unmarshaler](https://pkg.go.dev/encoding/json#Unmarshaler)
interface to detect the difference between null and missing fields. The `UnmarshalJSON` method allows us to inspect the
JSON token stream and decide how to unmarshal the JSON payload. The critical insight is that `UnmarshalJSON` is only
called when the field is present in the JSON payload. So, we can mark a `Set` flag as `true` when the field is present
and `false` when it is not.

Here is an example implementation:

```go
type Any[T any] struct {
    Set   bool
    Valid bool
    Value T
}

// MarshalJSON implements the json.Marshaler interface.
// Only Value is marshaled, and only if Valid is true.
func (s Any[T]) MarshalJSON() ([]byte, error) {
    if !s.Valid {
       return []byte("null"), nil
    }
    return json.Marshal(s.Value)
}

// UnmarshalJSON implements the json.Unmarshaler interface.
// Set is always set to true, even if the JSON data was set to null.
// Valid is set if the JSON data is not set to null.
func (s *Any[T]) UnmarshalJSON(data []byte) error {
    s.Set = true
    s.Valid = false

    if bytes.Equal(data, []byte("null")) {
       // The key was set to null, set value to zero/default value
       var zero T
       s.Value = zero
       return nil
    }

    // The key isn't set to null
    var v T
    if err := json.Unmarshal(data, &v); err != nil {
       return err
    }
    s.Value = v
    s.Valid = true
    return nil
}
```

We used a generic type `T` to allow `Any` to work with any type. The `Valid` flag distinguishes between `nil` and
`non-nil` values. The `Set` flag is set to `true` only when the field is present in the JSON payload.

Here is how we can use the `Any` type in a `Person` struct:

```go
type Person struct {
    Name    Any[string] `json:"name"`
    Age     Any[int]    `json:"age"`
    Address Any[Address] `json:"address"`
}
```

## Testing the custom unmarshalling logic

The following example demonstrates how the `Any` type works:

```go
type Form struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

type Config struct {
    Form   Any[Form] `json:"form"`
    Member Any[bool] `json:"member"`
}

func main() {
    tests := []struct {
       description string
       JSON        string
    }{
       {
          "Nothing set",
          `{}`,
       },
       {
          "Set all fields",
          `{"form": {"name": "John", "age": 30}, "member": false}`,
       },
       {
          "Set only member field, and leave form fields unchanged",
          `{"member": true}`,
       },
       {
          "Set only the form field, and leave the member field unchanged",
          `{"form": {"name": "Jane", "age": 25}}`,
       },
       {
          "Leave all fields unchanged",
          `{}`,
       },
       {
          "Clear all fields",
          `{"form": null, "member": null}`,
       },
       {
          "Set only form field",
          `{"form": {"name": "Chris", "age": 35}}`,
       },
    }

    c := Config{}
    for _, test := range tests {
       fmt.Printf("\nTest: %s\n", test.description)
       _ = json.Unmarshal([]byte(test.JSON), &c)
       fmt.Printf("Input: %s\n", test.JSON)
       fmt.Printf("%+v\n", c)
       data, _ := json.Marshal(c)
       fmt.Printf("Output: %s\n", data)
    }

}
```

The test output will show how the `Any` type behaves when unmarshalling JSON payloads with different fields set,
missing, or set to `null`.

```
Test: Nothing set
Input: {}
{Form:{Set:false Valid:false Value:{Name: Age:0}} Member:{Set:false Valid:false Value:false}}
Output: {"form":null,"member":null}

Test: Set all fields
Input: {"form": {"name": "John", "age": 30}, "member": false}
{Form:{Set:true Valid:true Value:{Name:John Age:30}} Member:{Set:true Valid:true Value:false}}
Output: {"form":{"name":"John","age":30},"member":false}

Test: Set only member field, and leave form fields unchanged
Input: {"member": true}
{Form:{Set:true Valid:true Value:{Name:John Age:30}} Member:{Set:true Valid:true Value:true}}
Output: {"form":{"name":"John","age":30},"member":true}

Test: Set only the form field, and leave the member field unchanged
Input: {"form": {"name": "Jane", "age": 25}}
{Form:{Set:true Valid:true Value:{Name:Jane Age:25}} Member:{Set:true Valid:true Value:true}}
Output: {"form":{"name":"Jane","age":25},"member":true}

Test: Leave all fields unchanged
Input: {}
{Form:{Set:true Valid:true Value:{Name:Jane Age:25}} Member:{Set:true Valid:true Value:true}}
Output: {"form":{"name":"Jane","age":25},"member":true}

Test: Clear all fields
Input: {"form": null, "member": null}
{Form:{Set:true Valid:false Value:{Name: Age:0}} Member:{Set:true Valid:false Value:false}}
Output: {"form":null,"member":null}

Test: Set only form field
Input: {"form": {"name": "Chris", "age": 35}}
{Form:{Set:true Valid:true Value:{Name:Chris Age:35}} Member:{Set:true Valid:false Value:false}}
Output: {"form":{"name":"Chris","age":35},"member":null}
```

## Complete code on Go Playground

The complete [Go code for unmarshalling JSON null, set, and missing fields](https://go.dev/play/p/AXxgO20M0kI) is
available on the Go Playground.

## Further reading

Recently, we published an article on
[how to optimize the performance of a Go application](../optimizing-performance-of-go-app/). We benchmarked JSON
decoding vs [gob](https://pkg.go.dev/encoding/gob) decoding in that article.

In addition, we wrote about [how to read program arguments from STDIN with Go](..//get-args-from-stdin/), which is more
secure than using environment variables or command-line arguments.

Also, we explained [the difference between Go modules and packages](../go-modules-and-packages/), which is essential for
organizing and managing Go code.

## Watch how to use Go to unmarshal JSON null, set, and missing fields accurately

{{< youtube yHhk5wGNxk4 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.

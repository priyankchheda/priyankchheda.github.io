+++
date = '2025-11-15'
title = 'Understanding Go Interfaces'
series = ['design patterns']
+++

This is a placeholder post about Go interfaces. Interfaces in Go provide a way to specify the behavior of an object: if something can do this, then it can be used here.

## Why Interfaces Matter

Go's type system is structural rather than nominal. A type implements an interface simply by implementing its methods — no explicit declaration needed.

## Example

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Any type with a `Write` method of that signature satisfies `io.Writer`.

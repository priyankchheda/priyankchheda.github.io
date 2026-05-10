+++
date = '2026-02-10'
title = 'Builder Pattern in Go'
series = ['design patterns']
+++

This is a placeholder post about the builder pattern. In Go, the functional options pattern is often preferred over a traditional builder.

## Functional Options

```go
type Server struct {
    port    int
    timeout time.Duration
}

type Option func(*Server)

func WithPort(p int) Option {
    return func(s *Server) { s.port = p }
}

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

This approach scales well and keeps the API clean.

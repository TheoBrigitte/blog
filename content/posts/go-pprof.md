---
title: "How to profile your Go application with pprof"
date: 2020-01-12T23:11:58+01:00
description: "Profile your Go application with pprof"
---

## Intro

Profiling is a way to measure the performance of your application. It can help you identify bottlenecks like memory leaks and optimize your code.

In this article, I will take Grafana [Alloy](https://github.com/grafana/alloy) as an example.

## Intrument your code

Intrumenting you Go application with profiling and exposing it via http is easy, it boils down to a single line of code :

```go
import _ "net/http/pprof"
```

More information on it at [net/http/pprof](https://pkg.go.dev/net/http/pprof).

## Visualize and analyse profiling data


go tool pprof -http localhost:8080 -source_path ~/ -trim_path /go/ localhost:12345/debug/pprof/heap


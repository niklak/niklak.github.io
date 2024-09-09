---
layout: post
title:  "Go: Exploring plugin system"
date:   2024-09-09 10:56:00 +0300
categories: go golang
---


Recently, I came across another gap in my experience with Go — specifically, the plugin system in Go. Even though I've been writing in Go since version 1.9 (or maybe even 1.8), and the plugin package has been around since 1.7, my paths had never crossed with this package—I didn't have the need.

So, I decided to fill this gap for myself. The plugin package hasn't been actively developed and has significant limitations that cannot be ignored.
I propose to consider this package from the perspective of its capabilities.

### Deployment

This package allows you to load a dynamic .so library and execute its code at runtime. Such libraries need to be compiled using `go build -buildmode=plugin`.
However, there are many limitations, which are described here: https://pkg.go.dev/plugin.

I'll add that in both the plugin and the main program, not only do GOOS, GOARCH, and GOVERSION need to match, but also the version of GCC used for compiling both the plugins and the program. More broadly, the runtime environment should match as closely as possible.

Let's say you decide to run a program in Docker that uses plugins you compiled in a different environment. Problems may arise if your environment differs from the Docker image's environment that will run your program. For example, if you (or rather, I) decided to create an image based on Alpine for the program, but compiled the plugins in a non-Alpine environment and placed them in a shared folder accessible to the program. GOOS, GOARCH, and GOVERSION might match, but the plugins throw an error because the Alpine environment differs from the one where you compiled the plugins. By default, for plugins to work in Alpine, you need to install GCC (Alpine uses musl), but the installed GCC will likely differ from yours. Therefore, if you want a minimal program image, you need to ensure that the plugins are compiled in an identical environment. More broadly, there should be a way to compile plugins in an environment identical to the one used to compile the main program.
These limitations reduce the flexibility of the software architecture. If you build your software stack around the idea of using these plugins, what might you end up with?

Overall, it's inconvenient. That's a big downside.

### Performance

I can't say that the performance of a program using plugins significantly drops compared to a standard compiled program. The performance of a dynamic program is only slightly lower than that of a regular program. In my opinion, this is a big plus.

### Integration

This is straightforward — this package is only for Go programs, and you don't need to consider any other nuances. The package provides a simple and understandable programming interface for creating plugins.

### Use Cases

I've seen plugins used in web development, particularly for creating middleware for servers. Where could this be useful? Suppose you need a service with basic functionality (a step) and additional functionality that is not required in every deployment. In that case, this functionality can be extracted as a plugin, and during deployment, at some point before starting the program (or container with the program), you could compile it. Then the main program would be able to read it in its environment.

Of course, you could make it even more fun by accepting plugin code via HTTP, writing it to a temporary file, compiling it through an external call in the program's environment, and immediately reading and loading it into the program's runtime. Just kidding — don't do that.

### Live example

[![Edit niklak/go-dyn-server/main](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/p/github/niklak/go-dyn-server/main?embed=1&file=%2Fcmd%2Fdyn-server%2Fmain.go)

### Conclusion

If it is possible to apply an alternative to go plugin I would recommend using them. Take a look at gRPC approach, or wasi. gRPC calls are almost as performant as FFI calls while WASI calls are relatively slow.
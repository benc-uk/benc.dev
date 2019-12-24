---
title: Getting On The Go Kart
tags: [golang, linux, go]
image:
  feature: header/h02.svg
---
I've dabbled with the Go programming language in the past, and came away with mixed feelings. However I've resolved to a have an extra "main" language to go alongside JavaScript/Node and Python (with C#, Java and PHP being long consigned to the "only if I must bucket"), so decided to revisit Go and get some real usage of the language under my belt. In addition some recent tooling changes means it has become a lot easier to work with, so it was time for a second look.

<!--more-->

# Introduction
In this first part I'm going to focus on the tooling (i.e. compiling your code & package management) and file/project structure. Yes that's the *really* boring stuff, but it's also the part any newcomer to Go is going to bounce off first. This isn't a normal "getting started with Go" article, there's plenty of them already all over the internet. I'll be zooming on the use of ***modules***, which I don't see much discussion  about, probably as they are quite new.  

I'll tackle the many quirks of the language itself in a second follow up post. I'll only be talking about versions of Go from 1.11 onwards, for reasons that will become apparent

Note. I use Linux (WSL v2 and Ubuntu to be precise) for all my development & coding work, I'm sure everything I talk about here still applies to Windows or MacOS, but I can't guarantee it  

# Notes On Go
There are some notable aspects to Go I'd like to mention, before we carry on:
- Go compiles to a static executable binary, no DLLs, no JARs, no minified JS bundle. Just a plain old simple exe file. Which is actually quite lovely.
- Which means... Go has no runtime! 
- Go has what the maintainers refer to as an "idiomatic" approach, which affects the design of the language and also how you write your code (more in part 2)
- Originally Go didn't have any way to manage or install packages. It's been a tortuous journey via tools like `dep` and the awful vendor directory experiment. Thankfully all that has been resolved, with the introduction of modules...
  
# Getting Started & Avoiding GOPATH Hell
[Installing Go is pretty simple](https://golang.org/doc/install), the instructions have you downloading and un-taring it directly into `/usr/local`. That to me seems quite strange, and I'm sure another path would work, but it appears to be the agreed way, so don't bother changing it. I've created a simple [helper install script](https://github.com/benc-uk/ubuntu-tools-install/blob/master/golang.sh) 

The Go bin directory you installed (e.g. `/usr/local/go/bin`) must be added to your system path, which is pretty obvious. However there's also something called the GOPATH. This used to be a major pain in versions prior to Go 1.11, as everything you did including your own projects and code had to be somewhere under this GOPATH location. It was just horrible.

Version v1.11 introduced modules which for the first time let you "work outside the GOPATH", so to keep things simple you can take the defaults and all but ignore the GOPATH. The default location of GOPATH is inside your user directory, e.g. `$HOME/go` and if you're happy with that (which you should be) you don't have anything else to do, except add one line to your `.bashrc` or `.zshrc` file

```
PATH="$HOME/go/bin:/usr/local/go/bin:$PATH"
```

All things being well, you should be able to run `go version` and `go env` successfully. From here you can effectively forget all about `$HOME/go` and this GOPATH business

Note. The majority Go tutorials [including those on golang.org site](https://golang.org/doc/code.html), still place everything under the `$GOPATH/src` directory. **YOU NO LONGER NEED TO DO THIS** and I will explain why

# Starting With Go Modules
Just to avoid ambiguity, I'll introduce and clarify some common terms at this point:

- A Go ***project*** is a directory containing Go source & other files, which is usually contained in a repo
- A ***repo***, a git repository, might contain a single Go ***project***, or several, or possibly a mix of Go and non-Go projects.
- A ***module*** is a collection of related Go packages that are versioned together as a single unit, as follows
  
Go v1.11 introduced something called [modules](https://github.com/golang/go/wiki/Modules#daily-workflow), this meant for the first time your project and source code could be anywhere on your system, it also introduced a sort of package management system to the language.

To enable this feature you currently need to set an environmental variable called `GO111MODULE`, this is because the feature is considered "preliminary" but I wouldn't worry about that it will likely be a core feature in Go 1.13 so hopefully we can omit this step in a few months

Add this line to your `.bashrc` or `.zshrc` file
```
export GO111MODULE=on
```
**UPDATE** Go 1.13 has been released. If you're using 1.13 or above, you can ignore this, setting `GO111MODULE` is no longer required.

From here I assume you've got a directory you want to work from, this is your Go ***project***

You need to initialize your project as a module, using `go mod init`. If you have no plans to make your code available to others you can call your module anything you like, for example
```
go mod init mymodule
```
If you don't provide a name you will get a baffling error: `go: cannot determine module path for source directory` which is pretty unhelpful.

You might be thinking *"But I'm just writing hello world here, I'm not creating anything like a module"* just ignore the terminology and create a module for your tiny hello world test app anyhow, you can simply invent a name.

This `init` command will result in a `go.mod` file being created in the root of your project. This file is a little like project.json in Node, or .csproj in .NET Core, or even requirements.txt in Python

`go.mod` is just a text file, so you can take a look, but trust me it won't be that interesting; at this point it will just be two lines. In my experience I've rarely, if ever had need to edit a `go.mod` file


## Golden Rule of Modules
Have only one module (i.e. `go.mod` file) per ***project*** and at the top level. Note, this might not be the top level of your ***repo*** if you're working in a repo with a mix of Go and non-Go code.


> The go.mod file only appears in the root of the module. **Packages in subdirectories have import paths consisting of the module path plus the path to the subdirectory**. For example, if in a module called 'example.com/hello' we created a subdirectory world, we would not need to (nor want to) run go mod init there. The package would automatically be recognized as part of the example.com/hello module, with import path example.com/hello/world.  
> (Taken from [https://blog.golang.org/using-go-modules](https://blog.golang.org/using-go-modules))

# Code Layout & Workflow
That seemed like a lot of work for a two line file! Well that's because we're stepping through the basics here. I find most of this kind of thing becomes second nature after working with any language or tooling for a while

With your module in place you should be able to simply run `go build` and/or `go run` from your module root directory to build/test your code. I'm not going to cover the basics of the go command line, if you need a guide [RTFM](https://golang.org/cmd/go/)

Lets move on to laying out code across multiple files and folders (i.e. beyond the single file helloworld.go example). Which was probably one of my first stumbling points when using modules, and in fact Go in general

First up, I suggest following [the 'Standard Go Project Layout' style guide](https://github.com/golang-standards/project-layout) which has several well adopted conventions for how to layout a project. It's kinda opinionated (that's the point really) and I'm not a big fan of some of the directory names, but it's well adopted and will make your project look a bit more grown up

Let's dig into a few very basic use cases:

## Multiple Files
Splitting your code across multiple files **IN THE SAME PACKAGE** will just work, no need to export/import anything, your functions and even top level variables will be visible.

If you've come from a Node.js, C# and Java background this will seem quite alien, in those languages things either need to be exported (JS) or placed inside classes & namespaces with public accessability and all that old OO baggage

## Package & Directory Names
The name of your packages and the directory they reside inside do not have to match, but if they don't **YOU WILL MAKE THINGS REALLY CONFUSING**

For example I don't have to place this inside a directory called `foo/` 
```
package foo

func HelloWorld() {
}
```
But if I don't (imagine I put it in a directory called `bar/`) when I import it I'll need to use the directory name, e.g. `import mymodule/bar`, when when calling it I would need to use the 'foo' package name, e.g. `foo.HelloWorld()` It's brainbender you want to avoid, at least when first starting out


## 3rd Party External Libraries
You can simply add a dependency to an external library as a regular `import` statements in your code and go ahead & compile, for example if I wanted to use the [Gorilla mux package](http://www.gorillatoolkit.org/pkg/mux)
```go
// This imports an external 3rd party library, which will be fetched automatically 
import "github.com/gorilla/mux"
```

**THIS IS WHERE THE MAGIC HAPPENS!**   
Running `go build` will result in the external code being fetched from the remote Git repo and compiled for you. The results are cached (inside GOPATH, but I said not to worry about that), and the `go.mod` file is automatically updated with a require statement for the external library

The way Go modules handles external imports here is very neat I think, it just works! You don't need to run `npm install` or `dotnet package add` or anything like that, just update your code and compile. At some point your going to want to worry about the versions of your dependencies, I won't cover that here, but the `go get` command and the `go.mod` file has all of that covered, [there's plenty of detail in the documentation](https://github.com/golang/go/wiki/Modules). A `go.sum` file will also appear, also don't worry about it.

## Using Local Libraries / Shared Code
This is where I started to really stumble over a few things, mainly due to my expectations on how things had worked with Node.js

Firstly there are **NO RELATIVE IMPORT PATHS**, don't try to do anything like this:
```go
// None of these will ever work, don't even try it
import "../mylib/utils"
import "./foo"
import "../../this/really/wont/work"
```

Always use the module name even when trying to reference a local file/package, for example these will work:
```go
// This imports a local package in the mylib directory, assuming your project is defined in 'mymodule'
import "mymodule/mylib"
// Things can be nested of course
import "mymodule/mylib/utils"
```
If you think of the module name as the "root" level, and you go through the directories from there, **not from where the .go file with the import statement is**. Once I started to think of things this way, I had no further issues

## Examples
I've put some simple code examples of the what I've been talking about here on Github to hopefully illustrate the point more clearly  
[https://github.com/benc-uk/blog/tree/master/etc/go-basics](https://github.com/benc-uk/blog/tree/master/etc/go-basics)

## What Do I Do with These go.mod and go.sum Files?
Not much. Don't ever touch `go.sum` it's basically there to [validate checksums of the dependencies your are bringing in](https://github.com/golang/go/wiki/Modules#is-gosum-a-lock-file-why-does-gosum-include-information-for-module-versions-i-am-no-longer-using). Should you delete the file, it's automatically recreated.

As for `go.mod`, I've not seen any need to ever edit it. Should you delete it, you can quickly get back to where you started with a `go mod init` and a `go build`

You **SHOULD COMMIT BOTH FILES TO YOUR GIT REPO** so don't be tempted put them in your `.gitignore` file

If you want to know more, [the FAQ has you covered](https://github.com/golang/go/wiki/Modules#faqs--gomod-and-gosum)


# Wrap Up
In hindsight this hasn't been the most thrilling blog post! But this low level & tooling detail is generally what gives people either a good or bad first impression of a language and what prevents people from being productive.

Go modules although new, make life working outside the GOPATH simple and easy. Hopefully will provide a common footing for the language and ecosystem around it to forward. I expect they will be in the new normal for Go developers within the next 6-12 months, just like Node.js developers run `npm install` without even thinking

In a follow up post I'll get into some of the other weirdness with Go, namely the language itself :)
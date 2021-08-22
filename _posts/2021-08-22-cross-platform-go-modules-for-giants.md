---
title: Cross platform Go modules for giants
author: Kobi
comments: true
---

When your Go modules become too large.

# Tl;dr

What can you do when your Go module becomes too large for the Go tools to handle?!

When you see this?

<pre><code class="language-go">module source tree too large (max size is 524288000 bytes)</code></pre>

# Introduction

Go modules are a convenient to share code. You define a module with its version in go.mod and to Go tools automatically take care of downloading the right version, checking its integrity and compiling it. 

CGo enables integrating non-Go code by either allowing you to compile C code or just link to external libraries and call functions exported there through a convenient interface.

Let’s say you have an externally compiled library that you want to distribute with your Go module but still use the usual Go tools. Adding a compilation step can’t happen, since that would require invoking, and perhaps installing, external tools, which the Go tools doesn’t support.

The solution is to commit your library as part of the Go module codebase, which gets downloaded together with the rest of the source code.

Now, let’s say you want to support multiple platforms. Fortunately, Go has built in capabilities to include source files based on the platform compiling, so we can:

- Compile the library for each target platform
- Create a platform-dependent source file that links to the right library

This works reasonably well, with the caveat that the Go module the tools need to download becomes larger as a result.

What happens when the library become too large? When it’s a static library that didn’t optimize for size and relied on consumers just taking what they need, which the Go tools do? This is what you see if your Go module is too large:

<pre><code class="language-go">module source tree too large (max size is 524288000 bytes)</code></pre>

Concretely, if you want to support Linux, macOS, Windows on x86_64, ARMv7, ARMv8 and s390x, where applicable, that becomes a big set of libraries that can exceed 500mb. 

Why is that important? Because the maximum module size Go supports is 500mb. This is intentional to prevent denial-of-service attacks, as mentioned in [https://golang.org/ref/mod#zip-path-size-constraints](https://golang.org/ref/mod#zip-path-size-constraints).

Can we have still use the usual Go workflow and support that many platforms? The answer is yes!

We first provide an overview of the workflow when the module size is less than 500mb, and then we’ll see how to make it work with the the giant libraries case. We'll use as a study case the [celo-bls-go](https://github.com/celo-org/celo-bls-go) library, which is a Go module that links to an underlying Rust library.

# Not too big

When your library is big, but not too big, you can employ a relatively simple structure. Assuming our Go package is `bls`, we can employ the following structure:

- Main code file, `bls.go`, that contains the invocations of the different functions from the underlying library using CGo. This comes together with a header file `bls.h` that describes the imported functions. For example:

<pre><code class="language-go">func GeneratePrivateKey() (*PrivateKey, error) {
	privateKey := &PrivateKey{}
	success := C.generate_private_key(&privateKey.ptr)
	if !success {
		return nil, GeneralError
	}

	return privateKey, nil
}</code></pre>

- Per-platform file that only describes the linking directives, and has a build directive causing it to be compiled only if targeting this platform. For example, `bls_darwin64.go`:

<pre><code class="language-go">// +build darwin,amd64,!ios

package bls

/*
#cgo LDFLAGS: -L${SRCDIR}/../libs/x86_64-apple-darwin -lbls_snark_sys -ldl -lm
*/
import "C"</code></pre>

Not too bad! 

# Is it all doomed?

The solution described in the previous section has worked for us for quite a while, until very recently when our underlying Rust library grew bigger and we wanted to support more platforms, such as the recent M1, 32-bit ARM and s390x.

We realized that it's a harder case quickly. Remember, our goal is to still be able to keep using the usual Go tools. 

Some solutions that we've tried and and failed or we rejected include:

- Downloading the library on demand per-platform. Go didn't seem to provide a way to do it.
- See if there's a way to remove the 500mb module size limit. We didn't find a way with the usual Go tools, since it's enforced on the client side.
- Reducing the library size. It's possible, but not a long-term or scalable solution.
- Have users install the library through other means, such as `apt`, like other common libraries like `openssl`do. Possible, but significantly worse developer and user experience for our specific library.
- Require developers who consumed the module go through a custom vendoring process. We didn't even test this.

So... now what?

# Big friendly giant

Having realized it's not going to be an officially-described method or something that is widely employed, we embarked on creating our own.

Since our problem is with the library size when distributing all platforms, we first handled that:

- Distribute the per-platform libraries to packages, such as `celo-bls-go-linux`, `celo-bls-go-macos`, etc
- In each package, have a similar structure as described previously. Namely, a main source file and a per-platform source file that has the linking and build directives. This is only for the platforms supported in this package.

This is still not seamless to consumers, since they would have to choose which `celo-bls-go-PACKAGE` to choose, depending on their platform. 

We then introduce a main module, `celo-bls-go`, that has a `router` per package called `bls_PACKAGE.go`:

- It has an appropriate build directive, which is the union of the build directives of the platforms included in the package.
- It imports the correct package, `celo-bls-go-PACKAGE`.
- It has wrapper functions for all public functions, consts and vars in the package. For example:

<pre><code class="language-go">func HashDirectFirstStep(message []byte, hashBytes int32) ([]byte, error) {
    return blsRoute.HashDirectFirstStep(message, hashBytes)
}</code></pre>

To ease in creating this structure, that unfortunately includes a lot of similar code, we then introduced the `distribute`program that consumes source templates, router templates and a `platforms.json` and produces the per-package files. The `platforms.json` file looks like this:

<pre><code class="language-go">{
	"linux": [{
		"name": "linux_arm64",
		"buildDirective": "linux,arm64",
		"linkageDirective": "-L${SRCDIR}/../libs/aarch64-unknown-linux-gnu -lbls_snark_sys -ldl -lm",
		"libDirectories": ["aarch64-unknown-linux-gnu"]
	}, ...
  ],
  "macos": [{
		"name": "darwin64",
		"buildDirective": "darwin,amd64,!ios",
		"linkageDirective": "-L${SRCDIR}/../libs/x86_64-apple-darwin -lbls_snark_sys -ldl -lm",
		"libDirectories": ["x86_64-apple-darwin"]
	}, ...
  ],
  ...
}</code></pre>

The code for `distribute.go`can be viewed here: [https://github.com/celo-org/celo-bls-go/blob/c0f37c3a9a6cfb448a152f785391efd42f03f895/cmd/distribute/distribute.go](https://github.com/celo-org/celo-bls-go/blob/c0f37c3a9a6cfb448a152f785391efd42f03f895/cmd/distribute/distribute.go)

This is it! 

## Final thoughts

The solution we described works reasonably well and achieves what we set out to achieve.

I imagine there's a bunch of ways it could improved. One example is that we could use the `parser` package to automatically derive the router rather than manually writing it.

I'd love to hear feedback and improvement suggestions!



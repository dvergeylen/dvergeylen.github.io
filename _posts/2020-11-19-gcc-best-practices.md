---
layout: post
title:  "Gcc / Clang best practices"
date:   2020-11-19 12:30:00 +0200
categories: programming compiler
permalink: /gcc-best-practices
---

#### Introduction
Compiling with `gcc` or `clang` from the command line can be scary at first but with a few tricks it can prove to be very useful and thrilling to use.

In this blog post / reference page I'll describe what I consider best practices, at least some tricks I use in my C/C++ projects and that saved me hours of headache.

I'll regularly update this post as far as I find time to cover the needed material to become confident when compiling with `gcc` / `clang`.

#### gcc ⚔️ clang ⚔️ ...?
I've heard this many times. Which one is better / faster / stronger? Ill-posed problem! If you are trying to oppose them against each other you are doing it wrong! 🙅

They can both reveal sometimes better, sometimes worse. Same applies to other compilers. I'm not talking about the binaries they produce (although observing significant performance or size differences between binaries from multiple compilers can put under the spotlight some code weaknesses or at least some weirdness because that mean they all not interpret your code the same way → most probably code smell 🧦) but about the (debug) information they can provide.

The best practice I use is that I compile my code with them **both** (well, not at the same time obviously). It's particularly useful when facing compiling errors you don't fully understand, because one can sometimes output a clearer error than the other and vice-versa. Their respective static analyzers (starting at GCC 10+) will help you find these bugs and give you why this is illegal code. That obviously doesn't mean your code will work once compiling without warnings / errors but it's at least a good start!

Don't oppose compilers between each other, embrace their differences. Having your code compiling on many of them is a very good sign your code is portable and respects standards.

### Useful gcc flags
You can install `gcc` man-pages via `sudo apt install gcc-doc`.

GCC compilation flags are grouped by usage in `man gcc`. Among all the categories, the first one to consider is: **Warning Options**.

It lists the flags controlling compilation warnings. There is A LOT of flags, you don't need to know them all. Some are very specific and are only useful if you (cross-)compile to some unusual architecture or deal with some hardware which behavior can confuse the compiler if not configured properly.

Here is a list of very useful warnings, covering most use-cases when used together:
* `-Wall` show all warnings. It won't activate ALL warnings as the name suggest but already a lot of them;
* `-Wextra` activate some extra warnings;
* `-Werror` treat warnings as errors. YOU WANT THIS. I always consider *"A warning at compile time is an error an runtime"*. Trust me, it will save you some time!
* `-Wfatal-errors` stop at first encountered error. Don't let `g++` be *that* verbose 🙉;
* `-Wpedantic` Issue all the warnings demanded by strict ISO C and ISO C++; reject all programs that use forbidden extensions, and some other programs that do not follow ISO C and ISO C++.
* `-pedantic-errors` Give an error whenever the base standard requires a diagnostic.

<p class="warning">
  🧐 Always consider <code>-Wpedantic</code> and <code>-pedantic-errors</code> as DEBUG flags.<br />Never put <code>-Wpedantic</code> in production code, as the outputted warnings can vary from standard to standard. Simply upgrading your compiler version can make it consider a more recent standard as the new default and some new pedantic warnings / errors might appear, preventing your code to compile. Suddenly your customer starts yelling at you as she can't compile your code anymore while nothing has changed on your side! Beware 🛡️ !
</p>

Aside from the **Warning Options** category flags, you'll most probably need three other ones to make your code compile in the real world: `-I`, `-L` and `-l`. These three flags tell the compiler where to look for header files (describing libraries' APIs), libraries locations and which libraries to link against, respectively.

As stated in `man gcc`, it makes a difference where in the command you write `-l`:

>the linker searches and processes libraries and object files in
>the order they are specified.  Thus, foo.o -lz bar.o searches
>library z after file foo.o but before bar.o.  If bar.o refers to
>functions in z, those functions may not be loaded.

A typical `gcc` line will be:

```bash
gcc -Wall -Werror ... -Ipath/to/header/files -Lpath/to/library/locations -lmylib
```

You can use `-I`, `-L` and `-l` flags multiple times.

#### pkg-config to the rescue
Knowing headers / lib files locations can be difficult when compiling on multiple platforms / OSes. `pkg-config` can help you by outputting compiler flags configured with their correct location on the local system. For instance:

```bash
# Library cURL, named libcurl
pkg-config --libs --cflags libcurl
# Will output:
-I/usr/include/x86_64-linux-gnu -lcurl
```

Very handy when writing a Makefile (see below)!

### Reference Makefile
Here is a minimal Makefile as reference. Say `libcurl` is a dependency we need to link against.

Build your program by typing `make` in the same directory.

```makefile
EXEC=my_output
CC=g++
DEBUGFLAGS=-g -Wpedantic -pedantic-errors
CPPFLAGS=-Wall -Werror -Wextra -Wfatal-errors
IFLAGS=$(shell pkg-config --cflags)
LFLAGS=$(shell pkg-config --libs)

all: $(EXEC)

$(EXEC):
	$(CC) $(DEBUGFLAGS) $(CPPFLAGS) $(IFLAGS) $(LFLAGS) main.cpp -o $(EXEC)

clean:
	rm -f *.o $(EXEC)
```

* `EXEC` is the executable name (the final output);
* `CC` is the C(++) compiler you use. `g++` is part of `gcc`;
* `DEBUGFLAGS` are the ... debug flags. `-g` makes the compiler to output symbols in the binary, allowing a debugger to run it and set breakpoints. Pedantic flags are here listed as debug flags as said above;
* `CPPFLAGS` are the compilation flags;
  * You might want to add the standard your code is written in via `-std=c2x`.
* `IFLAGS`: list of directories to be searched for header files during preprocessing. Contains one or multiple `-I` flags followed by relative or absolute paths.
* `LFLAGS`: list of libraries to link against.

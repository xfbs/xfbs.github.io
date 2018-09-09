---
layout: post
title:  "Tracing in Linux and macOS"
date:   2018-02-22
author: xfbs
---

If you’re coming from Linux, you may be familiar with the `ptrace` family of commands — `strace` and `ltrace`. If you’re coming from macOS, you may have had brief encounters with `dtruss` or `dtrace`, instead.

If you haven’t heard of them before or haven’t had the chance to play with them, this post is for you. I’m going to show you what they do and why they are important tools to know.

## Tracing syscalls

Let’s say you have an application, a small program, and you want to know analyze what it does. In this example, I’ll use a small program that checks if a file is present — if it’s not present, it will fail with a warning. I am using the `access` function, which is a POSIX API, to check if a file exists.

###### File safe.c, lines 0–31:

```c
#include 
#include 
#include 
#include 
#include

bool validate(char *seed)
{
  char *file = strdup(seed);

  for(size_t pos = 1; seed[pos] != '\0'; pos++) {
    // security by obscurity
    file[pos] = 65 + ((seed[pos] ^ 33 + file[pos-1]) % 26);
  }

  bool valid = access(file, F_OK) == 0;
  free(file);

  return valid;
}

int main(int argc, char *argv[])
{
  if(!validate(".secret_file_seed")) {
    fprintf(stderr, "error: secret file is missing.\n");
  return 1;
}

  printf("congratulations!\n");
  return 0;
}
```

For convenience, here’s a minimal `Makefile` to build this program.

###### File Makefile, lines 0–2:

```
# build all binaries (default target).
all: safe
```

### On Linux

I’m using a fresh Ubuntu VM to perform these tests. You’ll need some packages to compile this example — `make`, a compiler (`gcc` or `clang` will do just fine), and optionally `musl` and `musl-gcc`.

    $ apt update
    $ apt install -y build-essential musl musl-dev musl-tools

Compiling and running it (if you don’t want to use musl, just remove the `CC=musl-gcc`), we get:

    $ CC=musl-gcc make linux
    musl-gcc safe.c -o safe
    $ ./safe
    error: secret file is missing.

Why musl? Statically linking to musl instead of dynamically linking to your system `libc` means that the program will need to do fewer syscalls at startup.

So, what is the name of the file that it’s trying to access? That’s where [`strace`](https://strace.io) comes in! Basically, it intercepts and prints all syscalls that a program performs. That means we can sit back and watch what the program is doing — which files it is opening, what it is writing to those files, and much more. Here’s what strace tells me about my program when I run it on the Ubuntu machine:

    $ strace ./safe
    execve("./safe", ["./safe"], [/* 23 vars */]) = 0
    arch_prctl(ARCH_SET_FS, 0x7f0001204088) = 0
    set_tid_address(0x7f00012040c0) = 8456
    mprotect(0x7f0001202000, 4096, PROT_READ) = 0
    mprotect(0x600000, 4096, PROT_READ) = 0
    access(".IPSGNBIMHFCHAHMK", F_OK) = -1 ENOENT (No such file or directory)
    writev(2, [{"", 0}, {"error: secret file is missing.\n", 31}], 2error: secret file is missing.
    ) = 31
    exit_group(1) = ?
    +++ exited with 1 +++

Immediately you can see the `access` syscall, with `.IPSGNBIMHFCHAHMK`. That means the program is asking the kernel if a file with this name exists. The kernel replies with `ENOENT`, meaning that it doesn’t. What if we created that file and ran the program again?

    $ touch .IPSGNBIMHFCHAHMK
    $ ./safe
    congratulations!

So, `strace` can be used to snoop on a program and watch what it’s doing to the system — all of the syscalls it does will be in the output.

### On MacOS

Compilation on macOS works basically the same way as it does on Linux — but now we won’t be able to use musl, since it’s not supported. Instead, we’ll compile as usual with:

    $ make safe
    cc safe.c -o safe

Unfortunately, `strace` itself doesn’t exist on macOS. That would be too easy, wouldn’t it? Instead, there is something else, called *DTrace*, which is actually fairly comprehensive and complicated — there is [a book](http://dtrace.org/guide/preface.html#preface) on it, there are [quite](https://8thlight.com/blog/colin-jones/2015/11/06/dtrace-even-better-than-strace-for-osx.html) a [few](https://blog.wallaroolabs.com/2017/12/dynamic-tracing-a-pony---python-program-with-dtrace/) blog [posts](https://hackernoon.com/running-a-process-for-exactly-ten-minutes-c6921f93a4a9) about it, but don’t be intimidated yet.

You don’t need to know all of `DTrace` to be able to use it, all you need to know is which fontends do what. And the `dtruss` font-end happens to do basically the same as `strace`, meaning that it’ll show you which syscalls a binary performs. Let’s try it.

    $ dtruss ./safe
    dtrace: failed to initialize dtrace: DTrace requires additional privileges

Well, DTrace doesn’t work the same way as `strace` does, in spite of their similar naming. While `strace` just politely asks the kernel to monitor a process, `DTrace` hooks directly into the kernel, meaning that you potentially have access to every secret of every user, and you can actually break things (if you try really, *really* hard). Needless to say, `DTrace` and any related tools require `root` privileges to use.

    $ sudo dtruss ./safe | tail -n 10
    issetugid(0x101B2F000, 0x88, 0x1) = 0 0
    getpid(0x101B2F000, 0x88, 0x1) = 38431 0
    stat64("/AppleInternal/XBS/.isChrooted\0", 0x7FFF5E0D9D48, 0x1) = -1 Err#2
    stat64("/AppleInternal\0", 0x7FFF5E0D9CB8, 0x1) = -1 Err#2
    csops(0x961F, 0x7, 0x7FFF5E0D97D0) = -1 Err#22
    sysctl(0x7FFF5E0D9B90, 0x4, 0x7FFF5E0D9908) = 0 0
    csops(0x961F, 0x7, 0x7FFF5E0D90C0) = -1 Err#22
    proc_info(0x2, 0x961F, 0x11) = 56 0
    access(".IPSGNBIMHFCHAHMK\0", 0x0, 0x11) = -1 Err#2
    write_nocancel(0x2, "error: secret file is missing.\n\0", 0x1F) = 31 0

If you don’t pipe the output through `tail` (which you can try, if you are curious), you’ll get a lot of noise from the system setup routines, which we aren’t really interested in at this point. And just like in the `strace` example on Linux, we can see the `access` system call! With that information, the binary can be made to run on macOS, too:

    $ touch .IPSGNBIMHFCHAHMK
    $ ./safe
    congratulations!

There’s just one little gotcha with DTrace, or rather with macOS: You can’t, by default, trace builtin utilities, eg. anything in `/bin` or `/usr/bin`:

    $ sudo dtruss /bin/ls
    dtrace: failed to execute pp: dtrace cannot control executables signed with restricted entitlements
    $ sudo dtruss /usr/bin/git
    dtrace: failed to execute pp: dtrace cannot control executables signed with restricted entitlements

What is going on there? This has something to do with the *System Integrity Protection* that Apple introduced. Apparently, there are a few things [not working under SIP](https://8thlight.com/blog/colin-jones/2017/02/02/dtrace-gotchas-on-osx.html). The only workaround that seems to be working for me is to manually copy whatever you are trying to trace to a different folder, like so:

    $ cp `/usr/bin/which ls` .
    $ sudo dtruss ./ls | tail -n 10
    getdirentries64(0x5, 0x7FD761001000, 0x1000) = 392 0
    getdirentries64(0x5, 0x7FD761001000, 0x1000) = 0 0
    close_nocancel(0x5) = 0 0
    fchdir(0x4, 0x7FD761001000, 0x1000) = 0 0
    close_nocancel(0x4) = 0 0
    fstat64(0x1, 0x7FFF56D61AB8, 0x1000) = 0 0
    fchdir(0x3, 0x7FFF56D61AB8, 0x1000) = 0 0
    close_nocancel(0x3) = 0 0
    write_nocancel(0x1, ".git\n.gitignore\nMakefile\nls\npass.c\nsafe\nsafe.c\ntracing-linux-macos.lit.md\ntracing-linux-macos.md\n\004\b\0", 0x61) = 97 0

## Tracing library calls

What if we are not interested in syscalls, but we’d much rather know what calls a program does to a library, like the standard library or `zlib`? Let’s have a look at this little program right here. It taks a passphrase as argument, checks if the passphrase is correct, and returns a message depending that check.

###### File pass.c, lines 0–36:

```c
#include 
#include 
#include 
#include

bool check(const char *passphrase) {
  unsigned char data[] = {
    120, 218, 43, 72, 77, 204, 43, 45,
    41, 86, 72, 44, 74, 85, 40, 73,
    77, 206, 200, 203, 76, 78, 204, 201,
    169, 84, 200, 73, 77, 47, 205, 77,
    45, 102, 0, 0, 204, 161, 12, 27
  };

  size_t output_len = 50;
  unsigned char output[output_len];

  uncompress(output, &output_len, data, sizeof(data));

  return strcmp((char *) output, passphrase) == 0;
}

int main(int argc, char* argv[]) {
  if(argc < 2) {
    fprintf(stderr, "error: no passphrase provided.\n");
    return 1;
  }

  if(!check(argv[1])) {
    fprintf(stderr, "error: wrong passphrase.\n");
    return 1;
  }

  printf("congratulations!\n");
  return 0;
}
```

Once again we need to add a target to the `Makefile` for this:
We’ll need to add a target to the `Makefile` to be able to compile this.

###### File Makefile, lines 2–3:

```
all: pass
```

Since this program needs to be linked with `zlib`, we’ll have to tell make about that, too:

###### File Makefile, lines 29–33:

```
# compile 'pass' and link libz.
pass: LDFLAGS += -lz
pass: pass.o
$(CC) -o $@ $< $(LDFLAGS)
```

### On Linux

To get this example to compile under ubuntu, it needs `zlib`. If zlib isn’t installed already, just install it with

    $ apt install libz-dev

Next, we can go right ahead and compile everything using the rule we just created.

    $ make pass
    cc -c -o pass.o pass.c
    cc -o pass pass.o -lz

When we run `pass`, we will see that it doesn’t work:

    $ ./pass
    error: no passphrase provided.

    $ ./pass "a passphrase"
    error: wrong passphrase.

Oh well. What now? `ltrace` to the rescue! Similar idea as `strace` — but instead of snooping on the syscalls the binary does, we’ll silently record and spit out all the library calls it does. That includes both `zlib` library calls and `libc` library calls!

    $ ltrace ./pass
    __libc_start_main(0x4008ba, 1, 0x7fff82c4d0a8, 0x400950
    fwrite("error: no passphrase provided.\n", 1, 31, 0x7f0a7c7cd540error: no passphrase provided.
    ) = 31
    +++ exited (status 1) +++

Oh well. That’s not terribly useful, is it? I guess we should give it a (wrong) passphrase to see what it does.

    $ ltrace ./pass "a passphrase"
    __libc_start_main(0x4008ba, 2, 0x7ffe3c481eb8, 0x400950
    uncompress(0x7ffe3c481d00, 0x7ffe3c481d58, 0x7ffe3c481d70, 40) = 0
    strcmp("peanuts are technically legumes", "a passphrase") = 15
    fwrite("error: wrong passphrase.\n", 1, 25, 0x7f47cb184540error: wrong passphrase.
    ) = 25
    +++ exited (status 1) +++

As you can see, the program output is a little bit mangled with the `ltrace` output, for this example it’s fine because we can still see what’s going on, but you can tell ltrace to dump it’s output to a file. You can also filter which calls or which libraries it should trace, it has a bunch of useful options. But what we are looking for is there already and very visible, from the `strcmp` call we can see that it’s comparing the string that we passed as argument with `"peanuts are technically legumes"`. It seems like that is the string it’s looking for — let’s have a look:

    $ ./pass "peanuts are technically legumes"
    congratulations!

That was easy, wasn’t it?

### On MacOS

Compiling this under macOS is exactly the same as under Linux, with

    $ make pass
    cc -c -o pass.o pass.c
    cc -o pass pass.o -lz

However, once again we don’t have `ltrace` on macOS. And there isn’t really a direct equivalent to it — this is the part where we have to play around with DTrace. It took me a while to figure this out. Thankfully, there were a few useful [articles](https://www.joyent.com/blog/bruning-questions-debugging) and obviously, the [book](http://dtrace.org/guide/preface.html#preface).

The way `dtrace` works is that it offers problems — lots of them, actually. You can trace the probes themselves, or you can attach functions to them. I won’t really go into much detail about DTrace, there is simply too much, and I don’t understand all of it well enough yet to be able to explain it.

To trace all function calls in pass, you could do:

    $ sudo dtrace -F -n 'pid$target:pass::entry' -n 'pid$target:pass::return' -c "./pass hello"
    
    trace: description 'pid$target:pass::entry' matched 2 probes
    dtrace: description 'pid$target:pass::return' matched 2 probes
    error: wrong passphrase.
    dtrace: pid 54794 has exited
    CPU FUNCTION
    0 -> main
    0 -> check
    0 <- check
    0 &1 | tail -n 10
    
    0 264352 _platform_strcmp:entry __PAGEZERO __TEXT
    0 264352 _platform_strcmp:entry __TEXT __TEXT
    0 264352 _platform_strcmp:entry __DATA __TEXT
    0 264352 _platform_strcmp:entry __LINKEDIT __TEXT
    0 264352 _platform_strcmp:entry __PAGEZERO __TEXT
    0 264352 _platform_strcmp:entry __TEXT __TEXT
    0 264352 _platform_strcmp:entry __DATA __TEXT
    0 264352 _platform_strcmp:entry __LINKEDIT __TEXT
    0 264352 _platform_strcmp:entry peanuts are technically legumes hello

And once again, from this we can tell that the ‘secret’ passphrase is *peanuts are technically legumes*, which is easily comfirmed by running:

    $ ./pass "peanuts are technically legumes"
    congratulations!

## Conclusion

Being able to easily trace syscalls or library calls can be super handy when debugging. The `strace`, `dtruss` and `ltrace` utilities are definitely a must-have in a programmer’s toolbelt, even if many things can also be done in a debugger. DTrace however is a totally different beast. It takes some work to understand it, I *barely* scratched the surface of what it can do, but when you do have a grasp of it I think it’s a lot more powerful than a debugger or any of the other tools, because you can hook into *anything*.

If you want to play around with the code from this article, you may get it by running

    $ git clone https://gitlab.com/xfbs-blog/tracing-linux-macos.git

Feel free to send in corrections or suggesions.

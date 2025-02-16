---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: HPC Environments
info: |
  Getting started
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# HPC Environments

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: slide-left
---

# What is an Environment?


<br>

Before starting, what is your environment?

<br>

- I load `PrgEnv-gnu`, then `cray-python`, then `cd $HOME/magic` and `source activate.sh`
- I am using a virtual environment that I created 2 years ago - it has worked all this time
- I set `LD_LIBRARY_PATH` in by `.bashrc` - I can't remember why I do that - does that matter?
- I just copied the modules that I used on Daint
- I am using a container
<br>
<br>

We all intuitively have some understanding of what an environment is....
<br>
<br>

But it helps to stop and ask - **what is an environment?**


<!--
a comment
-->

---
transition: slide-left
---

# Environment Variables

are everywhere...

## they affect runtime behavior of applications

```bash
export PATH=$HOME/.local/bin:$PATH
export LD_LIBRARY_PATH=/apps/empa/cp2k/lib64
export XDG_CONFIG_HOME=$HOME/.local/$(uname -m)
export LC_ALL= # ... also affects build time.
```

## they control how software is built (we care about this)

They can impact how `cmake`, `meson`, `./configure`, `pip install` and friends behave

```bash
export CUDA_HOME=/opt/nvidia/cuda/12.4
export CFLAGS="-fopenmp"
```

## they are a shadowy cabal...

.. that exert [arcane and unfathomable](https://www.gnu.org/software/gettext/manual/html_node/Locale-Environment-Variables.html) control over the daily affairs of mortal users.

---
layout: two-cols
layoutClass: gap-2
---

# What are Environment Variables, actually?
<br>

**A set of per-process global variables**
<br>

```c
extern char **environ; // who allocates this? it depends!
```
A null-terminated list of pointers to strings:
```c
"NAME=VALUE"
```

No hash table, entries are not alphabetically sorted, and the only way to find the number of entries is to iterate over the entire list until we hit `NULL`.


::right::

## `glibc/stdlib/getenv.c`

```c
char *getenv(const char *name) {
  size_t len = strlen(name);
  char **ep;
  uint16_t name_start;

  if (__environ == NULL || name[0] == '\0')
    return NULL;

  // single character env var tests removed for clarity

  name_start = *(const uint16_t *)name;
  len -= 2;
  name += 2;

  for (ep = __environ; *ep != NULL; ++ep) {
    uint16_t ep_start = *(uint16_t *)*ep;
    if (name_start == ep_start
        && !strncmp(*ep + 2, name, len)
        && (*ep)[len + 2] == '=')
      return &(*ep)[len + 3];
  }
  return NULL;
}

```

---
layout: two-cols
layoutClass: gap-2
---

# What could possibly go wrong?

The implementation of `setenv()` is more involved.

[`glibc/stdlib/setenv.c`](https://github.com/lattera/glibc/blob/master/stdlib/setenv.c#L251)

* Come for the implementation, but stay for the [comments](https://github.com/lattera/glibc/blob/master/stdlib/setenv.c#L301-L303).

Unlike `getenv`, calls to `setenv` are guarded by a mutex
* a deadlock can be created when `setenv()` is called at the same time as `fork()` and the mutex state is [lazily copied](https://rachelbythebay.com/w/2014/08/16/forkenv/) into the child process.


::right::

However, calls to `getenv` are not guarded with a mutex.
* `getenv` might read invalidated environment
* there are at least 4 different ways that a segfault will manifest

> **fun fact** environment variables are thread safe in Windows. This is one of the main causes of [crashes](https://ttimo.typepad.com/blog/2024/11/the-steam-client-update-earlier-this-week-mentions-fixed-some-miscellaneous-common-crashes-in-the-linux-notes-which-i-wante.html) in the Steam for Linux port.

Environment variables is fraught with danger.
From the [glibc user manual]()

> Modifications of environment variables are not allowed in multi-threaded programs

Never use environment variables in order to ensure that your application won't crash.


---
layout: two-cols
layoutClass: gap-16
---

# We live in the real world

Environment variables can't be avoided
* often the only way to communicate some information at runtime to some applications and libraries.
* some dependencies will use them

For example, [OpenSSL uses them gratuitously](https://docs.openssl.org/3.1/man7/openssl-env/), but that's okay!

> The OpenSSL libraries use environment variables to override the compiled-in default paths for various data. To avoid security risks, the environment **is usually not consulted** when the executable is set-user-ID or set-group-ID.

---
layout: two-cols
layoutClass: gap-16
---

# How to avoid environment variables?

Environment variables are mutable global variables. Treat them as such.

In C++ we try to make functions _pure_

```cpp
string default_path;
void foo()
{ /* use default_path */ }
int main(int argc, char** argv) {
    default_path = set_params(argv);
    foo();
}
```
```cpp
struct params {string default_path;};
void foo(const string& default_path)
{ /* use default_path */ }
int main(int argc, char** argv) {
    const params = set_params(argv);
    foo(params.default_path);
}
```

::right::

```bash
# instead of setting environment variables
export MYAPP_MAXITERS=100
export MYAPP_COLOR=0
myapp

# pass them as arguments
myapp --max-iters=100 --no-color

# if you have many configuration options,
# use a configuration file
vim $XDG_CONFIG_HOME/myapp/config
./myapp --no-color
```

Both approaches are more **declarative**: the environment is concretely defined at time of execution as CLI arguments or a set configuration file.

---
---

# How to use environment variables

Sometimes we have to use environment variables

* to comply with a standard, e.g. `XDG_CONFIG_HOME` or [`NO_COLOR`](https://no-color.org/)
    * one of my personal bugbears is applications that still ignore this variable
    * looking at you [Docker](https://github.com/moby/moby/issues/20693) (JFrog, Podman, and friends all use `$HOME/.docker`)
    * that 8 year old ticket illustrates the importance of making the right decision early is so important.
* because you maintain widely used code that already uses environment vars.
* because your users demand it


---
---

# What is the scope of environment variables

Applications are "started" using the `execve()` system call

```c
#include <unistd.h>
int execve(const char *path, char *const argv[], char *const envp[]);
```

> `execve()` does not return on success, and the text, initialized data, uninitialized data (bss), and stack of the calling process are overwritten according to the contents of the newly loaded program.

https://lwn.net/Articles/630727/

---
layout: two-cols
---

# Can I change the environment of my calling environment?

If a process can't modify it's parents environment, how does `module load` do it?

### Backend that generates

```c
// set-message_.c
#include <stdio.h>
int main() {
  printf("export MESSAGE='all your env are ours'\n");
}
```

```bash
hello world
```

::right::
```bash
> set-message() { eval "$(set-message_)"; };
> export -f set-message
> type set-message
set-message is a function
set-message ()
{
    eval "$(set-message_)"
}
> set-message
> echo $MESSAGE
all your env are ours
```

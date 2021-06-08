# pause

A minimal program and Docker image that literally does nothing

## Backstory
It's not too uncommon to want to run a docker image indefinitely without it really doing anything, perhaps to exec into it later. 
There are several common ways to accomplish this, the most obviuos being `sleep 31557600` but this only sleeps for a year, and having that arbitrary number doesn't feel right.
My preferred way before I went down this rabbit hole was `tail -f /dev/null` or it's sibling `read /dev/null` which both also do nothing forever but feels like a mean trick to to poor patient process waiting eternally for some bytes to read.

What method is used of course depends on what tools you have at your disposal in the container. If it's a full linux base container like centos or debian, `sleep` and `tail` are readily available so there's no problem. But doesn't it feel like overkill to pull in the /bins and /libs of an entire OS just to do nothing?

## How small can we go?
[Scratch](https://hub.docker.com/_/scratch/) containers let us create a docker image with absolutely no extra stuff. 
No shell, no utilities, no libraries, nothing. The catch is we need a staticly compiled binary to be able to actually run anything since there are no libraries in the container's filesystem.

### Version 1: c
My first attempt was to create a tiny c program to just loop:
```c
int main() {
  while (1) {
  }
}
```
...compiled with the `-static` and `-nostdlib` flag in gcc:
```bash
gcc loop.c -nostdlib -static -o loop
```
This results in a file size of **OVER 9000!!** bytes:
```
wc -c loop
9184 loop

```
Not bad, but that still seems like a lot for doing nothing, doesn't it? Also, there's another problem: looping is not the same as doing nothing. Our program with just a simple while loop uses 100% of a CPU!
```
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                             
 138807 jschmidt  20   0    1012      8      0 R 100.0   0.0   2:06.60 loop    
 ```

So busy waiting is not what we want. Fortunately, it's also not what operating systems want processes to do so they provide mechanisms for processes to not be doing things _all the time_.
In linux, one such too for this is [pause](https://linux.die.net/man/2/pause). 
Pause is a linux system call that tells the kernel "wake me up when you've got someting interesting".

Our new program, pause.c accomplishes the same thing (nothing) with 100% less busy waiting:
```
#include <unistd.h>

int main() {
    pause();
}
```
However, we're now using a standard libary function which means we can't get away with using `-nostdlib` to cut down our executable size. In any case, we won't have to worry about that much longer...

### Version 2: assembly
While there are some further optimizations we could do while still using gcc, the most sure way to get the smallest possible executable size will be to write our own assembly.

Linux system calls in x86 assembly are done by setting the code you want in the `eax` register, arguments in `ebx`, etc. and calling `int 0x80`. 
For `pause`, the code is `0x1d` (29).
```
mov     eax, 0x1d                       ; sys_pause()
int     0x80                            ; system call
```


I had some early experiments using `gas` and `ld` separetely to create an executable and was astonished to find the resulting files were huge! (ok not actually huge but still thousands of bytes when my program code was literally only a few).
It was at this point I stumbled in to the inner workings of [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) and found this excellent article on how to make the smallest linux executable possible: https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html
The [final version](./pause.asm) is based on the example there but does not (yet) use all of the tricky minimization techniques such as overlapping the sections of the ELF section and sneakily omitting things the OS doesn't technically check.

Total size? 91 bytes.
```
➜  pause git:(main) ✗ nasm -f bin -o pause pause.asm                  
➜  pause git:(main) ✗ wc -c pause
91 pause
```
Not too bad 😎

## I call it "wheel"
So after doing all this, I discovered there is an official pause image from scratch as part of the Kubernetes project itself: https://github.com/kubernetes/kubernetes/tree/master/build/pause

...but...mine is smaller!
```
docker image ls pause     
REPOSITORY        TAG     IMAGE ID      CREATED            SIZE
localhost/pause   latest  686d78f0ed5c  About an hour ago  3.36 kB
k8s.gcr.io/pause  3.1     da86e6ba6ca1  3 years ago        749 kB
```

Maybe someday I'll dig into where those other 3.35kB are coming from in the docker image and truly make the smallest image possible...


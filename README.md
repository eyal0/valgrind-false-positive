# valgrind-false-positive
**tl;dr:** A deep investigation into a Valgrind false-positive that took me like 15 hours to figure out.

# Intro

I maintain [pcb2gcode](https://github.com/pcb2gcode/pcb2gcode), which is a program that I took over from the original authors a long time ago.  It takes a [gerber file](https://en.wikipedia.org/wiki/Gerber_format), which is a normal way to save a PCB design, and converts it into milling instructions for your 3D printer or CNC milling machine.

3 days ago a PR failed testing with an error from Valgrind.  The error says:

```
==28297== Conditional jump or move depends on uninitialised value(s)
```

That means that Valgrind has detected that I tried to use a value for a branch.  Like if I had written this:

```c
int x; // Left uninitialized
if (x > 5) { do_it(); }
```

Valgrind thinks that I did something like that so it's letting me know that my program will have unpredictable behavior.  So I went back to my most previous successful CI run and:

![Re-run all jobs](https://user-images.githubusercontent.com/109809/109353238-56d1ef80-7839-11eb-8d3a-245b366102cb.png)

It failed.

![Job failure](https://user-images.githubusercontent.com/109809/109353324-76691800-7839-11eb-8240-8a12d79bf6fa.png)

It only failed with `clang`.

My CI grabs Valgrind from sources at the head of the master branch (which is probably a bad idea).  The CI [caches](https://github.com/eyal0/cache) it so I figured that maybe the cache expired, I got a new version, and then Valgrind reported an error.

Nope.  Old versions of Valgrind fail just as bad as new ones.  What's worse, it doesn't fail on my computer, even when I use clang!

So if it's not the version of Valgrind, maybe it's the verson of clang?  I luckily had the foresight to output the version number of most of my dependencies in my CI runs and sure enough, passing vs failing:

![clang version 9](https://user-images.githubusercontent.com/109809/109354059-74ec1f80-783a-11eb-8c39-3b75ac971620.png)
![clang version 10](https://user-images.githubusercontent.com/109809/109354135-8df4d080-783a-11eb-9c7b-57bffa8fe629.png)

The ubuntu apt repository must have been updated and is now grabbing clang version 10.  Luckily I'm able to install multiple versions of clang on my computer so I ran `sudo apt-get install clang-10` and I was able to run my code with it and see a failure locally.  In the [Boost voronoi geometry library](https://www.boost.org/doc/libs/1_75_0/libs/polygon/doc/voronoi_main.htm) of all place.  At least I could reproduce the issue now!

# Suspects

1. It seems like clang could be the problem.  Maybe they broke something in the new version?
2. It's possible that Valgrind was the issue.  It could be that Valgrind is not able to handle clang's latest optimizations.
3. Libboost: I hadn't upgraded but maybe an optimization or improvement in Valgrind surfaced somthing new?

<details> <summary>Spoiler warning</summary> It's number 2. </details>
 
Boost hadn't caused me trouble before and clang is quite well-used so I started by investigating Valgrind.  I searched the web for some bug report that mentioned "clang" and "boost" and "Valgrind" and I wanted it to be pretty new because it's only in clang-10 and after.  I don't know if I found *my* bug but I found [*a* bug](https://bugs.kde.org/show_bug.cgi?id=432801#c12) that sounded similar.  It's also clang only and not gcc and only with optimizations at `-O2`.  And it had some sample code that is pretty short.  Sweet!  I tried it on my computer and confirmed that the Valgrind only reports an error:
* with clang, not gcc
* clang version 10, not 9 nor 8 or 7
* only with optimizations at `-O2`

Here's that code:

```c
// clang -W -Wall -g -O2 Standalone.c && valgrind --track-origins=yes ./a.out

#include <signal.h>

unsigned long myLen(const char* p_str)
{
    unsigned long i=0;
    while (p_str[i]!=0)
        ++i;
    return i;
}

int main()
{
    {
        struct sigaction act;
        if (sigaction(SIGTERM, 0, &act) < 0)
            return 1;
        if (sigaction(SIGTERM, 0, &act) < 0)
            return 1;
    }

    char pattern[] = "12345678 ";
    pattern[0] = '1';
    const unsigned long plen = myLen(pattern);

    size_t hs=0;
    size_t hp=0;
    for (size_t i = 0; i < plen; ++i)
        hp += pattern[i];

    volatile int j=0;
    if (hs==hp)
        j=1;

    return 0;
}
```

Lots of weird stuff about the bug:

1. The error goes away if you don't call sigaction twice or more.
2. The error goes away if you don't set the first byte in `pattern` to a `1`, even though it's already a `1`.
3. The error goes away if you make the `pattern` shorter.
4. The error goes away if you remove the comparison.
5. The error goes away if you replace plen with a constant 9, even though `plen` is always equal to 9.

![WTF?](https://user-images.githubusercontent.com/109809/109370251-cc9b8280-785c-11eb-94c6-4a4aae1e9ecb.png)

# Digging into Valgrind

I started with looking into Valgrind's memcheck.  Valgrind is *enormous* and I did a lot of reading of [code](https://sourceware.org/git/?p=valgrind.git;a=tree;f=memcheck;h=d4028ce4318612ec3e43e91fdf1e849b20976186;hb=HEAD) and eventually the [white paper](https://www.valgrind.org/docs/memcheck2005.pdf), too.

## How Valgrind works

First, I learned how Valgrind memcheck works: For every bit in memory and in registers in the CPU, Valgrind keeps track of whether or not that bit has a "defined" value.  "Defined" means that the bit was set by the program.  For example, if you declare a variable, all the bytes in that variable are marked undefined.  After you assign a value to it, those bits get marked defined.  (Another thing that Valgrind keeps track of, per byte in memory, is whether or not it is "accessible", that is, you didn't walk off the end of your array.  That part isn't pertinent for this bug.)

Valgrind stores all of this in the "shadow" memory.  Knowing that most memory is never accessed and knowing that most bytes are either enitrely defined or entirely undefined, Valgrind can compress the shadow memory instead of storing 9 bits per byte.

Valgrind can also compute the definedness of variables based on that of other variables:

```c
int a = b + c;
```

`a` is defined if both `b` and `c` are defined.  But if either of them is unknown, like maybe you forgot to initialize, then so is `a`.  The definedness equation of that one might be:

```c
defined_a = defined_b && defined_c;
```

Some definedness equations can be trickier!  For example, if you shift a variable 5 bits to the left, the definedness of the bits also shitft to the left and the lower five bits become defined, they are 0s.  Even in the addition example above: If you know the lower bits then the lower bits are defined, even if you don't know the upper ones.

So Valgrind memcheck just needs to disassemble your program, run it one instruction at a time while maintaining the shadow, and everytime you hit a branch, check if the direction of the branch depends on undefined bits.  That would be incredibly slow.  Instead, Valgrind just-in-time replaces your instructions with new instructions that will both do the original work **and** also update the shadow or check it as needed.  Memcheck is the plugin to Valgrind that determines how to instrument the code.  Valgrind is the part that disassembles and re-assembles the code.

# Back to debugging

I used a lot of `printf` in Valgrind to try to figure out what's going on and it was mostly a waste of time.

A good tool was putting the code in [godbolt](https://godbolt.org/z/6s1rTd) and looking at the assembly.  What I did learn was this:  _Some of the modificatons to the code, like using a constant intead of `strlen()` or not modifying the string before processing it caused clang's super-smart optimizer to remove all the code and compute the answer at compile-time._  I guess that was expected.

I also stepped through the code in gdb and found some more interesting stuff:  _If the string is fewer than 9 bytes, it will not enter the highly optimized loop and instead just do some specialized step for strings of length 8 or fewer.  So the problem must be in the loop._

I still didn't know why calling `sigaction` twice or more makes a different.

Eventually I returned to the docs and found the [guide on using gdb with valgrind](https://www.valgrind.org/docs/manual/manual-core-adv.html).  There are a few options there but but the most important one is that you must run it like this:

```sh
valgrind --vgdb=full --vgdb-error=0 --vgdb-shadow-registers=yes --error-exitcode=127 YOUR_PROGRAM
```

* `--vgdb=full` is like turning off optimizations.  Valgrind will run slower but the "definedness" of variables will stay in lock-step with your breakpoints, which is much more useful for reading out the values.
* `--vgdb-error=0` will cause it to halt as soon as starts up, so that you can set a breakpoint.  It'll also print out some instructions on how to use `gdb`.
* `--vgdb-shadow-registers=yes` will let you inspect not just the shadow memory for definedness but also the definedness of the registers.  Crucial.

Then follow the instructions to startup gdb and connect.  Because you're going to be debugging the assembly, it's useful to run `layout asm` in gdb so that you can see what instruction you are on.

# Stepping through the code

The problematic loop which just adds up characters in the string is only for strings with more than 8 characters because it has a complex optimization in it: It uses [x86's SIMD](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) registers, `xmm0` through `xmm4`, to add up all the bytes with less looping.  I assume that the clang people did the measurements and figured out that this complex code is faster than a much shorter and simpler routine that does a lot more branching.

I compiled the code both with and without the second call to `sigaction` and ran it with `vgdb`.  What I discovered was that the second call to `sigaction` was causing more of those xmm SIMD registers to be marked as undefined by valgrind.  Why?  Well, stepping through the `sigaction` code with `vgdb` turned up some small usage of `xmm` registers in `sigaction`.  That usage loaded into xmm registers from memory.  I don't know what it did with it but some of what it read up was junk values.  Though each time the reading into xmm was done from the same place in memory, the first time read it while it was defined and the second time didn't.  I don't know how those values got marked undefined but it was only on the second go.  Maybe due to some write that saved registers to the stack that weren't defined in the first place?

# Aha, a bug!

No, __not a bug__.  Just having garbage in your registers isn't dangerous.  After all, when you allocate an array, it starts with garbage.  Even reading it is okay.  Only when your code depends on it for observable output, for example, a branch.  I also read up on the [x86 calling conventions](https://stackoverflow.com/a/18024743/4454).  Turns out a function *is* allowed to foul your xmm registers.  Programs must not rely on xmm registers being unchanged before and after a function call.  (aka caller-saved)

Anyway, the fouled information in the xmm registers was all zeros anyway because that part of the stack was always zeros in my testing.  Maybe the clang optimization was faulty and Valgrind was right?  One way to find out is to fill that xmm with garbage and see what happens.  [That's what I did with some assembly code.](https://godbolt.org/z/6s1rTd).  And sure enough, clang was giving all sorts of wrong answers!

# I found a bug in clang?!

No, __not a bug__.

I hopped on to the [LLVM discord](https://discord.com/channels/636084430946959380/636732781086638081) to ask if I'd found a bonafide clang bug.  I hadn't.  Once you start mucking with assembly instructions in your code improperly, you are liable to confuse the compiler.  I didn't mark my function as reserving any of those `xmm` registers so it happily inlined my code and I got all sorts of chaos.  Back to the drawing board.

I figured that if I can fill the stack with garbage instead of zeros, then sigaction might read garbage instead of zeros into the `xmm` registers.  Then the routine would continue onward and eventually screw up the answer **without writing assembly code**.  I needed a way to add a bunch of junk on the stack, and it had to be something that the compiler wouldn't optimize away.  Maybe this isn't the shortest way to do it but here's what I came up with:

```c
#define STACK_FILL_SIZE 500

int foo() {
  float stack_filler[STACK_FILL_SIZE];
  for (int i = 0; i < STACK_FILL_SIZE; i++) {
    // Just putting some of my favorite number into the stack for safe keeping.
    stack_filler[i] = cos(((double)(i)) + 0.1); // Never 0 I think.
  }
  float best = 0;
  for (int i = 0; i < STACK_FILL_SIZE; i++) {
    if (stack_filler[i] * 100 > best) {
      best = stack_filler[i];
    }
  }
  return ((int)best);
}
```

When you call it, the stack will stretch itself out to fit those 500 floats.  Then it will retract itself back when it returns from the function.  And after that, sigaction will have a garbage pile from which to read into xmm.  And the computation is complex enough that the optimizer won't be able to get rid of it.  Just in case, I put it in a different c file and linked them together.

![diagram of the stack changes](https://user-images.githubusercontent.com/109809/109371364-be9c3080-7861-11eb-9e81-fff5c83d6c64.png)

_It worked!_  xmm registers were full of seemingly random data when the loop started.  But, the answer was still correct.  Valgrind was still complaining about  computation based on undefined values.

# My final attempt

Now I was back to thinking that Valgrind is the problem.  clang was always giving the right answer so it could hardly be blamed!

I started to read the assembly code that clang generates to optimize the loop.  What it's doing is putting characters from the string into the SIMD registers and doing math on those registers so that some of the additions can happen in parallel.  I stepped through the code with vgdb looking for the spot where an xmm register's "undecidedness" taints the rest of the calculation.  Most of the xmm registers get decidedness during the 40-ish lines of asm except for xmm3.  The undecidedness of that register spreads its undecidedness on all the other registers until the Valgrind complaint.  I narrowed it down to these lines of assembly code:

```
<main+229>     movd   %edx,%xmm2              (1)
<main+233>     punpcklbw %xmm2,%xmm2          (2)
<main+237>     punpcklwd %xmm2,%xmm3          (3)
<main+241>     movzwl 0xa(%rsp,%rsi,1),%edx
<main+246>     movd   %edx,%xmm2              (4)
<main+250>     punpcklbw %xmm2,%xmm2
<main+254>     punpcklwd %xmm2,%xmm2
<main+258>     pxor   %xmm4,%xmm4             (5)
<main+262>     pcmpgtd %xmm3,%xmm4            (6)
<main+266>     psrad  $0x18,%xmm3
```

For the explanation of the code above, we'll use:
* small letters (**abcdqrst**) for known bytes, 
* **X** for undefined garbage bytes, and 
* **0** for defined 0 bytes.  One letter per byte.

* Step (1) is putting a known value into xmm2.  mov**d** is double-word, 4-bytes, so xmm2 is now well-defined as **`abcd0000000000000000`**
* Step (2) is byte-wise interleaving the value in xmm2 with itself.  So `xmm2` is now **`aabbccdd00000000`**.
![image](https://user-images.githubusercontent.com/109809/109371646-1a1aee00-7863-11eb-91f2-f81dfc7ab2cf.png)
* Step (3) is word-wise (16b) interleaving `xmm2` with `xmm3`.  Remember that `xmm3` starts totally undefined so `xmm3` is now **`aaXXbbXXccXXddXX`**.
* Ignore step (4) for now, just notice that it clobbers whatever value was in `xmm2`.
* Step (5) sets `xmm4` to all 0: **`0000000000000000`**

Step (6) is the tricky one.  It's doing a [signed, double-word (32-bit) SIMD comparison](https://www.felixcloutier.com/x86/pcmpgtb:pcmpgtw:pcmpgtd) of `xmm3` and `xmm4` and putting the result as a 0 or -1 into xmm4.  It works like this: Chop up the bits into 32-bit numbers.  If the `xmm4` number is bigger than the `xmm3` number, the `xmm4` position will be filled with ones.  Otherwise, zeros.  So the 4 comparisons look like this (MSB-first):

```
0000 > aaXX ? -1 : 0
0000 > bbXX ? -1 : 0
0000 > ccXX ? -1 : 0
0000 > ddXX ? -1 : 0
```

Or written another way:

```
aaXX < 0 ? -1 : 0
bbXX < 0 ? -1 : 0
ccXX < 0 ? -1 : 0
ddXX < 0 ? -1 : 0
```

That just a check to see if the number on the left is negative.  And to know if a number is negative in 2's-complement, you only need to know the first bit of it.  And we **do** know the first bit.  So the answer is actually fully defined.  clang has no bug!  So why is Valgrind complaining?

It's because Valgrind **doesn't** know about zeros.  Remember, it only knows defined and undefined.  It doesn't keep track of whether a bit is a zero or not.  It *could* but that would potentially make Valgrind run slower and it's already a 5-10x slowdown on your code.  So for Valgrind, it sees 4 comparisons like these:

```
aaXX < qrst ? -1 : 0
```

It's no longer enough to look at just the first bit because what if `aa` is the same as `qr`?  You'll have to look at those `X`s to figure out the answer and so the undefinedness taints the comparison and xmm4 is **not** considered defined.  That undefinedness eventually `valgrind` complains.

# A fix

Remember above in step (4) where the value in `xmm2` is so quickly discarded?  Why use `xmm2` anyway?  Why not just use `xmm3` for the whole thing?  It turns out that the rest of the code does that way:

```
<main+246>     movd   %edx,%xmm2
<main+250>     punpcklbw %xmm2,%xmm2
<main+254>     punpcklwd %xmm2,%xmm2

<main+305>     movd   %edx,%xmm0
<main+309>     punpcklbw %xmm0,%xmm0
<main+313>     punpcklwd %xmm0,%xmm0

<main+322>     movd   %edx,%xmm1
<main+326>     punpcklbw %xmm1,%xmm1
<main+330>     punpcklwd %xmm1,%xmm1
```

How do I convince the compiler to use `xmm3` for the whole time?  I don't.  I just modify the program.  Following [these instructions](https://stackoverflow.com/a/52587878/4454), I wrote this little snippet:

```c++
void asm_test() {
  // what I have
  __asm__ ("movd %edx, %xmm2");
  __asm__ ("punpcklbw %xmm2, %xmm2");
  __asm__ ("punpcklwd %xmm2, %xmm3");

  // what I want
  __asm__ ("movd %edx, %xmm3");
  __asm__ ("punpcklbw %xmm3, %xmm3");
  __asm__ ("punpcklwd %xmm3, %xmm3");
}
```

Compiled it, and then ran `objdump -D` on it and got this:

```
  401697:       66 0f 6e d2             movd   %edx,%xmm2
  40169b:       66 0f 60 d2             punpcklbw %xmm2,%xmm2
  40169f:       66 0f 61 da             punpcklwd %xmm2,%xmm3
  4016a3:       66 0f 6e da             movd   %edx,%xmm3
  4016a7:       66 0f 60 db             punpcklbw %xmm3,%xmm3
  4016ab:       66 0f 61 db             punpcklwd %xmm3,%xmm3
```

Then I opened up my editor (emacs in hexl-mode) and carefully modified the binary.  Where it looked like the former, I made it look like the latter.  I looked at the diff of the objdump of each binary and it looked right.  I ran it through Valgrind and...

```
==11693== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

Great success.

# Next steps

Find out where in LLVM's clang code the generation for that optimization exists and impose on it some requirement that the registers for the `movd`/`punpcklbw`/`punpcklwd` sequence must all be the same `xmm` register.  There could be concerns with that:

* I don't see how there could be any danger in clobbering an xmm register that we're about to clobber anyway.  But maybe there's some reason.
* There might be a _delay_.  A delay is when the CPU is designed such that if you call a certain sequence of instructions, it will put everything on hold.  For example, if you read from memory and then try to use it immediately, maybe your CPU will pause until the read is done.  Better to interleave something else.
* There might be a _hazard_.  A hazard is like a delay except that instead of putting the CPU on pause, you simply get undefined behavior if you don't wait long enough.

I don't expect any of those to be true because there are plenty of examples where other registers are read in sequence without a problem.  But maybe there's a good reason.  Maybe the others were in sequence because the compiler ran out of registers to allocate.  The chip manufacturers will publish these details.

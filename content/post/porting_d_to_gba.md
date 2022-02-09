+++
title = "porting D to the gba"
date = "2022-02-08"
author = "redthing1"
toc = true
+++

# intro

[D](https://dlang.org/) is an [excellent language](https://tour.dlang.org/) and we wanted it ported to GBA.

## why port D?

initially, when we began developing for the GBA, we used [devkitarm](https://devkitpro.org/wiki/Getting_Started) as a build toolchain, and the [makefiles](https://github.com/redthing1/duster/blob/ee741183d9a19e3759a1cc11427d01751a13e2d3/src/DusterGBA/Makefile) I created for [dusk](https://github.com/redthing1/dusk), which in turn were based on tonc makefiles. this worked fine and all, but I really don't love C; it has simplicity going for it, but it's full of ugly syntax, extremely unsafe code, and hidden pitfalls.

D, on the other hand, emphasizes compile time checking and sanity. on a system like the GBA, your code is running on bare metal, and there's really no runtime error checking of any kind. any undefined behavior you do will just get steamrolled over. good luck tracking down what caused a bizarre memory corruption bug. overall, using D would be a huge benefit in preventing entire classes of errors before they happen, just through virtue of strict and smart compile time checking available with D. C++ has modules now? i don't care, I don't like C++.

## why not C++?

C++ is an unbelievably convoluted mess of legacy features and tacked on modern features. unfortunately because they are committed to backwards compatibility, their new features are added with bizarre syntactic conventions. also, `std::me::donot_care_for_the_naming_convention`, terrible template syntax, and slow compile times. i also hate header files and redefinition and forward declarations, and much prefer modules.

## an assessment of the problem

so now I had a good reason and a need to port D to GBA. but doing this blind could prove pretty hard! who knows what needs to be done to get D to run on GBA? on this front, thankfully there was a [project that ported d to 3ds with devkitarm](https://github.com/redthing1/3ds-hello-dlang). if that worked, it was quite likely I could whip up something working for the GBA!

# the process

## starting from what works

of course, i used the obvious starting point: copy pasting from the D on 3ds project!

it looks like that project worked by using [LDC](https://github.com/ldc-developers/ldc), the LLVM compiler for D to generate executable arm7tdmi code. and then using the devkitarm tools to link and package the rom. as for providing basic minimal runtime D functionality, it utilized a *very* minimal replacement runtime library.

so i took that [minimal runtime library](https://github.com/redthing1/duster/commit/2d49dd97d5b1c60bfc1c114f55b416b94708ab17) and added some code that defined conditional compilation for the GBA, for the most part just copying what had been put in for the 3ds.

## forced to use betterc

of course, the functionality required for full D support would include things like garbage collection and threading, which would likely either be extremely tedious or not feasible to support on these bare metal devices. all of this is normally provided by the D runtime library, which we are replacing with a stripped down minimal one. luckily, D [provides](https://dlang.org/blog/2017/08/23/d-as-a-better-c/) a mode called [betterc](https://dlang.org/spec/betterc.html).

with betterc mode, we [lose access](https://dlang.org/spec/betterc.html#consequences) to some language features, most notably:
- garbage collection
- runtime reflection and type metadata
- classes
- threading
- dynamic arrays
- static module constructors/destructors

despite that, we get to keep everything else that's cool about D, which still makes it a large win over C or C++.

## compiling D code

updating the makefile to compile object files from D source was [surprisingly simple](https://github.com/redthing1/duster/commit/bfb0c1c0ea0157351edf6551729dcfe2c4bfaaf9). i did find a [pitfall](https://github.com/redthing1/duster/commit/dd463933e78dce0d3b2b90584ee12a4a9aec7fd1) where I had to explicitly specify strict memory alignment, because the GBA force-aligns misaligned memory accesses.

i also ported all of dusk from C to D manually, and left some parts of the contrib C files, like gbfs and gbamap as C. D has great support for linking to C object files, so that doesn't present us any issues as long as we're specifying `extern (C)`.

## porting tonc

porting tonc headers to D was [fairly straightforward](https://github.com/redthing1/duster/commits/3f13ad5fe4e0affe06284da7145da99a2dd4b608/src/libd/tonc). nothing out of the ordinary really. i just used [dstep](https://github.com/jacob-carlborg/dstep) to automate 90% of the conversion, then did a little manual fixing. i also had to add back in inline functions, as you can't link to tonc functions that are inline.

at this point, our `libd` tonc is pretty much a drop in replacement for C tonc. note that it's just headers; we still link against the very same libtonc binary! so there's no need to install anything new, our support for D compilation is solely within our own source tree!

## creating a minimal D library

at this point, we were ready to separate our D library into its own thing. i [moved](https://github.com/redthing1/gba_dlang/commit/51b5787f77cbd9843e1dfd60a1af932cdbfc3e83) everything into [gba_dlang](https://github.com/redthing1/gba_dlang). this way, using my ported D support for the GBA was as simple as adding a submodule and updating your makefile. also, this new project comes complete with a [working example](https://github.com/redthing1/gba_dlang/tree/main/demo). additionally, [libd](https://github.com/redthing1/gba_dlang/tree/main/libd) contains our D standard library for GBA, known simply as `libd`!

# how it all works

i want to quickly give an overview of how the whole thing functions. the main purpose of this is to serve as a point for future reference. porting D to any other weird system? this overview might be the framework to follow.

## distilled summary

- `ldc` used to compile D sources to `.o` files.
- `libd` provides a minimal D standard library that is added to default include path, to provide core functionality and access to the C standard library via `core.stdc.*` modules.
- `.elf` is created from linking D and C compiled object files with devkitarm linker `gcc`, which is the standard C linker.
- `.GBA` is created as usual with devkitarm toolchain.

all of this can be [concretely seen](https://github.com/redthing1/gba_dlang/blob/main/demo/Makefile) in the demo makefile.

# conclusion

we now have ability to program in D for GBA!

of course, one of my projects i embarked on right after getting gba_dlang to a functioning state was porting the entire duster codebase from C to D. today, [duster](https://github.com/redthing1/duster) is written entirely in D, save for a couple files.

with this work, anyone can now program in D for the GBA! need a starting point? a good way is to just clone duster and strip it down to the bare minimum, then build your own project using that skeleton. that should provide a fully functioning project for your own GBA project in dlang.

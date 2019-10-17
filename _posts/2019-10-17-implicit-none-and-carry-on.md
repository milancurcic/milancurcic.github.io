---
layout: post
title: "Implicit none and carry on"
author: "Milan Curcic"
---

Fortran has an interesting historic feature called implicit typing: 
Undeclared variables whose name begins with letters i, j, k, l, m, or n 
are implied to be integers, and real otherwise. 
For example, this is a valid Fortran program that prints a square of first five positive integers:

```fortran
do i = 1, 5    ! i is implied integer
  x = i**2     ! x is implied real
  print *, x
end do
end
```

While there aren’t any variable declarations here, both `i` and `x` have a well defined type. 
The output?

```
   1.00000000    
   4.00000000    
   9.00000000    
   16.0000000    
   25.0000000
```

This "feature" originated in the early days before FORTRAN 66 introduced 
the way to explicitly declare variables and their data type. 
FORTRAN 77 introduced the `IMPLICIT` statement to modify the implicit typing rules, 
although this wasn't enough to prevent this joke from coming into existence:

> In Fortran, GOD is REAL (unless declared INTEGER).

Implicit typing does add mental burden to the programmer, 
and can lead to surprising and unexpected behavior, 
especially in numerical code that relies on type coercion (mixed-mode arithmetic). 
Fortran 90 brought us the now ubiquitous `implicit none`, 
which enforces explicit declaration of all variables. 
Its adoption was so widespread that today you have to dig deep (and know where to look) 
to find legacy Fortran code without this statement. 
Typing `implicit none` at the start of any program is one of the first things a novice Fortran programmer learns today.

There has been discontent expressed recently on 
[comp.lang.fortran](https://groups.google.com/forum/#!topic/comp.lang.fortran/m6qz7hC4a7M) 
about implicit typing, and requests to change the default behavior in the 
Fortran Standard — the reference specification of the language — to enforce explicit typing by default. 
In general, such requests aren't well received by neither the Standard Committee members, nor by compiler developers. 
My opinion was unambiguous, if a bit blunt:

> I'm fine with typing implicit none for the next 50 years, and perhaps longer. 
> I prefer compiler developers focusing on more productive tasks.

The only benefit of such change would mean typing 13 characters less. 
It's been argued that needing to type `implicit none` deters newcomers to the language and hinders its wider adoption. 
To date I haven't seen evidence of this.

Don't get me wrong — I wholeheartedly agree that implicit typing is an anti-pattern and I never ever recommend it. 
However, I disagree that changing the language to enforce explicit typing by default is a useful solution, not in the Fortran 202x era. 
Especially considering that compilers already have a flag to enforce this behavior. 
For example with gfortran and the program above:

```
$ gfortran implicit.f90 -fimplicit-none
implicit.f90:1:4:do i = 1, 10
    1
Error: Symbol ‘i’ at (1) has no IMPLICIT type
implicit.f90:2:3:x = i**2
   1
Error: Symbol ‘x’ at (1) has no IMPLICIT type
```

With Intel Fortran compiler, the flag is `-implicitnone`. 
Easy, if that’s your cup of tea. 
I never use these myself. 
If you want it, you got it.

The downside to such change in the Standard is that it would break backwards compatibility, one of Fortran’s key strengths. 
If the default behavior changed, all of a sudden, with a single compiler update, 
a whole population of previously standard conforming code would stop building. 
Maintainers of such legacy systems would need to either update their code — 
by explicitly declaring all previously implicitly typed variables — 
or use a compiler option to allow it as an exception. 
In practice, the change would happen on both application and compiler ends, incurring development and maintenance costs.

Remember that _Fortran is a mature language_. 
The goal is not to move fast and break things. 
It's to change the language as little as possible while incrementally improving it.

How about new code? 
If you always use `implicit none` in your Fortran code, as I do, you only need to type it in programs and modules. 
As long as you organize your functions and subroutines in modules (or, in programs under the contains statement), 
you’re really only typing `implicit none` every time you write a new program or a new module.

Is this such a big deal? 
Let's see: It takes me about 2 seconds to type `implicit none`, new lines including. 
I only type it in new programs and modules. 
If you define procedures in modules exclusively, they inherit the its explicit typing behavior. 
I write a new module or a program once every week, two at most. 
Let's be generous and say I type `implicit none` 100 times per year. 
_Over the next 50 years, I will have typed it about 5000 times, spending a total of about 3 hours_. 
There's no cognitive burden here, it's all muscle memory.

If you still think typing `implicit none` is a nuissance, a shift in perspective may help. 
Typing it can be a beautiful and meditative ritual. 
Like when you wake up in the morning and make that first stretch. 
Type `implicit none` to set the intention. 
The intention to write simple and correct code. 
Code that is fun to write, a joy to read, and most important, code that works. 
Take a slow, deep breath in...

![]({{ "/assets/implicit-none.png" | absolute_url }})

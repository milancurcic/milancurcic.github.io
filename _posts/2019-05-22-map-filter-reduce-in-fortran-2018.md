---
layout: post
title: "Map, Filter, Reduce in Fortran 2018"
author: "Milan Curcic"
---

![]({{ "/assets/nik-shuliahin-map.jpg" | absolute_url }})

_Also on [Medium](https://medium.com/modern-fortran/map-filter-reduce-in-fortran-2018-e40b93668ed9)_.

While not a purely functional language, 
Fortran allows the programmer to express themselves functionally. 
John Backus, the original creator of FORTRAN at IBM in 1956, 
argued for functional programming in his [1977 Turing award lecture](https://dl.acm.org/citation.cfm?id=359579). 
Map, filter, and reduce are the core tools of a functional programmer — 
they allow you to solve problems by chaining recursive functions 
instead of piling do-loops and if-blocks one on top of another. 
Map applies a function element-wise to an arbitrary array. 
Filter applies a function to an array and returns only those elements 
that meet the criteria defined in the function. 
Reduce, often also called _fold_, applies a reduction operation to 
an array and returns a scalar result.

Fortran 2018 now offers (almost) out-of-the-box support for map, 
filter, and reduce patterns. This article explains how.

## Map

Let’s start with map. This higher-order function applies a function `f` for 
each element in array `x` and returns the resulting array. 
Given function `f` and array `x`, the typical syntax is `map(f, x)`. 
This expression returns an array of same size as `x`. 
On first look, Fortran doesn’t have anything like map in its standard library. 
However, Fortran has had similar functionality built into the language 
since the Fortran 90 standard — elemental procedures.

Elemental procedures allow you to define a procedure that can operate 
both on a scalars and arrays at the same time. A minimal, though not terribly useful example, is:

```fortran
pure elemental real function square(x)
  real, intent(in) :: x
  square = x**2
end function square
```

`square(2.)` then returns `4.`. However, the elemental attribute makes this function surprisingly powerful. 
Without any modifications to the code, you can apply square to an array — `square([2., 3., 4.])` returns `[4., 9., 16.]`. 
Further, you can apply square to an array of any rank. Fortran supports up to 15-dimensional arrays. 
I’ve never used more than 5 dimensions myself but I’m sure there are application domains that make good use many-dimensional arrays.

However, `elemental` originally came with a restriction — you couldn’t make a procedure both elemental _and_ recursive. 
If you wanted to apply a recursive function to an array, you had to resort to a do-loop. 
This is why I made `map` available as part of [functional-fortran](https://github.com/wavebitscientific/functional-fortran). 
Say you have a recursive function, an ubiquitous textbook example being the Fibonacci function:

```fortran
pure recursive integer function fibonacci(n) result(fib)
  integer, intent(in) :: n
  if (n == 0) then
    fib = 0
  else if (n == 1) then
    fib = 1
  else
    fib = fibonacci(n-1) + fibonacci(n-2)
  end if
end function fibonacci
```

Prior to Fortran 2018, you had to specify the attribute `recursive` if the function was to be used recursively. 
However, if you tried to make this function both `recursive` and `elemental`, 
you’d get scolded by the compiler soon enough. Here’s the output from gfortran-8.2.1:

```
$ gfortran map.f90 
map.f90:7:27:

pure elemental recursive integer function fibonacci(n) result(fib)
                           1
Error: ELEMENTAL attribute conflicts with RECURSIVE attribute at (1)
```

In his human-readable rendering of the updates to the Fortran standard, John Reid writes:

> The restriction against elemental recursion was intended to make elemental procedures easier to implement and optimise, 
> but recursion has become normal so it is not needed.

This means that Fortran 2018 finally drops the restriction for recursive procedures also being elemental. 
However, we can’t compile our programs with the ISO standard, so until the compiler developers implement these features, 
we still have to resort to a home-cooked `map`:

```fortran
pure function map(f, x)
  procedure(f_int) :: f ! Mapping function
  integer, intent(in) :: x(:) ! Input array
  integer :: map(size(x)), i
  map = [(f(x(i)), i = 1, size(x))]
end function map
```

That’s it, the whole function. We loop over all elements of `x`, and apply function `f` to each, 
all wrapped in an array comprehension. The result array `map` is a so-called automatic array — 
it assumes the size of `x` and will by default be allocated on the stack. 
For the above to work though, we still need to define an interface for `procedure(f)`:

```fortran
interface
  pure integer function f_int(x)
    integer, intent(in) :: x
  end function f_int
end interface
```

Now that we have the recursive function we want to apply (`fibonacci`) and `map`, we can use it as advertised. 
For example, `map(fibonacci, [5, 10, 20])` yields `[5, 55, 6765]` as the result. 
To get all Fibonacci numbers, say, from 1 to 30, you’d do `map(fibonacci, [(n, n = 1, 30)])`.

Once you map a function to all elements of an array, what else would you do but filter the results?

## Filter

Filter asks for a function that returns a logical (Fortran word for a Boolean) 
True or False given an input value. Common example of a function used to 
demonstrate filtering numbers are `even` or `odd`:

```fortran
pure logical function even(n)
  integer, intent(in) :: n
  if (mod(n, 2) == 0) then
    even = .true.
  else
    even = .false.
  end if
end function even
```

Let’s define a higher-order filter function to get even numbers out of an array of integers 
so that we can just do `filter(even, x)`. Like map, filter can be constructed with existing Fortran building blocks.

Since Fortran 95, we’ve had the function pack in the standard library. 
[GNU Fortran documentation for pack](https://gcc.gnu.org/onlinedocs/gfortran/PACK.html) reads:

> **PACK — Pack an array into an array of rank one**
> 
> Description: Stores the elements of ARRAY in an array of rank one.

It sounds like this is meant to unroll (or flatten, if you come from the numpy world) 
a multi-dimensional array into a one-dimensional array. Keep reading though:

> The beginning of the resulting array is made up of elements whose MASK equals TRUE. 
> Afterwards, positions are filled with elements taken from VECTOR.

OK, there are a few more elements to the syntax of `pack` here, namely the arguments `mask` and optional `vector`. 
To understand what these do, we need to look at the complete syntax for `pack`:

```fortran
result = pack(array, mask[, vector])
```

The square brackets here indicate optional syntax. The docs go on to describe the arguments:

> `ARRAY` Shall be an array of any type.
>
> `MASK` Shall be an array of type `LOGICAL` and of the same size as `ARRAY`. Alternatively, it may be a `LOGICAL` scalar.
> 
> `VECTOR` (Optional) shall be an array of the same type as `ARRAY` and of rank one. 
> If present, the number of elements in `VECTOR` shall be equal to or greater than the number of true elements in `MASK`. 
> If `MASK` is scalar, the number of elements in `VECTOR` shall be equal to or greater than the number of elements in `ARRAY`.

It turns out that `pack` does the very same thing as `filter`, except that instead of the filtering function, 
you pass a logical array that says which elements of the original input array to return. Great! 
This means that the filter could be defined as a syntactic-sugar kind of wrapper around `pack`:

```fortran
pure function filter(f, x)
  procedure(f_int_logical) :: f ! An int -> logical function
  integer, intent(in) :: x(:) ! Input array
  integer, allocatable :: filter(:)
  integer :: i
  filter = pack(x, [(f(x(i)), i = 1, size(x))])
end function filter
```

We declare `filter(:)` as an allocatable array since we don’t know its size ahead of time. 
However, we don’t need to allocate it explicitly, as we can use automatic allocation on assignment, 
a neat feature of Fortran 2003. Note that as an allocatable array, the result will be allocated on the heap, 
which is likely to cause a performance hit relative to automatic arrays that typically go on the stack.

As before, to use `procedure(f_int_logical)` as an input argument, 
you need to define its interface first (just the interface, no body!):

```fortran
interface
  pure logical function f_int_logical(x)
    integer, intent(in) :: x
  end function f_int_logical
end interface
```

Having covered `map` and `filter`, we can now combine them to map a recursive function over all elements of an array, 
then filter the results by applying the filtering function:

```fortran
result = filter(even, map(fibonacci, [(n, n = 1, 30)]))
```

This snippet takes an integer array `x`, computes a Fibonacci number for each element, 
and then filters out only even numbers out of the lot. 
It yields `[2, 8, 34, 144, 610, 2584, 10946, 46368, 196418, 832040]` as the result.

## Reduce

We’ve mapped a function to the input array and filtered the results. 
The last step is to apply a reduction operation on the array to get to a scalar result. 
Reduction (or folding) recursively applies a binary operation to all elements of the array, 
in or out of order, until it exhausts the array and reaches the final scalar result. 
Common examples of Fortran functions that perform reduction on arrays are `sum`, `product`, `minval`, or `maxval`.

Admittedly, reduction is tad more difficult to get your head around compared to map or filter. 
Guido van Rossum [argued against reduce](https://www.artima.com/weblogs/viewpost.jsp?thread=98196) 
in Python because it’s difficult to reason about reduction 
unless the operation is an extremely simple one, such as addition and multiplication. 
In those cases, he argued for including `sum` and `product` functions in the standard library, 
whereas any reduction more complex than that would be more clearly expressed with list comprehensions. 
Should you reduce or not in your code? Totally up to you — try it out and see if you like it. 
Let’s see how we can use it in Fortran today.

### The Fortran 2018 reduce

Fortunately for functional Fortranners, Fortran 2018 brings a new reduction intrinsic 
(a Fortran word for a built-in, or a standard library function), `reduce`. 
There are two forms to this function (Reid, 2018):

```fortran
reduce(array, operation[, mask, identity, ordered])
reduce(array, operation, dim[, mask, identity, ordered])
```

The input array can be of any type and rank. 
The second argument is a pure function with two arguments and a result of same type as array. 
If 2nd form is used, `dim` indicates the one dimension along which to perform the reduction, 
and the result is an array of rank reduced by one relative to the input array. 
`mask` a logical array which will filter the input array, much like in the built-in function `pack`. 
`identity` is a fall-back value that the result will take if the input sequence is empty. 
Finally, if `ordered` is `.true.`, the reduction is applied left-to-right. 
Otherwise, the will compiler will assume that the operation is commutative and will 
evaluate the reduction in the most optimal way it can find.

The built-in reduce function looks quite useful and flexible, however, 
it’s a recent addition to the language, and as of May 2019, 
stable release of gfortran (9.1) doesn’t support it. 
Until it does, we need to roll our own implementation. 
Note that the parallel reduction `co_reduce` is supported 
and ready to use with gfortran and [OpenCoarrays](http://www.opencoarrays.org), 
and works well in practice.

### Implementing your own reduce function

Let’s implement a custom `reduce` high-order function that will allow us to do the following:

```fortran
result = reduce(add, filter(even, map(fibonacci, x)), 0)
```

where `add` is a function to add two integer scalars. 
First argument is the function that we’ll use as a reduction operation, 
second argument is the input array, 
and third argument is the starting value (I’ll get to this in a bit). 
This single line will compute a sum of all even Fibonacci numbers from 1 to 30.

`reduce` applies a binary function or operator to elements of an array, recursively. 
Here’s the code for a left-to-right reduction (a so-called _left-fold_):

```fortran
pure recursive integer function reduce(f, x, start) result(res)
  procedure(f2_int) :: f
  integer, intent(in) :: x(:), start
  if (size(x) < 1) then
    res = start
  else
    res = reduce(f, x(2:), f(start, x(1)))
  end if
end function reduce
```

where `f2_int` is an interface of a function that takes two integers and returns an integer result:

```fortran
pure integer function f2_int(x, y)
  integer, intent(in) :: x, y
end function f2_int
```

and `start` is the starting value to use when applying the reduction to the first element of `x`, 
and also the value of the result should the input array be empty. 
Intrinsic function `sum` could then be written as `reduce(add, x, 0)`, 
while the product function could be written as `reduce(mul, x, 1)`. 
Note that the `start` argument can be made optional — 
Python’s `functools.reduce()` does so, for example.

Alternatively, you can also apply reduction to the array elements right-to-left, also called a _right-fold_:

```fortran
pure recursive integer function reduce_right(f, x, start) result(res)
  procedure(f2_int) :: f
  integer, intent(in) :: x(:), start
  if (size(x) < 1) then
    res = start
  else
    res = f(x(1), reduce_right(f, x(2:), start))
  end if
end function reduce_right
```

Left- versus right-fold defines the associativity of the operation `f`. 
For example, left-fold will evaluate the sum of `[1, 2, 3, 4, 5]` as `(((1 + 2) + 3) + 4) + 5`, 
whereas the right-fold will evaluate it as `1 + (2 + (3 + (4 + 5)))`. 
While the order is irrelevant in this particular case, 
it will matter for reductions of floating-point arrays, or if applying a non-commutative function.

If you put all the pieces together, you can now write:

```fortran
integer, allocatable :: x(:)
x = [(n, n = 1, 30)]
print *, reduce(add, filter(even, map(fibonacci, x)), 0)
```

and the result is 1089154.

For another example of a common reduction operation, 
a Fortran programmer often uses intrinsics `any` and `all` to test 
if any or all values of a Boolean array are True, respectively. 
If you think about these in the context of `reduce`, 
these could be written as `reduce(or, x, .false.)` and `reduce(and, x, .true.)`, respectively. 
If you get creative, you can do other cool stuff too, such as reverse or sort arrays.

Finally, should you use it in production? 
In his paper on universality of recursive reduction, Graham Hutton (1999) wrote:

> Programs written using fold can be less readable than programs written using explicit recursion, 
> but can be constructed in a systematic manner, and are better suited to transformation and proof.

Thus, while using recursive reduction likely leads to more expressive and cleaner code 
that is a composition of small building blocks, 
it can also be less readable and more difficult to reason about, 
merely due to it being recursive and not iterative. 
Try to recurse in your head — it’s not easy!

Functional patterns, and recursion in general, may come with a performance hit in Fortran. 
This is simply due to the fact that these patterns haven’t been used much in legacy Fortran historically, 
so they haven’t ranked highly on the priority list for compiler developers. The usual recommendations hold:

* Use the patterns that are most intuitive to you. 
There’s no point in writing smart code if you won’t understand what it does when you look at it a year later.
* Profile your code before using functional patterns in production. 
They may be less computationally efficient than the imperative implementation, 
and this may vary wildly between compiler vendors.

## Summary

Although it’s not immediately obvious, Fortran 2018 now provides (almost) 
complete out-of-the-box support for map-filter-reduce patterns:

* **Map**: Use elemental functions that are defined on scalars, 
but can operate element-wise on arrays of any rank. 
Prior to Fortran 2018, there was a restriction on elemental functions being recursive. 
This restriction is now lifted, and I can’t think of a good reason to cook up your own implementation of `map`. 
If you write your functions as `elemental`, what you’d write in some other language as `map(f, x)` 
in Fortran 2018 is just `f(x)`. Beauty!
* **Filter**: Use the built-in function `pack`, together with an array comprehension, 
e.g. `pack(x, [(f(x(i)), i = 1, size(x))])`, or use the `filter` function from the 
[functional-fortran](https://github.com/wavebitscientific/functional-fortran) 
library if array comprehension is too verbose for you.
* **Reduce**: Fortran 2018 brings `reduce` (serial) and `co_reduce` (parallel) to its standard library. 
Until your compiler supports `reduce`, use a custom implementation of `reduce` from this article, 
or use one of the `fold` implementations from [functional-fortran](https://github.com/wavebitscientific/functional-fortran).

## References

* [Backus, J. (1978): Can programming be liberated from the Von Neumann Style? A functional style and its algebra of programs. Commun. ACM, 21 (8), 613–641.](https://dl.acm.org/citation.cfm?id=359579)
* [Hutton, G. (1999): A tutorial on the universality and expressiveness of fold. J. Functional Programming, 9 (4), 355–372.](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)
* [Reid, J. (2018): The new features of Fortran 2018. ISO/IEC JTC1/SC22/WG5 N2161.](https://isotc.iso.org/livelink/livelink?func=ll&objId=19867230&objAction=Open)
* [Van Rossum, G. (2005): The fate of reduce() in Python 3000, All Things Pythonic.](https://www.artima.com/weblogs/viewpost.jsp?thread=98196)

---

_Cover photo by [Nik Shuliahin](https://unsplash.com/@tjump)._

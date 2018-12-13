---
layout: post
title: "Analyzing stock price time series with modern Fortran, Part 2"
author: "Milan Curcic"
---
<link rel="canonical" href="https://freecontent.manning.com/analyzing-stock-price-time-series-with-fortran-arrays-part-2">

![]({{ "/assets/rick-tap-wallstreet.jpg" | absolute_url }})

This article is an excerpt from my book on [Modern Fortran](http://bit.ly/2EeLiT6),
and originally published [here](https://freecontent.manning.com/analyzing-stock-price-time-series-with-fortran-arrays-part-2).
You can also read it on [Medium](https://medium.com/modern-fortran/analyzing-stock-price-time-series-with-fortran-arrays-part-2-50721ed40efc).

## Allocating, indexing, and slicing arrays for stock price analysis

In [Part 1](https://milancurcic.com/2018/11/06/analyzing-stock-price-time-series-with-modern-fortran-part1.html), 
I covered the basics of declaring and initializing dynamic Fortran arrays. 
I also introduced the array constructors, which provide an easy way to allocate 
and initialize an array in a single line of code. If you haven’t yet read Part 1, 
I suggest that you go [back and read it](https://milancurcic.com/2018/11/06/analyzing-stock-price-time-series-with-modern-fortran-part1.html) 
before moving on. 
In Part 2, we’ll learn how allocation and deallocation of Fortran arrays works in depth, 
allowing you to allocate arrays with arbitrary ranges (start and end indices). 
You’ll also learn how to slice the arrays in any direction and with custom stride to 
subset exactly the elements that you need. Finally, in this part we’ll apply this knowledge 
to implementing a solution to our first challenge — finding the best and worst performing stocks.

Jump to:

* [Allocating arrays of certain size or range](#allocating-arrays-of-certain-size-or-range)
  - [Allocating an array from another array](#allocating-an-array-from-another-array)
  - [Automatic allocation on assignment](#automatic-allocation-on-assignment)
  - [Cleaning up after use](#cleaning-up-after-use)
  - [Checking for allocation status](#checking-for-allocation-status)
  - [Catching allocation and deallocation errors](#catching-allocation-and-deallocation-errors)
  - [Implementing the CSV reader subroutine](#implementing-the-csv-reader-subroutine)
* [Indexing and slicing arrays](#indexing-and-slicing-arrays)
  - [Intermezzo: Reversing an array](#intermezzo-reversing-an-array)
* [Summary](#summary)

## Allocating arrays of certain size or range

In Part 1, we’ve learned how to declare and initialize dynamic arrays. 
However, what if we need to assign values to individual array elements, one by one, in a loop? 
This will be the case as we load the data from CSV files into arrays — we’ll iterate over records in files, 
and assign values to array elements one at a time. However, we don’t really have a way to initialize 
the data arrays like we did before with `stock_symbols`. Note that implicitly allocating by assigning an 
empty array `[integer ::]` or `[real ::]` won’t work here because we may need to index elements of an array 
in some other order than just appending values. 
This calls for a more explicit mechanism to allocate the array without assigning known values to it:

```fortran
real, allocatable :: a(:) ! declare a dynamic array a
integer :: im = 5 !
allocate(a(im)) ! allocate memory for array a with im elements
```

The above statement tells the program to reserve memory for the array `a` of size `im`, in this case 5. 
When invoked like this, `a` will by default have the lower bound of 1, and upper bound of `im`. 
The lower bound of 1 is the default, similar to Julia, R, or MATLAB. 
This is unlike C, C++, Python, or JavaScript, where array or list indices begin with 0.

However, Fortran doesn’t impose a constraint to the start index being 1, unlike Python, 
where the first index is always 0. You can specify the lower and upper bounds in the allocation statement:

```fortran
integer :: is = -5, ie = 10
allocate(a(is:ie)) ! Allocate a with range from is to ie inclusive 
```

Notice that I know used a colon (`:`) between `is` and `ie` to specify the range. 
This range is inclusive (unlike in Python!), so the size of `a` is now `ie - is + 1`, in this case 16. 
You can use intrinsic functions `lbound` and `ubound` to get the lower and upper bound, respectively, of any array.

### Allocating an array from another array

It’s also possible to dynamically allocate an array based on the size of another array. 
The `allocate` statement accepts two optional arguments:

* `mold` — a variable or an expression that has the same type as the object being allocated.
* `source` — equivalent to `mold`, except that the values of source are used to initialize the object being allocated.

For example, allocating `a` from `b` using `mold` will reserve the space in memory for `a`, 
but will not initialize its elements:

```fortran
real, allocatable :: a(:), b(:)
allocate(b(10:20))
allocate(a, mold=b) ! allocate a with same range and size as b
a = 0 
```

However, if we allocate `a` from `b` using source, it will be allocated and initialized with values of `b`:

```fortran
real, allocatable :: a(:), b(:)
b = [1.0, 2.0, 3.0]
allocate(a, source=b) ! allocate and initialize a from b
```

> **Style Tip**: No matter how you choose to allocate your arrays, 
> always initialize them immediately after allocation. 
> This will minimize the chance of accidentally using uninitialized arrays in expressions. 
> While Fortran will allow you to do this, you’ll likely end up with gibberish results.

You may have noticed that when describing the array constructors earlier, 
I initialized the arrays without explicitly allocating them with an `allocate` statement. 
Why did I do that? You may rightfully ask, do I need to explicitly allocate arrays or not? 
Since Fortran 2003, there has been a convenient feature of the language called _allocation on assignment_.

### Automatic allocation on assignment

If you assign an array to an allocatable array variable, 
the target array variable is automatically allocated with 
the correct size to match the array on the right-hand side. 
The array variable can be already allocated or not. 
If it is, it will be re-allocated if its current size differs 
from that of the source array. 
A great example to play with this is appending elements to an array on the fly:

```fortran
integer, allocatable :: a(:)

a = [integer ::] ! create an empty array []
a = [a, 1] ! append 1 to a, now [1]
a = [a, 2] ! append 2 to a, now [1, 2]
a = [a, 2 * a] ! [1, 2, 2, 4]
```

This feature is particularly useful when trying to assign an array that is a result of a function, 
and whose size is not known ahead of time. There is another important difference between explicit 
allocation with the `allocate` statement and allocation on assignment. 
The former will trigger a runtime error if issued twice, that is, 
if you issue an `allocate` statement on an object that is already allocated. 
On the other side, the latter will gracefully re-allocate the array if already allocated. 
To be able to effectively reuse dynamic arrays, Fortran gives as a counterpart to the 
`allocate` statement which allows us to explicitly clear the object from memory.

### Cleaning up after use

When we’re done working with the array, we can clean it from memory like this:

```fortran
deallocate(a) ! clear a from memory
```

After issuing `deallocate`, array `a` must be allocated again before using it 
in the right-hand side of expressions. We’ll apply this mechanism to reuse arrays between different stocks.

> **Automatic deallocation**: An allocatable array is automatically deallocated 
> when it goes out of scope. For example, if you declare and allocate an array 
> inside of a function or subroutine, it will be deallocated on return.

Much like it’s an error to allocate an object that is already allocated, 
it’s also an error to deallocate an object that is not allocated! 
In the next section I explain how you can check the allocation status 
so that you never erroneously allocate or deallocate an object twice. 
Otherwise, there is no restriction with regard to whether the array has been initialized or not. 
You are free to deallocate an uninitialized array, for example if you learn that the array 
is not of the expected size, or similar.

> **Style Tip**: Deallocate all allocatable variables when you are done working with them.

I illustrate a typical life cycle of a dynamic array in the diagram below.

![]({{ "/assets/allocation_diagram.png" | absolute_url }})

We first declare the array as `allocatable`. 
At this point, the array is not yet allocated in memory and its size is unspecified. 
When ready, we issue the `allocate` statement to reserve a chunk of memory to hold this array. 
This is also where we decide the size of the array or the start and end indices (in this example, 3 and 8). 
If not allocating from another source array, the values will be uninitialized. 
We thus need to initialize the array before doing anything else with it. 
Finally, once we’re done working with the array, we issue the `deallocate` statement 
to release the memory that was holding the array. 
The status of the array is now back to unallocated, and is available for allocation. 
A dynamic array can be reused like this any number of times, even with different sizes or start and end indices. 
This is exactly what we’ll do in our stock price analysis app. 
For each stock, we’ll allocate the arrays, use them to load the data from files, work on them, 
and then deallocate them before passing them on to the next stock.

### Checking for allocation status

It will at times be useful to know the allocation status of a variable, that is, 
whether it’s currently allocated or not. To do this, we can use the intrinsic `allocated` function:

```fortran
real, allocatable :: a(:)
print *, allocated(a) ! will print “F”
allocate(a(10))
print *, allocated(a) ! will print “T”
deallocate(a)
print *, allocated(a) ! will print “F” 
```

Trying to allocate an already allocated variable, or to deallocate a variable that is not allocated, 
will trigger a run-time error.

> **Style Tip**: Always check for allocation status before explicitly allocating or deallocating a variable.

### Catching allocation and deallocation errors

Your allocations and deallocations will occasionally fail. 
This can happen if you try to allocate more memory than available, 
if you try to allocate an object that is already allocated, 
or free an object that has been freed. When this happens, the program will abort. 
However, the `allocate` statement also comes with built-in exception handling if 
you want finer control over what happens when the allocation fails:

```fortran
allocate(u(im), stat=stat, errmsg=err)
```

where `stat` and `errmsg` are optional arguments:

* `stat` — an integer that indicates the status of the allocate statement. 
`stat` will be zero if allocation was successful, otherwise, it will be a non-zero positive number.
* `errmsg` — a character string that contains the error message if an error occurred 
(such as stat being non-zero), and is undefined otherwise.

By using the built-in exception handling, you get the opportunity to decide 
how should the program proceed if the allocation fails. 
For example, if there is not enough memory to allocate a large array, 
perhaps we can split the work into smaller chunks. 
Even if you want the program to stop on allocation failure, 
this approach lets you handle things gracefully and print a meaningful error message.

> **Style Tip**: If you want control over what happens if (de)allocation fails, 
> use `stat` and `errmsg` in your `allocate` and `deallocate` statements to catch any 
> errors that may come up. Of course, you’ll still need to tell the program what to 
> do if an error occurs, for example, stop the program with a custom message, 
> print a warning message and continue running, or try to recover in some other way.

I think we should use the built-in exception handling in our stock analysis app. 
However, we’re going to need this for several arrays. This seems suitable to 
implement once in a subroutine, and then reuse it as needed. Explicitly allocating 
and deallocating arrays can be quite tedious. This is especially true if you decide 
to make use of the built-in exception handling. If you’re working with many different 
arrays at a time, this can quickly build up to a lot of boilerplate code.

Let’s implement simple subroutines `alloc` and `free` that allocate and deallocate, 
respectively, an input array, and also handle exceptions. Both subroutines should 
use the `stat` and `errmsg` arguments to catch and report any errors if they occur. 
Once implemented you should be able to allocate and free your arrays like this:

```fortran
call alloc(a, 5)
 ! do work with a
call free(a)
```

Let’s start with the allocator subroutine `alloc`. 
For the key functionality to work our subroutine needs to:

1. Check if input array is already allocated, and if yes, deallocate it before proceeding.
2. Allocate the array with input size `n`.
3. If an exception occurs during allocation, abort the program and print the error message to screen.

Here’s the implementation:

```fortran
subroutine alloc(a, n)
  real, allocatable, intent(in out) :: a(:)
  integer, intent(in) :: n
  integer :: stat
  character(len=100) :: errmsg
  if (allocated(a)) call free(a)
  allocate(a(n), stat=stat, errmsg=errmsg)
  if (stat > 0) error stop errmsg
end subroutine alloc
```

Now, let’s take a look at the implementation of the `free` subroutine:

```fortran
subroutine free(a)
  real, allocatable, intent(in out) :: a(:)
  integer :: stat
  character(len=100) :: errmsg
  if (.not. allocated(a)) return
  deallocate(a, stat=stat, errmsg=errmsg)
  if (stat > 0) error stop errmsg
end subroutine free
```

The code is very similar to `alloc` except that here, 
at the start of the executable section of the code, 
we check if a is already allocated. 
If not, our job here is done and we can return immediately.

These subroutines are also part of the stock-prices repository. 
You can find them in `stock_prices/src/mod_alloc.f90`, and are used by the CSV reader 
in `stock_prices/src/mod_io.f90`. We’ll use these convenience subroutines to greatly 
reduce the boilerplate in the `read_stock` subroutine.

### Implementing the CSV reader subroutine

Having covered the detailed mechanics of allocating and deallocating arrays 
including the built-in exception handling, 
we finally arrive to implementing the CSV file reader subroutine:

```fortran
subroutine read_stock(filename, time, open, high,&
  low, close, adjclose, volume)
  ...
  integer :: fileunit
  integer :: n, nm
 
  nm = num_records(filename) — 1
 
  if (allocated(time)) deallocate(time)
  allocate(character(len=10) :: time(nm))
  call alloc(open, nm)
  call alloc(high, nm)
  call alloc(low, nm)
  call alloc(close, nm)
  call alloc(adjclose, nm)
  call alloc(volume, nm)
 
  open(newunit=fileunit, file=filename) ! open the file
  read(fileunit, fmt=*, end=1) ! use read() to skip the CSV header
  do n = 1, nm ! loop over records and store into array elements
    read(fileunit, fmt=*, end=1) time(n), open(n),&
      high(n), low(n), close(n), adjclose(n), volume(n)
  end do
  1 close(fileunit) ! close file when done
 
end subroutine read_stock
```

To find the length of the arrays before I allocate them, 
I inquire the length of the CSV file using a custom function `num_records`, 
defined in `stock-prices/src/mod_io.f90`. If you’re wondering what the number 1 
means in the `1 close(fileunit)` line, it’s just a line label that Fortran uses 
if and when it encounters an exception in the `read(fileunit, fmt=*, end=1)` statements. 
If you’re interested about how this function works, take a look inside `stock-prices/src/mod_io.f90`. 
I won’t spend much time on the I/O-specific code here, 
as we just need it get going with the array analysis.

On every subroutine entry, the arrays `time`, `open`, `high`, `low`, `close`, `adjclose`, and `volume` 
will be allocated with size `nm`. The subroutine `alloc` now seamlessly reallocates the arrays for us. 
Notice that we still use the explicit way of allocating and deallocating the array of time stamps. 
This is because we implemented the convenience subroutines `alloc` and `free` that work on real arrays. 
Because of Fortran’s strong typing discipline, we can’t just pass an array of strings to a subroutine 
that expects an array of reals. We’ll learn later how to write generic procedures that can accept arguments 
of different type. For now, explicitly allocating the array of time stamps will do. 
Furthermore, we also need to specify the string length when allocating the time array.

Having read the CSV files and loaded the stock price arrays with the data, 
we can move on the the actual analysis and fun with arrays.

> **Getting the number of lines in a text file**: If you’re curious how the `num_records` function is implemented, 
feel free to take a look at `stock-prices/src/mod_io.f90`. This function opens a file and counts the number 
of lines by reading it line by line.

## Indexing and slicing arrays

Did you notice that the stock data in the CSV files are ordered from most recent to oldest? 
This means that when we read it into arrays from top-to-bottom, 
the first element will correspond to the most recent stock price. 
Let’s reverse the arrays so that they are oriented in a more natural way, 
going forward in time with the index number. 
If we express the reverse operation as a function, 
we could apply it to any array like this:

```fortran
adjclose = reverse(adjclose)
```

The reverse function will prove useful for the other two objectives of the stock-prices app. 
Before implementing it, we need to know a few things about how array indexing and slicing works.

To select a single element, we enclose an integer index inside the parentheses, 
for example `adjclose(1)` will refer to the first element of the array, 
`adjclose(5)` to the fifth, and so on.

To select a range of elements, for example from fifth to tenth, 
use the start and end indices inside the parentheses, separated by a colon:

```fortran
real, allocatable :: subset(:)
  ...
subset = adjclose(5:10)
```

In this case, subset will be automatically allocated as an array with 6 elements, 
and values corresponding to those of `adjclose` from index 5 to 10.

By default, the slice `adjclose(start:end)` will include all elements between indices start and end, inclusive. 
However, you can specify an arbitrary stride. For example, `adjclose(5:10:2)` will result in a slice with elements 5, 7, and 9. 
The general syntax for slicing an array a with a custom stride is:

```fortran
a(start:end:stride)
```

where `start`, `end`, and `stride` are integer variables, constants, or expressions. 
`start` and `end` can have any valid integer value, including zero and negative values. 
`stride` must be a non-zero (positive or negative) integer.

Similar rules apply as for `start`, `end`, and `stride` of `do`-loops:

1. If `stride` is not given, its default value is 1.
2. If `start` > `end` and `stride > 0`, or if `start < end` and `stride < 0`, the slice is an empty array.
3. If `start == end`, for example `a(5:5)`, the slice is an array with a single element. 
Be careful not to mistake this for `a(5)` which is a scalar and not an array.

Furthermore, if `start` equals the lower bound of an array, it can be omitted, 
and same is true if `end` equals the upper bound of an array. 
For example, if we declare an array as `real :: a(10:20)`, 
then the following array references and slices all correspond to the same array: 
`a`, `a(:)`, `a(10:20)`, `a(10:)`, `a(:20)`, `a(::1)`. 
The last syntax from this list is particularly useful when you need to slice 
every n-th element of the whole array — it’s as simple as `a(::n)`.
If you have experience with slicing lists in Python, I bet this feels familiar.

### Intermezzo: Reversing an array

Let’s write a function `reverse` that accepts a real 1-dimensional array as input argument, 
and returns the same array in reverse order. We’ll use array slicing rules to perform the reversal.

We can solve this in only two steps and write it as a single-line function (not counting the declaration code). 
First, we know that since we are just reversing the order of elements, 
the resulting array will always be of same size as the input array. 
The size will also correspond to the end index of the array. 
Second, once we know the size, we can slice the input from last to first 
and use the negative stride to go backward:

```fortran
pure function reverse(x)
  real, intent(in) :: x(:) ! assumed-size input array
  real :: reverse(size(x)) ! declare reverse with same size as x
  reverse = x(size(x):1:-1) ! use negative stride to copy backwards
end function reverse
```

Notice that our input array doesn’t need to be declared `allocatable`. 
This is the so-called assumed-size array and it will take whatever size array is passed by the caller. 
We can use the information about the size directly when declaring the result array.

You can test your new function by reversing the input array twice and comparing it to itself:

```fortran
print *, all(a == reverse(reverse(a))) ! should always print “T”
```

You may be wondering why make this a separate function at all when we can just do `x(size(x):1:-1)` to reverse any array. 
There are two advantages to making this a dedicated `reverse()` function. 
First, if you need to reverse an array more than a few times, the slicing syntax above soon becomes unwieldy. 
Every time you read it, there is an extra step in the thought process to understand the intention behind the syntax. 
Second, the slicing syntax is allowed only when referencing an array variable, and you can’t use it on expressions, 
array constructors, or function results. In contrast, you can pass any of those as an argument to `reverse()`. 
This is why we can make such a test like `all(x == reverse(reverse(x)))`. Try it!

Take some time to play with various ways to slice arrays. 
Try different values of `start`, `end`, and `stride`. 
What happens if you try to create a slice that is bigger than the array itself? 
In other words, can you reference an array out-of-bounds?

> **Referencing array elements out of bounds**: Be very careful to not reference array elements that are out of bounds! 
> Fortran itself doesn’t forbid this, but you will either end up with an invalid value or with a segmentation fault, 
> which can be particularly difficult to debug.
> 
> By default, compilers don’t check if an out-of-bounds reference occurs during run-time, 
> but you can enable it with a compiler flag. Use `gfortran -fcheck=bounds` and `ifort -check bounds` 
> for GNU and Intel Fortran compilers respectively. 
> Keep in mind that this can result in significantly slower programs, 
> so it’s best if used during development and debugging, but not in production.

Now that we understand how array indexing works, 
it’s straightforward to calculate the stock gain over the whole time series. 
It’s simply a matter of taking a difference between last and first element 
of the adjusted close price to calculate the absolute gain in USD:

```fortran
adjclose = reverse(adjclose)
gain = (adjclose(size(adjclose)) — adjclose(1))
```

Here, I am using the intrinsic `size` function, which returns the integer total number of elements, 
to reference the last element of the array. Like everything else we did before, `gain` must be declared, 
in this case a real scalar. The absolute gain, however, only tells us how much the stock grew over a period of time, 
but it doesn’t tell us anything about whether that growth is small or large relative to the stock price itself. 
For example, a gain from <span>$</span>1 to <span>$</span>10 per share 
is greater than the gain from <span>$</span>100 to <span>$</span>200 per share, 
assuming you invest <span>$</span>100 in either stock. In the former case, you will come out with <span>$</span>1000, 
whereas in the latter case you will come out with just <span>$</span>200! To calculate the relative gain in percent, 
we can divide the absolute gain by the initial stock price, multiply by 100 to get to percent, 
that is, `gain / adjclose(1) * 100`. For brevity, I will also round the relative gain to the nearest 
integer using the intrinsic function `nint`:

```fortran
print *, symbols(n), gain, nint(gain / adjclose(1) * 100)
```

The output of the program is:

```
2000–01–03 through 2018–05–14
Symbol, Gain (USD), Relative gain (%)
---------------------------------
AAPL   184.594589            5192
AMZN   1512.16003            1692
CRAY   9.60000038              56
CSCO   1.71649933               4
HPQ    1.55270004               7
IBM    60.9193039              73
INTC   25.8368015              89
MSFT   59.4120979             154
NVDA   251.745300            6964
ORCL   20.3501987              77
```

From this output, we can see that Amazon had the largest absolute gain of <span>$</span>1512.16 per share, 
and Hewlett-Packard had the smallest gain of only <span>$</span>1.55 per share. 
However, the relative gain is more meaningful then the absolute amount per share because 
it tells us how much has the stock gained relative to its starting price. 
Looking at relative gain, Nvidia had a formidable 6864% growth, 
with Apple being the runner up with 5192%. 
The worst performing stock was that of Cisco Systems (CSCO), 
with only 4% growth over this time period.

If you have cloned the stock-prices repo from Github, 
it’s straightforward to compile and run this program. 
From the `stock-prices` directory, type:

```
make
./stock_gain
```

You can read the full program in `stock-prices/src/stock_gain.f90`.

We have now covered a lot of the nitty-gritty of how arrays work. 
We’ll use this knowledge in Part 3 of this article sequence to 
implement solutions to the remaining two stock price analysis challenges.

## Summary

In this article, we continued from where we left of at the end of Part 1, 
and dug deeper into the mechanics of allocation and deallocation of dynamic Fortran arrays. 
We covered allocating arrays with arbitrary index ranges, and saw how we can slice and dice 
through the arrays with custom strides. In this article we also implemented the solution 
to the first of the three challenges. In the third and final part of this sequence, 
we’ll use built-in Fortran functions and whole-array arithmetic to calculate metrics 
such as moving average and standard deviation. 
We’ll use these to tackle the remaining two challenges — identifying which stocks 
are riskier than others at any given time, and finding good times to buy or sell stock.

If you want to learn more about the book, check it out on liveBook [here](http://bit.ly/2EcJ5HL)
and see this [slide deck](http://bit.ly/2EdMGpc). Take 37% off [*Modern Fortran*](http://bit.ly/2EeLiT6). 
Just enter code **fcccurcic** into the discount
box at checkout at [manning.com](http://bit.ly/2EaViNj).

---
layout: post
title: "Analyzing stock price time series with modern Fortran, Part 1"
author: "Milan Curcic"
---

![]({{ "/assets/chris-liverani-screen.jpg" | absolute_url }})

This article is an excerpt from my book on [Modern Fortran](http://bit.ly/2EeLiT6),
and originally published [here](https://freecontent.manning.com/analyzing-stock-price-time-series-with-fortran-arrays-part1/).
You can also read it on [Medium](https://medium.com/modern-fortran/analyzing-stock-price-time-series-with-modern-fortran-part-1-a0282e1012eb).

## Declaring and initializing arrays for stock price data

Stock price analysis and prediction has been an increasingly popular topic
since the early days of high-level programming, and Fortran has been used in
the bowels of many financial trading and banking systems, mainly thanks to its
robustness, reliability, and efficiency. In this article, we’ll work with a
dataset that is freely available, small enough to be easily downloaded, and yet
large enough to demonstrate the power of Fortran arrays.

I’m no trader and I won’t go into the details of true technical stock market
analysis. Therefore I recommend that you don’t use this article as trading
advice. Instead, I will merely show you how you can leverage the power of
Fortran arrays to perform any kind of time series analysis that you can think of,
whether it’s stock or commodity prices, measurements, or signal processing.

Jump to:

* [Introduction](#introduction)
  - [Objectives](#objectives)
  - [About the data](#about-the-data)
  - [Getting the data and code](#getting-the-data-and-code)
* [Finding the best and worst performing stocks](#finding-the-best-and-worst-performing-stocks)
  - [Declaring arrays](#declaring-arrays)
  - [Array constructors](#array-constructors)
  - [Implicit save](#implicit-save)
  - [Combining different numeric types in expressions](#combining-different-numeric-types-in-expressions)
  - [Reading stock data from files](#reading-stock-data-from-files)
* [Summary](#summary)

## Introduction

An array is a sequence of data elements of same type that are also contiguous
in memory. While this may at first seem restrictive, it does come with advantages.
First, it allows you to write simpler code with expressive one-liners that can
work on millions of elements at once. While arrays have been part of Fortran
since its birth, whole-array operators and arithmetic have been introduced in
Fortran 90, allowing programmers to write cleaner, shorter, and less error-prone
code. In a nutshell, arrays allow you to easily work on large datasets and apply
functions and arithmetic operators on whole arrays without resorting to loops or
other verbose syntax.

In this article, we’ll take a dive into Fortran arrays and learn the mechanics
of declaring arrays, allocating them in memory, and using them with familiar
arithmetic operators. We’ll do this by writing a small stock price analysis app.
You will learn how to declare, allocate and initialize dynamic arrays, read and
store data into them, and then perform whole-array arithmetic to quantify stock
performance, volatility, and other metrics. Let’s get started by setting some
objectives for our application, and looking at the data that we’ll work with.

### Objectives

Let’s set some tangible goals for this exercise. We can think of them as
challenges.

1. **Find the best and worst performing stocks**. We’ll first evaluate which
stock grew (or lost value) the most, relative to its starting price. To do this,
we’ll need to know the stock price from the start and the end of the time series,
and calculate the difference relative to the initial price. In this challenge,
you will learn how to declare, allocate, and initialize dynamic arrays,
calculate the size of an array, and reference individual array elements.

2. **Identify risky stocks**. Some stocks are just riskier than others. This
can be quantified by the so-called stock volatility, which is related to the
standard deviation of the stock price. Standard deviation is a statistical
measure of how much do the values deviate from the average value, and it can be
defined over arbitrary time periods. In this challenge, you will learn how to
slice arrays and perform whole-array arithmetic.

3. **Identify good times to buy and sell**. Traders commonly use a technique
called the moving average crossover to decide whether it is a good time to buy
or sell a certain stock. The moving average crossover tells us when the stock
price crosses the moving average (average over a limited time window), and is
an indicator of a change in longer term trend of the stock price, despite the
high-frequency fluctuations. In this challenge, you will learn how to search the
arrays for specific values and extract elements that meet any criteria that you want.

Before we dive into implementing the solutions to these challenges, let’s first
get familiar with the data that we’ll work with.

### About the data

We’ll work on daily stock price time series from 10 technology companies,
including Apple, Amazon, and Intel. The data is stored in the comma-separated
value (CSV) format. Here’s a sample of the Apple stock daily data. Rows are
ordered from newest to oldest:

```
timestamp,open,high,low,close,adjusted_close,volume,dividend_amount,split_coefficient
2018-05-14,189.0100,189.5300,187.8600,188.1500,188.1500,20364542,0.000,1.0000
2018-05-11,189.4900,190.0600,187.4500,188.5900,188.5900,26212221,0.7300,1.0000
2018-05-10,187.7400,190.3700,187.6500,190.0400,189.3072,27989289,0.0000,1.0000
2018-05-09,186.5500,187.4000,185.2200,187.3600,186.6376,23211241,0.0000,1.0000
2018-05-08,184.9900,186.2200,183.6650,186.0500,185.3326,28402777,0.0000,1.0000
...
```

The columns in each CSV file are:

* `timestamp`: Date in `YYYY-mm-dd` format.
* `open`: Opening price at the start of the trading day.
* `high`: Highest price that the stock reached during the trading day.
* `low`: Lowest price that the stock reached during the trading day.
* `close`: Closing price at the end of the trading day. This reflects the price
of the last stock that was traded that day.
* `adjusted_close`: Closing price that has been retroactively adjusted for stock
splits (see split coefficient below).
* `volume`: The total number of shares traded during the day.
* `dividend_amount`: The amount of dividend paid per share.
* `split_coefficient`: Occasionally, a stock can be split for various reasons,
and the split coefficient indicates the factor by which the stock was split.
If `split_coefficient` is 1, the stock was not split. If it is 0.5, the stock
price is halved due to the split.

For a birds-eye view of how these stocks performed since 2000, I plotted the
adjusted close price against time:

![]({{ "/assets/adjusted-close-price.png" | absolute_url }})
**Figure 1**: Adjusted close stock prices (USD) of 10 technology companies. The y-axis on each panel has different scale.

Most of the companies had their stock grow in the overall. We can even spot some
trends. For example, IBM grew considerably from 2009 to 2012 following the
growth of cloud-based technologies and expansion in that area. Nvidia (NVDA)
entered a period of explosive growth in early 2016 thanks to the mass adoption
of their GPUs for machine learning.

You may be wondering, why use the adjusted close and not just the closing price?
Occasionally, a company splits its stock, resulting in a much different closing
price than the day before. We could see this in effect in June of 2014 when
Apple split its stock sevenfold:

```
timestamp,open,high,low,close,adjusted_close,volume,dividend_amount,split_coefficient 2014-06-09,92.7000,93.8800,91.7500,93.7000,87.1866,75414997,0.0000,7.0000 2014-06-06,649.9000,651.2600,644.4700,645.5700,85.8134,12497800,0.0000,1.0000
```

On June 6, the closing price was <span>$</span>645.57, while on the next trading day, June 9
(exchange markets close on weekends), the opening price was <span>$</span>92.70. Notice that
the split coefficient on this day is 7, indicating the factor by which the stock
price was divided. If we analyzed long time series of closing prices, we would
also capture occasional large increases or drops due to these events that do not
reflect the market value of the stock. Adjusted closing price retroactively
accounts for all stock splits that occured, and results in price time series
that are consistent with actual stock value. It is thus useful when analyzing
long-term historical performance of a stock.

### Getting the data and code

The full code for the stock analysis exercise is available on [Github](https://github.com/modern-fortran/stock-prices).
If you use git, you can clone it directly from the command line:

```
git clone https://github.com/modern-fortran/stock-prices
```

Otherwise, you can download it as a [zip file](https://github.com/modern-fortran/stock-prices/archive/master.zip).

The repository already includes the stock price data needed for this exercise
in the `stock-prices/data` directory. However, if this exercise leaves you
hungry for more in-depth analysis or larger stock price datasets, there’s an
easy way to get more.

To download the stock data in CSV format for this exercise, I used the free
service [Alpha Vantage](https://alphavantage.co) which provides an HTTP API to
obtain data in JSON or CSV format. To download your own data beyond that used in
this article, you will need to register for the API key at their website. Once
you have one, you can make your own API requests to get various stock data. For
an example, see the download script that I used to download the daily data for
this exercise in `stock-prices/data/get_data.sh`.

Cloning the repository (or downloading the zip file) will also get you the
complete code that implements the 3 data analysis challenges. We’ll implement
these step-by-step in the following sections, so if you want to follow along,
defer reading the final code until the end of the article.

## Finding the best and worst performing stocks

Let’s start from the basics. Before we do any data analysis, we need to take
care of the logistics. The following steps will generally apply to each of the
three challenges in this exercise:

1. Define the arrays to hold the data. We’ll learn how to declare dynamic arrays
whose size is not known at compile time. In this article, we’ll still rely on
core numeric types: real for stock prices and character for time stamps.
2. Read data from the CSV files. We’ll implement a custom subroutine to read the
files and store the data into arrays defined in step 1.
3. Calculate statistics from raw data. Most of the crunchy stuff described in
this article will deal with this step.

For a start, we can estimate the performance of different stocks by calculating
their gain over the whole period of the time series data — from January 2000 to
May 2018. When we implement the solution to our first challenge, the output of
the program will look like this:

```
2000-01-03 through 2018-05-14
Symbol, Gain (USD), Relative gain (%)
-------------------------------------
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

For each stock, we’ll calculate its gain, that is, the difference between the
closing price at the end of the beginning of the time series, and the gain in
percent relative to the starting price. The main program (not including the
utility functions) looks like this:

```fortran
program stock_gain

  use mod_arrays, only: reverse
  use mod_io, only: read_stock

  implicit none

  character(len=4), allocatable :: symbols(:)
  character(len=:), allocatable :: time(:)
  real, allocatable :: open(:), high(:), low(:),&
                       close(:), adjclose(:), volume(:)
  integer :: i, im, n
  real :: gain

  symbols = ['AAPL', 'AMZN', 'CRAY', 'CSCO', 'HPQ ',&
             'IBM ', 'INTC', 'MSFT', 'NVDA', 'ORCL']

  do n = 1, size(symbols)

    call read_stock('data/' // trim(symbols(n)) //  '.csv',&
      time, open, high, low, close, adjclose, volume)

    adjclose = reverse(adjclose)
    gain = (adjclose(size(adjclose)) - adjclose(1))

    if (n == 1) then
      print *, time(size(time)) // ' through ' // time(1)
      print *, 'Symbol, Gain (USD), Relative gain (%)'
      print *, '-------------------------------------'
    end if

    print *, symbols(n), gain, nint(gain / adjclose(1) * 100)

  end do

end program stock_gain
```

In this program, we first declare the dynamic (allocatable) arrays to hold the
list of stock symbols, and time stamps and stock price data for each stock. We
then loop over each stock, and read the data from CSV files one at a time.
Finally, we calculate the difference between end and start price, and print the
results to screen. The following few sections go into details of how dynamic
Fortran arrays work, specifically, how to declare them, allocate in memory,
initialize their values, and finally clear them from memory when done.

### Declaring arrays

The basic way to declare a Fortran array of fixed size is by indicating the
size in the parentheses:

```fortran
real :: u(100) ! declare a real array u with 100 elements
```

When you specify the size of the array at the declaration line, like I did in
the snippet above, you tell the compiler to declare a static array. The size of
the array is known at compile time, and the compiler can use this information to
generate more efficient machine code. Effectively, when you declare a static
array, it is allocated in memory when you run the program.

However, you will not always know the size of the arrays ahead of time. It just
so happens that each stock data CSV file has the same number of records (4620),
but this may not always be the case, as some companies may have much longer
presence in the public markets than others. Furthermore, if you choose to later
work on a different or larger stock prices dataset, it would be unwieldy to have
to hardcode the size of the arrays every time. This is where the dynamic, or in
Fortran lingo, allocatable arrays, come in. Whenever the size of the array is
not known ahead of time, or, you anticipate that it will change at any time
during the life of the program, declare it as allocatable:

```fortran
real, allocatable :: u(:) ! declare a dynamic array u
```

Writing more general and flexible apps will also require allocating arrays at
run-time.

Notice that there are two key changes here relative to declaring a static array.
We added the allocatable attribute, and we used the colon (`:`) as a placeholder
for the array size. At this point in the code, we did not allocate this array in
memory, but simply stated “We’ll use a real, 1-dimensional array `u`, whose size
is yet to be determined.”

You’re likely wondering when to use dynamic over static arrays? Use dynamic over
static arrays whenever you don’t know the size of the arrays ahead of time, or
know that it will change. A few examples come to mind:

* Storing user-input data, entered either by standard input (keyboard) or read
from an input file;
* Reading data from multiple files of different length;
* Arrays that will be re-used across datasets;
* Arrays that may grow or shrink during the lifetime of the program.

Dynamic arrays will help you write more general and flexible code, but may carry
a performance penalty as allocation of memory is a slow operation compared to,
say, floating-point arithmetic.

It does seem that for our use case, we should use dynamic arrays. Following the
data description from the previous section, we’ll need:

* An array of character strings to hold stock symbols (AAPL, AMZN, etc.)
* An array of character strings to hold the time stamps (`2018-05-15`,
  `2018-05-14`, etc.)
* Arrays of real numbers to hold the actual stock data, such as opening and
closing prices, and others.

We can apply the dynamic array declaration syntax to declare these arrays:

```fortran
program stock_gain

  implicit none

  character(len=4), allocatable :: symbols(:)
  character(len=:), allocatable :: time(:)
  real, allocatable :: open(:), high(:), low(:), close(:),&
                       adjclose(:), volume(:)

end program stock_gain
```

Notice that for symbols I declared an array of character strings of length,
while for the time array I didn’t specify the length ahead of time (len=:).
This is because we’ll determine the length of timestamps in the subroutine that
is in charge of reading the data files, and we don’t need to hardcode the length
here. For the rest of the data, I declared real (floating point) arrays. Even
though the volume is an integer quantity (number of shares traded), real will
work just fine for typical volume values and will help simplify the code. You
can compile and run this program, but it won’t do anything useful at this point
since it only declares the arrays that we’ll use. To loop over stock symbols and
print each to screen, we’ll use an array constructor to initialize the array symbols.

### Array constructors

Let’s begin with a modest first step — initializing the stock symbols that we’ll
work on. When we properly initialize the stock symbols, loop over them and print
each to screen, the output will look like this:

```
Working on AAPL
Working on AMZN
Working on CRAY
Working on CSCO
Working on HPQ
Working on IBM
Working on INTC
Working on MSFT
Working on NVDA
Working on ORCL
```

As you can imagine, specific symbols depend on what data we have. Since in this
exercise we’ll work with only 10 stocks, we can type these directly in code:

```fortran
program stock_gain
  ...
  integer :: n

  symbols = ['AAPL', 'AMZN', 'CRAY', 'CSCO', 'HPQ ',&
             'IBM ', 'INTC', 'MSFT', 'NVDA', 'ORCL']

  do n = 1, size(symbols)
    write(*,*) 'Working on ' // symbols(n)
  end do

end program stock_gain
```

Here for the first time I invoke the intrinsic function size, which returns an
integer size of an input array, in this case 10. I will show you how size works
in more details a bit later in this article.

I also introduce a new syntax element, the so-called array constructor, to
assign stock symbols to the symbols array. Array constructors allow you to
create arrays on the fly and assign them to array variables:

```fortran
integer :: a(5) = [1, 2, 3, 4, 5]
```

In the example above, I used the square brackets to enclose a sequence of five
integers. Together, this syntax forms a literal constant array that is then
assigned to `a`. For static arrays, the size and shape of the array constructor
must match the size and shape of the array variable on the left-hand side.

In the above snippet, I initialized a on the declaration line. This makes for
an easy and concise declaration and initialization of a small array. However,
there’s one exception case in which you’re not allowed to do this: pure
procedures. In that case, you have no choice but to declare and initialize on
separate statements:

```fortran
integer :: a(5)
a = [1, 2, 3, 4, 5]
```

This is no big deal, but you may rightfully ask, why this restriction? It stems
from a historical feature of Fortran called implicit save behavior.

### Implicit save

Adding a save attribute to the declaration statement in a procedure causes the
value of the declared variable to be saved between calls. In other words, the
procedure would “remember” the value of that saved variable. Now, here’s the
twist: If you initialize a variable in the declaration statement, this will
implicitly add the save attribute to the declaration. A variable with the save
attribute will maintain its value in memory between procedure calls. Since this
is a side effect, it can’t be used in pure procedures.

I don’t recommend using the save attribute, or relying on implicit save feature
to maintain state between calls. In main programs and modules, it’s harmless
and you can safely initialize on declaration. In procedures (if not pure), you
can initialize on declaration, but I recommend not exploiting the implicit save
behavior as it results in more stateful and bug-prone code.

There is another, more general way of constructing an array. In the trivial
example above, I assigned to a an array of 5 elements, and they were easy to
type in by hand. However, what if you wanted to assign a hundred or a thousand
elements? This is where we can use the so-called implied do-loop constructor:

```fortran
integer, allocatable :: a(:)
integer :: i

a = [(i, i = 1, 100)]
```

The above syntax is called an implied do-loop because `(i, i = 1, 100)` is just
syntactic sugar for an explicit do-loop:

```fortran
do i = 1, 100
  a(i) = i
end do
```

With an implied do-loop array constructor, you are not restricted to just the
loop counter. You can use it to assign array values from arbitrary functions or
expressions:

```fortran
real, allocatable :: a(:)
integer :: i
real, parameter :: pi = 3.14159256

a = [(sin(2 * pi * i / 1000), i = 0, 1000)]
```

Here, I used the integer index `i` to construct an array of sines with arguments
that go from 0 to 2π in 1000 steps. However, you don’t have to use the index in
the expression, but merely as a way to repeat the same element over and over
again. For example, initializing an array of thousand zeros is trivial:

```fortran
a = [(0, i = 1, 1000)]
```

Finally, Fortran also lets you create empty arrays using `[integer ::]` or
`[real ::]`. In practice, these could be useful if invoking a generator – a
kind of function that adds an element to an array on every call.

### Combining different numeric types in expressions

Notice that in the above listing I have mixed integer and real variables in a
single expression: `sin(2 * pi * i / 1000)`. What is the type of the result
then? Integer or real? Fortran follows two simple rules:

1. The expression is first evaluated to the strongest (most precise) type. For
example, multiplying a real with an integer always results in a real, and
multiplying a complex number with either real or integer always results in a
complex number. Same goes for kinds of different precision – adding a `real32`
to a `real64` results in a `real64` value.
2. If you’re assigning the result of the expression to a variable, its type is
automatically promoted (or downgraded!) to the type of the variable.

In this specific example, 2, `i`, and 1000 are integers, and `pi` is a real.
The whole expression is thus a real number. This is generally known as type coercion.

### Reading stock data from files

Now that we have the list of stock symbols that we’ll work on, let’s use this
information to load the data from file and store it in our newly declared
dynamic arrays. The prototype of our main loop should look like this:

```fortran
do n = 1, size(symbols)
  call read_stock('data/' // trim(symbols(n)) //  '.csv', time,&
    open, high, low, close, adjclose, volume)
end do
```

However, we haven’t implemented the read_stock subroutine yet! Based on the
calling signature, we should pass the file name as the first argument, an array
of times as the second, and 6 real arrays to hold the stock data as the remaining
arguments. At this point, we are passing arrays that have not been allocated yet.
As we iterate over the stocks, we’ll need to explicitly allocate our arrays
before loading the data from files. The declaration of data in our `read_stock`
subroutine prototype may thus look something like this:

```fortran
subroutine read_stock(filename, time, open, high, low,&
                      close, adjclose, volume)
  character(len=*), intent(in) :: filename
  character(len=:), allocatable, intent(in out) :: time(:)
  real, allocatable, intent(in out) :: open(:), high(:), low(:),&
    close(:), adjclose(:), volume(:)
  ...
end subroutine read_stock
```

Let’s look at our arguments in this subroutine definition. `filename` is
declared as `character(len=*)`. This is the so-called assumed-length character
string. It says, whatever the length of the string is passed to the subroutine,
this argument will accept and assume that length. This is useful whenever you
want to pass character strings that are of either varying or unpredictable
length. time, however, is declared as `character(len=:)` allocatable array to
match the declaration in the calling program. Finally, the arrays to hold the
actual stock data are declared as real and allocatable. We’re using
`intent(in out)` for all subroutine arguments, which means that they will be
passed back and forth between the main program and the subroutine. Notice also
that here we’ve matched the data type and `allocatable` attributes for the stock
data with those declared in the main program.

## Summary

In this article, we covered the essentials of declaring and initializing
Fortran arrays. Specifically, we focused on the so-called allocatable (dynamic)
arrays, whose size and range isn’t known at compile time, and is determined at
run time instead. You’ll find yourself to rely on allocatable arrays whenever
you need to work with real-world data with records of varying shape and size.
Finally, I’ve introduced the concept of array constructors, which is a convenient
way of allocating and initializing Fortran arrays at the same time. In Part 2 of
this article, we’ll dive into the details of how explicit allocation and
deallocation works. This will allow you to specify custom index range for your
arrays, as well as use built-in exception handling to make your code more robust
and fault-tolerant. We’ll also cover slicing arrays with arbitrary range and stride.
We’ll use these techniques to tackle the first stock price analysis challenge –
finding the best and worst performing stocks.

If you want to learn more about the book, check it out on liveBook [here](http://bit.ly/2EcJ5HL)
and see this [slide deck](http://bit.ly/2EdMGpc). Take 37% off [*Modern Fortran](http://bit.ly/2EeLiT6). Just enter code **fcccurcic** into the discount
box at checkout at [manning.com](http://bit.ly/2EaViNj).

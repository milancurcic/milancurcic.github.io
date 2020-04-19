---
layout: page
title: Software
permalink: /software/
---

I make software almost daily
for my [research](https://milancurcic.com/publications), 
[business](https://cloudrun.co), 
and [writing](https://www.manning.com/books/modern-fortran?a_aid=modernfortran&a_bid=2dc4d442). 
Some of it eventually becomes open source. Enjoy! :)

## Simulation

* [University of Miami Wave Model](https://umwm.org):
A parallel spectral ocean wave model.
It's been used in research and production for a variety of applications.
It's particularly good for coupling with weather and ocean circulation models.
Read the paper [here](https://github.com/milancurcic/publications/blob/master/Donelan_etal_JGR2012.pdf).
* [wavy](https://github.com/wavebitscientific/wavy):
A library to build a hierarchy of ocean wave models.
Useful for research and toy models, less so for production.
* [tsunami](https://github.com/modern-fortran/tsunami):
A parallel shallow water solver used to teach modern and parallel Fortran.
Developed as the running example for the 
[Modern Fortran](https://www.manning.com/books/modern-fortran?a_aid=modernfortran&a_bid=2dc4d442) book.

## Machine learning

* [neural-fortran](https://github.com/modern-fortran/neural-fortran):
A parallel neural net microframework.
Proof-of-concept with promising performance and ease of use.
It supports deep feed-forward networks of arbitrary size, 
several activations functions, and data-based parallelism.
Read the paper [here](https://arxiv.org/abs/1902.06714).

## General purpose

* [Fortran stdlib](https://fortran-lang.org/stdlib): Community-developed Fortran Standard Library.
* [datetime-fortran](https://github.com/wavebitscientific/datetime-fortran):
Date and time library for Fortran.
* [functional-fortran](https://github.com/wavebitscientific/functional-fortran):
Functional programming in Fortran, with support for all built-in types.
* [tcp-client-server](https://github.com/modern-fortran/tcp-client-server):
A minimal Fortran TCP client and server, used to teach interoperability with C.
Uses [libdill](http://libdill.org) as the sockets library.

## Devices / embedded

* [scanivalve-mps-python](https://github.com/sustain-lab/scanivalve-mps-python):
A high-level interface to a 64-port miniature pressure sensor,
built on top of the TCP protocol.
* [arduino-wavemaker](https://github.com/sustain-lab/arduino-wavemaker):
Programming Arduino Uno to power a linear actuator for making waves in a tank.

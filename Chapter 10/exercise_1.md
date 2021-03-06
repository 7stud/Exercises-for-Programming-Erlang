If you haven't tried **eunit** for testing yet, the simplest way to use **eunit** is to run all the functions named \*_test in a file.  For example, suppose you have a file like this:

```erlang
-moduel(a).
-export([do/0, go/0]).

do_test() ->
    dummy.

do() ->
    dummy.

go_test() ->
    dummy.

go() ->
    dummy.

    
```

You can run a specific test function just like any other function:
```erlang
1> c(a).
{ok,a}

2> a:do_test().
dummy
```
With **eunit**, you can run both `do_test()` and `go_test()` with one command:

```erlang
4> c(a).
{ok,a}

5> a:test().
  All 2 tests passed.
ok
```
You can enable **eunit** by adding the following line to the top of your file:

    -include_lib("eunit/include/eunit.hrl").
    
Then make sure that the names of your test functions end in `_test` (eunit automatically exports the test functions for you). The ability of eunit to run all the test functions in a file with one command is handy for solving this exercise.

The book uses the old "suffix rules" in the makefile, which have been replaced by "pattern rules".  I put a modern interpretation of a basic makefile alongside this file in the Chapter 10 directory. It is heavily commented with explanations. Here is that makefile modified to run all the test functions in all the modules in your project: 

```makefile
modules = a b    #Assign all the names of your modules to a variable.

.PHONY: all
all: $(modules:%=%.beam) test  #Added an additonal prerequisite file named test.
                               #make needs to look further down the makefile to 
                               #figure out how to create the test file.
%.beam: %.erl
	erlc -W $< 

.PHONY: test
test: $(modules:%=%_test)      #This tells make how to create the test file: make needs
                               #to create the prerequisite files a_test and b_test.

%_test:                        #This tells make how to create a *_test file.
	erl -noshell -s $* test -s init stop  # $* is the part of the file name matched by the % wildcard

.PHONY: clean
clean:
	rm $(modules:%=%.beam) erl_crash.dump

```

On the command line:
```
~/erlang_programs/ex10_1$ gmake
erlc -W a.erl 
erlc -W b.erl 
erl -noshell -s a test -s init stop
  All 2 tests passed.
erl -noshell -s b test -s init stop
  All 2 tests passed.
```
Note that make shows you the commands it's executing as well as the stdout of those commands.

I just learned about makefiles while reading Chapter 10. If you are familiar with C/C++:

http://www.cs.colby.edu/maxwell/courses/tutorials/maketutor/

Other really excellent makefile resources:

http://www.oreilly.com/openbook/make3/book/ch02.pdf

https://www.gnu.org/software/make/manual/make.html#Overview

(Oh yeah, use gmake instead of make!)






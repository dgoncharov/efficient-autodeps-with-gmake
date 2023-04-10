## Efficient autodeps with gmake.


The modern technique of tracking dependencies relies on include directive
[[1]](http://make.mad-scientist.net/papers/advanced-auto-dependency-generation).
While this technique is infinitely superior to the manual maintenance of deps
there is still room for improvement.


### 1. include is not a part of the dag.


```
src:=$(wildcard *.c)
dfiles:=$(src:.c=.d)
%.o: %.c %.d
	gcc -I. -MMD -o $@ -c $<
%.d: ;
include $(dfiles)
```

When the user runs

```
$ make hello.o
```

make includes all dep files in the current directory, even though only hello.d
is needed. This is not optimal.

include is not cheap. Even in a medium project, with multiple source files
include of all dep files usually takes more than 50% of make total time (make
itself, not its children, such as cc).


### 2. One makefile for multiple programs.

include is especially problematic when a project has one test driver per
source file. E.g. a lib can have implementation files api.c, util.c, and
engine.c and test programs api.t.c, util.t.c and engine.t.c. Each test program
contains function main and is supposed to be compiled and linked with the
module that it tests.


```
%.t.tsk: %.o %.t.o
	$(CC) -o $@ $^
```

When the user runs

```
$ make api.t.tsk
```

Only api.d and api.t.d are required to be included. However, our makefile shown
above will include all dep files.



### 3. One makefile for multiple projects.

Another scenario is a makefile containing a set of implicit rules and shared
between multiple projects.

include takes filenames as parameters. This hinders reuse of the makefile. We
would rather not hardcode filenames of included files in a makefile, but let
implicit rule search figure it out.



### 4. Phony targets.

Another issue with unconditional include is phony targets, such as clean, gzip,
etc.

The usual workaround is to to figure out the specified target and have a set of
ifeq statements to avoid including dep files. This piece of ifeq code
is difficult, especially when there are a lot of phony targets and the user
can specify multiple targets on the command line. E.g.

```
$ make api.t.tsk install
```



### Solution.

It is possible to achieve the same automatic tracking of dependencies with
include (or rather its substitue) being a part of the dag.


```
SHELL=/bin/bash
.SECONDEXPANSION: %.o

%.o: %.c %.d $$(file <%.d)
	gcc $(CPPFLAGS) $(CFLAGS) -MD -MF $*.td -o $@ -c $<
	read obj src headers <$*.td; echo "$$headers" >$*.d
	touch -c $@

%.d: ;
%.h: ;
```

1. -MD generates a regular dep file.
Note, gcc option -MP is not used. The contents of dep files is of the form

```
api.o: api.c api.h stddef.h stdlib.h\
    stdio.h...\
    <more header files>...
```

2. Postprocessing.

```
read obj src headers <$*.td; echo "$$headers" >$*.d
```

The purpose of this shell code is to extract header files from the generated
.td file and store this list of headers files to a .d file.

The resultant .d file contains a space separated list of headers files, all on
one line.

E.g.

```
api.h stddef.h stdlib.h stdio.h...
```

Postprocessing in this example uses bash code and handles the gcc format of dep
files.

Some other compilers (e.g. xlc, sun cc) generate dep files in
one-prerequisite-per-rule format.

E.g.

```
api.o: api.c
api.o: api.h
api.o: stddef.h
api.o: stdlib.h
api.o: stdio.h
...
```

To postprocess dep files of this format the following code can be used.
This example contains sun cc specific compiler options.

```
SHELL=/bin/bash
.ONESHELL:
%.o: %.c %.d $$(file <%.d)
	$(CC) $(CPPFLAGS) $(CFLAGS) -xMD -xMF $*.td -o $@ -c $<
	{
		while :
		do
		  IFS=: read obj dep
		  status=$$?
		  [[ $$dep =~ \.h$$ ]] && echo -n "$$dep"
		  [[ $$status -ne 0 ]] && break
		done
		echo
	} <$*.td >$*.d
	touch -c $@
```

3. $(file) appends the contents of %.d file to the list of prerequisites.


4. Second expansion ensures $$(file) is expanded only when this rule is used to build
the current target. This is the magic maker here.

This allows make to perform expensive work only for those targets which are
being built.


```
.SECONDEXPANSION:
hello.tsk: $$(file <hello.d); $(info $@ from $^)
bye.tsk: $$(file <bye.d); $(info $@ from $^)
clean:; -rm -f hello.tsk bye.tsk
```

Assuming there are files hello.d and bye.d which contain the prerequisites of
hello.tsk and bye.tsk respectively,
```
$ make hello.tsk
```
will second expand `` $$(file <hello.d) `` and will not second expand ``
$$(file <bye.d). ``
```
$ make clean
```
will not second expand `` $$(file <hello.d) `` or `` $$(file <bye.d) ``.


Before version 4.4 GNU Make would avoid second expansion of the prerequisites
of unrelated targets only in the case of implicit rules.  In the case of
explicit and static pattern rules GNU Make avoids second expansion of the
prerequisites of unrelated targets starting with version 4.4.

See https://savannah.gnu.org/bugs/?62706.


### .NOTINTERMEIDATE.

There is still one missing piece in this makefile. Make considers generated dep
files and all headers files to be intermediate.  We need a mechanism to tell
make that all files which match %.d and %.h are not to be treated as
intermediate.


So, we introduce special target .NOTINTERMEDIATE and our makefile becomes


```
.NOTINTERMEDIATE: %.d %.h
.SECONDEXPANSION: %.o

%.o: %.c %.d $$(file <%.d)
	gcc $(CPPFLAGS) $(CFLAGS) -MD -MF $*.td -o $@ -c $<
	read obj src headers <$*.td; echo "$$headers" >$*.d
	touch -c $@

%.d: ;
%.h: ;
```


This makefile solves all the above described issues of unconditional include.


An additional bonus is that $(file) is faster than include. When the file
exists, include parses the file and evals its contents, and when the file is
missing include searches a list of directories. $(file) does none of that.


### Another use case for .NOTINTERMEDIATE.

gcc has option -MP to generate an explicit target for each header file.

There are compilers which do not have such an option.

```
.NOTINTERMEDIATE: %.h
```

can be used to mark header files not intermediate with those compilers.


### $$*.d vs %.d.

With the fix from https://savannah.gnu.org/bugs/?60188 it is possible to make
dep files and header files explicit in certain cases with


```
%.o: %.c $$*.d $$(file <$$*.d)
```

However, this workaround loses when the object file is not located in the
current directory. With %.d the implicit search algo prepends directory name to
the stem.


Another advantage of .NOTINTERMEDIATE over $$* is ability to be used with
built-in rules, if needed.


### References

[1] http://make.mad-scientist.net/papers/advanced-auto-dependency-generation.

[2] https://savannah.gnu.org/bugs/?62706.

[3] https://savannah.gnu.org/bugs/?60188.

### Author.

Dmitry Goncharov.
dgoncharov@users.sf.net.
2021.03.27.

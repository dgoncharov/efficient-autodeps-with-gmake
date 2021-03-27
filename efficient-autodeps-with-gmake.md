## Efficient autodeps with gmake.


The modern technique of tracking dependencies uses include directive.  While
this technique is infinitely superior to the manual maintenance of deps there
is still room for improvement.


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
api.o: api.c api.h <other header files>...\
    <more header files>...

```

2. Postprocessing.

```
read obj src headers <$*.td; echo "$$headers" >$*.d
```

is used to extract header files from the generated .td file and store this list of
headers files to a .d file.

The contents of the generated .d file is a space separated list of headers
files, all on one line.


3. $(file) appends the contents of %.d to the list of prerequisites.


4. Second expansion ensures $$(file) is expanded only when this rule is used to build
the current target. This is the magic maker here.



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



### Notes.

Postprocessing in this example uses bash code and handles the gcc format of dep
files. To handle the one-rule-per-line format, that other compilers use, read
can be run in a loop.


### Author.

Dmitry Goncharov.
dgoncharov@users.sf.net.
2021.03.27.

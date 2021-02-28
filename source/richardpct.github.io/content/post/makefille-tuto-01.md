---
title: "Makefile: Quick Start"
date: 2019-08-07T15:20:47Z
draft: true
---

<img src="https://raw.githubusercontent.com/richardpct/images/master/windmill01.png" style="max-width:20%;min-width:40px;float:right;" alt="windmill" />

## Introduction
I have been working since more than ten years as Linux System
Administrator/DevOps Engineer and I have noticed that the Make tool is not
widely used for building a project involving the managing of infrastructure
such as Docker, Terraform or miscellaneous automation scripts. In some cases,
it may be more efficient to use Makefile instead of Bash script.

## What is Make ?
Make is a tool that helps to automate some tasks, but what is the benefit of
using a Makefile against a shell script ?<br/>
For instance you might want to build a simple program in C and without to go
into details you may follow these rules:

* To compile a program you need the object file \*.o
* But to build an object \*.o you need the source file \*.c

Here is the dependency graph which is straightforward:<br/>
`program.c -> program.o -> program`

We can write the Makefile to describe these rules as follows:
```makefile
program: program.o
	gcc -o program program.o

program.o: program.c
	gcc -c program.c
```

A simple Make rule syntax is as follows:
```makefile
target_A: target_B
	commands

target_B: target_C
	commands
```

Whenever you run this Makefile, the program will be rebuilt only if program.o
is altered, and program.o will be rebuilt only if program.c is modified. But
how Make knows if program must be rebuilt when program.c is modified ? By
default Make compares the timestamp of each targets and their prerequisites, if
a target is older than their prerequisites, the target will be rebuilt.

## Make by example
I wrote a Bash script for installing some programs which are not installed on
my MacOS system such as Tmux.

### Bash script version
The following script can be found
[here](https://github.com/richardpct/macos-packages/blob/master/Install_all_go.sh).

```bash
#!/bin/bash

set -e -u

packages="libtool openssl autoconf automake libevent tmux tree"

if [ "$#" -ne 1 ]; then
  echo "[Error]: need parameter for install destination"
  exit 1
fi

dest=$1

for package in $packages; do
  go get -u github.com/richardpct/macos-${package}
  macos-${package} -destdir=${dest}
done
```

This script executes a loop over a list of packages, for each iteration I use
`go get` command to retrieve the source code written in Go language then I
compile and install it.<br/>
The drawback of this script is if I want to rebuild only one specific package,
then it will rebuild all packages because this Bash script doesn't handle
dependencies between the packages.

### Make script version
Here is the Makefile script version which is better than the Bash script and it
can be found [here](https://github.com/richardpct/macos-packages/blob/master/Makefile):

```makefile
.DEFAULT_GOAL := help
AWK           := /usr/bin/awk
DEST_DIR      ?= $(HOME)/test
GO            ?= $(HOME)/opt/go/bin/go
GO_BIN        ?= $(HOME)/go/bin
REPO          ?= github.com/richardpct
GO_SRC        := $(HOME)/go/src/$(REPO)
PACKAGES      := tree \
                 make \
                 htop \
                 tmux \
                 libevent \
                 libtool \
                 openssl \
                 automake \
                 autoconf
VPATH         := $(DEST_DIR)/bin $(foreach pkg,$(PACKAGES),$(GO_SRC)/macos-$(pkg))
vpath %.a $(DEST_DIR)/lib

# If default GO does not exist then looks for in PATH variable
ifeq "$(wildcard $(GO))" ""
  GO_FOUND := $(shell which go)
  GO = $(if $(GO_FOUND),$(GO_FOUND),$(error go is not found))
endif

# $(call install-package,package_name)
define install-package
  $(GO) get -u $(REPO)/macos-$1
  $(GO_BIN)/macos-$1 -destdir=$(DEST_DIR)
endef

.PHONY: help
help: ## Show help
	@echo "Usage: make [DEST_DIR=/tmp] TARGET\n"
	@echo "Targets:"
	@$(AWK) -F ":.* ##" '/.*:.*##/{ printf "%-13s%s\n", $$1, $$2 }' \
	$(MAKEFILE_LIST) \
	| grep -v AWK

.PHONY: all
all: $(PACKAGES) ## Build all packages

.PHONY: update_repo
update_repo: ## Update all repositories
	for pkg in $(PACKAGES); do \
	  $(GO) get -d $(REPO)/macos-$$pkg; \
	done

$(foreach pkg,$(PACKAGES),$(pkg).go): update_repo

tree: tree.go ## Build tree
	$(call install-package,$@)

make: make.go ## Build make
	$(call install-package,$@)

htop: htop.go ## Build htop
	$(call install-package,$@)

tmux: tmux.go libevent.a ## Build tmux
	$(call install-package,$@)

.PHONY: libevent
libevent: libevent.a ## Build libevent

libevent.a: libevent.go glibtool openssl automake
	$(call install-package,libevent)

.PHONY: libtool
libtool: glibtool ## Build libtool

glibtool: libtool.go
	$(call install-package,libtool)

openssl: openssl.go ## Build openssl
	$(call install-package,$@)

automake: automake.go autoconf ## Build automake
	$(call install-package,$@)

autoconf: autoconf.go ## Build autoconf
	$(call install-package,$@)
```

You may notice this Makefile is longer than the Bash script, because it handles
more features such as dependencies and checking.<br/>
Some packages have no prerequisites and can be built by itself such as tree,
make and htop.

Let's explain the following excerpt in detail:

```makefile
htop: htop.go ## Build htop
	$(call install-package,$@)
```

htop is the target representing the program we want to build from the source
file htop.go. Whether the htop binary does not exist or it is older than the
source code htop.go then the rules is executed.

The htop binary will be installed in the path provided by the DEST_DIR variable
(by default in $(HOME)/test/bin), the source files are retrieved from Github
then are copied in the path provided by the GOPATH environment variable, that
is in $(HOME)/go/src/$(REPO) by default. If these two paths are not located in
the same directory as the Makefile, how Make looks for the htop and htop.go
files in the right directory ? If it does not find htop and htop.go in the
current directory then it looks for them in the directories defined by the
VPATH variable, it is similar to the PATH variable in Bash shell.

Here is the excerpt that illustrates the definition of VPATH:

```makefile
...
DEST_DIR ?= $(HOME)/test
...
REPO     ?= github.com/richardpct
GO_SRC   := $(HOME)/go/src/$(REPO)
PACKAGES := tree \
            make \
            htop \
            tmux \
            libevent \
            libtool \
            openssl \
            automake \
            autoconf
VPATH    := $(DEST_DIR)/bin $(foreach pkg,$(PACKAGES),$(GO_SRC)/macos-$(pkg))
...
```

As you can see, to create a variable in a Makefile the syntax is as follows:

```makefile
VAR1  = value1
VAR2 := value2
VAR3 ?= value3
```

There are 3 types of assignment for creating a variable, `=` is for recursive
variable, `:=` is for simple variable and `?=` affects the value if the
variable was not defined.

A VPATH variable may contain multiple values separated by a space, I use a loop
using the foreach function to range the list of packages provided by the
PACKAGES variable, these two following assignments yields the same result:

```makefile
VPATH := $(DEST_DIR)/bin $(foreach pkg,$(PACKAGES),$(GO_SRC)/macos-$(pkg))
```

is equivalent to:

```makefile
VPATH := $(HOME)/test/bin \
         $(HOME)/go/src/github.com/richardpct/macos-tree \
         $(HOME)/go/src/github.com/richardpct/macos-make \
         $(HOME)/go/src/github.com/richardpct/macos-htop \
         $(HOME)/go/src/github.com/richardpct/macos-tmux \
         $(HOME)/go/src/github.com/richardpct/macos-libevent \
         $(HOME)/go/src/github.com/richardpct/macos-libtool \
         $(HOME)/go/src/github.com/richardpct/macos-openssl \
         $(HOME)/go/src/github.com/richardpct/macos-automake \
         $(HOME)/go/src/github.com/richardpct/macos-autoconf
```

As you can see the first one is more concise.  

Let's go back to the rule, the command `$(call install-package,$@)` calls a
function, a function is defined as follows:

```makefile
# $(call install-package,package_name)
define install-package
  $(GO) get -u $(REPO)/macos-$1
  $(GO_BIN)/macos-$1 -destdir=$(DEST_DIR)
endef
```

$1 is an argument of this function, we can call this function as follows:

```makefile
	$(call install-package,$@)
```

$@ is the value of the argument and its syntax is a shorcut that contains
the target, that is htop.

Now I explain a rule more complex:

```makefile
libevent.a: libevent.go glibtool openssl automake
	$(call install-package,libevent)
```

To build the target libevent.a, Make looks for the source code libevent.go, and
also build the targets glibtool, openssl and automake if it is necessary.<br/>
libevent.a is built whether libevent.go, glibtool, openssl or automake are
newer than the libevent.a file.

Notice that the file libevent.a is not seeked in VPATH variable because I have
defined a specific vpath variable for the file type \*.a as follows:

```makefile
vpath %.a $(DEST_DIR)/lib
```

When I use an external program such as Go, Docker or Terraform, I am used to
set a dedicated variable with its full path as follows:

```makefile
...
GO ?= $(HOME)/opt/go/bin/go
...
# If default GO does not exist then looks for in PATH variable
ifeq "$(wildcard $(GO))" ""
  GO_FOUND := $(shell which go)
  GO = $(if $(GO_FOUND),$(GO_FOUND),$(error go is not found))
endif
```

To make this Makefile more portable, I check if the path provided by the value
of GO variable exists, otherwise I look for the go command in the PATH
environment variable, if it is not found, Make stops and return an error.

I would remind you that it is very important to write a documentation or a man
page of your own programs. There is a trick to automatically create a man page
as follows:

```makefile
...
.PHONY: help
help: ## Show help
	@echo "Usage: make [DEST_DIR=/tmp] TARGET\n"
	@echo "Targets:"
	@$(AWK) -F ":.* ##" '/.*:.*##/{ printf "%-13s%s\n", $$1, $$2 }' \
	$(MAKEFILE_LIST) \
	| grep -v AWK
...
tree: tree.go ## Build tree
...
htop: htop.go ## Build htop
...
```

For each rules I add a description prefixed with a `##` on the same line as the
target.<br/>
When you run `make help`, the help rule looks for comments which are on the
same line of the targets then display them.

You may be wondering what means this line:

```makefile
.PHONY: help
```

By default a target represents a file on the filesystem and Make excecutes the
rules depending to the file status and their prerequisites, but in some cases
we just need to excecute a rule without condition each time we call it.<br/>
`.PHONY: target` tells to Make that the target is not a file, and the rule
associated to it is always excecuted.

A common usage of .PHONY is to create a target that calls all the targets, this
target is generally named `all` as follows:

```makefile
.PHONY: all
all: $(PACKAGES) ## Build all packages
```

The command `make all` will build all packages.

I also use .PHONY when I want to create an alias to a target as follows:

```makefile
.PHONY: libevent
libevent: libevent.a ## Build libevent
```

I can now install libevent by running:

    $ make libevent

instead of:

    $ make libevent.a

Now, let's try my Makefile to show you the power of Make!

Retrieve the source then install all packages:

    $ git clone https://github.com/richardpct/macos-packages
    $ cd macos-packages
    $ make all

If you run `make all` again, none of the packages will be rebuilt because Make
rebuilds only if it is necessary.

Let's do a last test, remove libevent.a then run `make all` again:

    $ rm ~/test/lib/libevent.a
    $ cd macos-packages
    $ make all

Make will rebuild libevent then tmux packages because tmux depends on libevent
, this is the power of Make, it rebuilds only if necessary.

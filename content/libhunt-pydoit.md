Title: Lib recommendation - pydoit
Date:Fri 27 Jan 2023 13:07:33 UTC

Tags: programming, python
Category: Programming


# Introduction

Currently in the workflow/automation tool industry(docker wrappers, Ansible, CI tools,) there seems to be quite a lot of sophistication.

This kind of sophistication is not bad per-se, As long as it fit the aim of what you do, but in many cases it have the pigeonhole effect where you fit your solution to the tool.

In addition it is really hard to find the right platform in such condition as you only reach the real and important limitations where it hurts the most and after you are far too deep into the ecosystem the tool provide.

In order to combat it I've recently looked for alternatives that are more minimalist, which means you own more of the tooling, but also that you can modify them more readily and in the search I've found a nice library called [pydoit](https://pydoit.org/) which I would like to highlight that can offer solution in case the need is more lightweight.

# What it is

Pydoit is a library/tool that allow you to write your usual workflow needs in straight python. It works by extracting all the function in the modules that start with *task_* and running them by order of which he get to them.

# Strength
1. seamless integration of python functions, as it is basically running a python module.
1. shell commands
1. Reporter abstraction - each task accumulate to internal data model from which you can extract a report.
1. JSON reporter seems to do the work just fine
1. Verasity tracking of the commands - handling if command succeeded or failed.(In case of python based if it returned *True* or *False* and in shell based on non-zero exit)
1. Makefile-like target based tooling

# Use cases

1. A set of complex tasks that are too complex for shell scripts but also need to be portable
1. Integrated runner
1. Integrating more tightly the workflow engine to a complex application that compiled to native(using bindings) or have heavy use of python
1. Simple workflow engine for small on the fly runner with more programmable access

# Limitation/ambiguous

1. Documentation is not the best However it is relatively small code base which I've found to be pretty hackable and easy to understand.
1. A lot of basic thing you usually use naturally in a big ecosystem such as Ansible are now in your hand - This is both a strength but also come with a cost since all the R&D cost and bugs are now from your side rather than from a community and open-source collaboration[^1]
1. Sometimes unexpected in how the workflow code works due to the reflective methodology - For example if you run a function outside task it will be executed upon loading the module, and sometimes it is not as obvious how to insert your own function.


# How does a pydoit project can look like

During the discovery phase I've found the tooling to be quite versatile and here are few deployment ideas[^2]

## Using the CLI tool

Pydoit by default have a CLI tool which is built using a modular python class.

In terms of folder structure you need to have a dodo.py with the workflow module and run the doit executable wrapper

In such context I will also prefer to use it with some Makefile or a shell script just to tidy up unexpected results and have easy expandability when it is needed to create more complex interaction with other tools

### Folder structure(Plain + INI config)
```
├── dodo.py
├── doit.cfg
└── Makefile
```

### [Code(taken from the official documentation)](https://pydoit.org/tasks.html)
```
def task_hello():
    """hello"""

    def python_hello(targets):
        with open(targets[0], "a") as output:
            output.write("Python says Hello World!!!\n")

    return {
        'actions': [python_hello],
        'targets': ["hello.txt"],
        }
```

### [doit.cfg](https://pydoit.org/configuration.html#doit-cfg)

```
[GLOBAL]
backend = json
reporter = json

[list]
status = True
subtasks = True
```

### Makefile[^4]
```
DOIT=$(shell command -v doit)
JQ=$(shell command -v jq)

RESULTS=hello.txt __pycache__

.PHONY: run clean
all: run

run:
	${DOIT} | ${JQ}

clean:
	${RM} -r ${RESULTS}
```



### Result
```
<fullpath>/doit | /usr/bin/jq
{
  "tasks": [
    {
      "name": "hello",
      "result": "success",
      "out": "",
      "err": "",
      "error": null,
      "started": "2023-02-05 20:35:56.767689",
      "elapsed": 7.319450378417969e-05
    }
  ],
  "out": "",
  "err": ""
}

```


## doit Executable + Embedded config

### Folder structure(Plain + INI config)
```
├── dodo.py
└── Makefile
```

### [Code(taken from the official documentation)](https://pydoit.org/tasks.html)
```
DOIT_CONFIG = {
    'backend': 'json',
    'reporter': 'json'
}


def task_hello():
    """hello"""

    def python_hello(targets):
        with open(targets[0], "a") as output:
            output.write("Python says Hello World!!!\n")

    return {
        'actions': [python_hello],
        'targets': ["hello.txt"],
        }
```


### Makefile
```
DOIT=$(shell command -v doit)
JQ=$(shell command -v jq)

RESULTS=hello.txt __pycache__

.PHONY: run clean
all: run

run:
	${DOIT} | ${JQ}

clean:
	${RM} -r ${RESULTS}
```



### Result
```
<fullpath>/doit | /usr/bin/jq
{
  "tasks": [
    {
      "name": "hello",
      "result": "success",
      "out": "",
      "err": "",
      "error": null,
      "started": "2023-02-05 20:35:56.767689",
      "elapsed": 7.319450378417969e-05
    }
  ],
  "out": "",
  "err": ""
}

```


## Overriding the runner

The runner is a modular aspect of the system and as such can be implemented in python in an integrated manner[^3]


### Folder structure
```
├── Makefile
├── run.py
└── workflows
    └── my_workflow.py
```

### Code(taken from the [official documentation](https://pydoit.org/tasks.html) and [this](https://pydoit.org/configuration.html#configuration-at-dodo-py)
```
DOIT_CONFIG = {
    'backend': 'json',
    'reporter': 'json'
}


def task_hello():
    """hello"""

    def python_hello(targets):
        with open(targets[0], "a") as output:
            output.write("Python says Hello World!!!\n")

    return {
        'actions': [python_hello],
        'targets': ["hello.txt"],
        }
```

### run.py
```
#! /usr/bin/env python3

import sys

from doit.cmd_base import ModuleTaskLoader
from doit.doit_cmd import DoitMain

if __name__ == "__main__":
    from workflows import my_workflow
    sys.exit(DoitMain(ModuleTaskLoader(my_workflow)).run(sys.argv[1:]))
```

### Makefile
```
DOIT=./run.py
PYTHON=$(shell command -v python3)
JQ=$(shell command -v jq)


RESULTS=hello.txt __pycache__

.PHONY: run clean deps

all: run

deps:
	ifeq ($(strip $(PYTHON)),)
	$(error "python3" not found)
	endif

	ifeq ($(strip $(JQ )),)
	$(error "jq" not found)
	endif

run:
	${PYTHON} ${DOIT} | ${JQ}

clean:
	${RM} -r ${RESULTS}
```

### Result
```
/usr/bin/python3 ./run.py | /usr/bin/jq
{
  "tasks": [
    {
      "name": "hello",
      "result": "success",
      "out": "",
      "err": "",
      "error": null,
      "started": "2023-02-05 20:35:56.767689",
      "elapsed": 7.319450378417969e-05
    }
  ],
  "out": "",
  "err": ""
}

```

# Summary

I hope the article opened some easier on-boarding and open ideas for newer solutions




[^1]: Although there isn't a strict ecosystem like in the case of ansible you still have the pythonic ecosystem and it is possible foreign bindings
[^2]: The code examples are given with free of usage, copying and redistributing
[^3]: This is not strictly necessary but just a personal practice I've found useful. Alternatively you can just run `doit | jq` in the folder
[^4]: the following structure is just for the pydoit aspect and can be extended as needed and the norms of the particular codebase

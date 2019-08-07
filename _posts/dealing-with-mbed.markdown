---
layout: post
title:  "Living with mbed-cli"
date:   2019-08-06
categories:
---

MBED is neat but the mbed-cli tool is a pain (on Linux at least). 
This post is about how I get mbed-cli to work on my system.

# install virtualenv

Can't remember how I did this, most likely simply followed an up to date guide. Mbed-cli at
time of writing requires Python 2.7.

# create an app folder

~~~
mkdir my_app
cd my_app
~~~

# create virtualenv within app folder

~~~
virualenv my_env --no-site-packages
~~~

I found I had to use --no-site-packages because pip (later) was finding
some of the dependencies elsewhere on my system and not installing them
in the virtualenv, but then python doesn't find them when it runs.

# activate the environment

virtualenv seems to need to be activated/deactivates, there is no "exec" type command.

~~~
source my_env/bin/activate
~~~

# install mbed-cli in the environment

~~~
pip install mbed-cli
~~~

# init the project

~~~
mbed-cli new .
~~~

# install mbed's dependencies

~~~
cd mbed-os
pip install -r requirements.txt
cd -
~~~

This step might happen first time you "mbed-cli compile" since I seem
to be experiencing an "auto-install" delay each time I run that command.

The auto-install thing never seems to succeed so there is a painful delay
each time you use the tool.

# do stuff

Like compile:

~~~
mbed-cli compile -t GCC_ARM -m NUCLEO_F429ZI
~~~

# remember to deactivate the virtualenv when you are done

~~~
deactivate
~~~

# I find it useful to use shell scripts for common commands

For example, you can activate/deactive the virtualenv for each 
command you wish to perform. 

e.g.

~~~
#!/usr/bin/env bash

source mbed-cli/bin/activate

mbed-cli compile -t GCC_ARM -m NUCLEO_F429ZI "$@"

deactivate
~~~

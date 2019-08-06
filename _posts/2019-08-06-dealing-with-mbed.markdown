---
layout: post
title:  "Living with mbed-cli"
date:   2019-08-06
categories:
---

MBED is neat but the mbed-cli tool is a pain (on Linux at least). 
This post is about how I get mbed-cli to work on my system.

# Install Virtualenv and Python

mbed-cli at time of writing requires Python 2.7.

# Create an app folder

{% highlight terminal %}
mkdir my_app
cd my_app
{% endhighlight %}

# Create virtualenv within app folder

{% highlight terminal %}
virualenv my_env --no-site-packages
{% endhighlight %}

I found I had to use --no-site-packages because pip (later) was finding
some of the dependencies elsewhere on my system and not installing them
in the virtualenv, but then python doesn't find them when it runs.

# Activate the environment

virtualenv seems to need to be activated/deactivates, there is no "exec" type command.

{% highlight terminal %}
source my_env/bin/activate
{% endhighlight %}

# Install mbed-cli in the environment

{% highlight terminal %}
pip install mbed-cli
{% endhighlight %}

# Init the project

{% highlight terminal %}
mbed-cli new .
{% endhighlight %}

# Install mbed's dependencies

{% highlight terminal %}
cd mbed-os
pip install -r requirements.txt
cd -
{% endhighlight %}

This step might happen first time you "mbed-cli compile" since I seem
to be experiencing an "auto-install" delay each time I run that command.

The auto-install thing never seems to succeed so there is a painful delay
each time you use the tool.

# Do stuff

Like compile:

{% highlight terminal %}
mbed-cli compile -t GCC_ARM -m NUCLEO_F429ZI
{% endhighlight %}

# Remember to deactivate the virtualenv when you are done

{% highlight terminal %}
deactivate
{% endhighlight %}

# I find it useful to use shell scripts for common commands

For example, you can activate/deactive the virtualenv for each 
command you wish to perform. 

e.g.

{% highlight terminal %}
#!/usr/bin/env bash

source mbed-cli/bin/activate

mbed-cli compile -t GCC_ARM -m NUCLEO_F429ZI "$@"

deactivate
{% endhighlight %}

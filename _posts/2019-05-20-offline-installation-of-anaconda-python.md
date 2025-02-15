---
title:  "Offline Installation of Anaconda Python"
seo_title: "offline installation of anaconda python"
seo_description: "offline installation of anaconda python"
date:   2019-05-20 00:00:00 +0700
categories:
  - Programming
tags:
  - Python
excerpt: "My daily workloads consist of data analytics stuffs like report generation automation and capturing node performance statistics, and some NetOps automation....."
---
### The Problem
My daily workloads consist of data analytics stuffs like report generation automation and capturing node performance statistics, and some NetOps automation (configuration management, file transfer, CLI adhoc check). I was given a bare metal server with plain CentOS 6 as its operating system, without internet connectivity.

Offline installation is a way to go, but I had to wrestle with all the dependencies (in my experience, I had to install from a recompiled Python source with `zlib` enabled, and then finding all required packages to enable the `FORTRAN 77` compiler).

### The Solution
There is actually an all-in-one installation bundle to serve that purpose. As taken from [this link](https://docs.scipy.org/doc/numpy-1.10.1/user/install.html).

> In most use cases the best way to install NumPy on your system is by using an installable binary package for your operating system. Good solutions for Windows are, Enthought Canopy, Anaconda (which both provide binary installers for Windows, OS X and Linux) and Python (x, y). Both of these packages include Python, NumPy and many additional packages.

I chose Anaconda since it’s the one that most people are working with. In this short post, I will share my experience installing Anaconda Python, the offline way.

### The Steps
* Download Anaconda Python installer Tarball from [their website](https://www.anaconda.com/distribution/). Choose the appropriate installer for your OS (mine was : "Python3.7 64-Bit Command Line Installer").

* Verify the data integrity of the Anaconda installer files and check the output to be sure it matches the checksums provided by [Anaconda](https://docs.anaconda.com/anaconda/install/hashes/).For example:

{% highlight bash %}
$ md5sum Anaconda3-2019.03-Linux-x86_64.sh
43caea3d726779843f130a7fb2d380a2  Anaconda3-2019.03-Linux-x86_64.sh
{% endhighlight %}

* Execute the installation script.
Choose "yes" to accept end-user agreement and Conda's environment initialization. Exit the shell and re-login afterwards to see the changes:

{% highlight bash %}
# can be executed as non-root
$ bash Anaconda3–2019.03-Linux-x86_64.sh
# your prompt will look like this after relogin
(base) [gandhi@devserver ~]$
{% endhighlight %}

* Make sure that PYTHONPATH is not set. PYTHONPATH is used by the Python interpreter to determine which modules to load. This is to make sure that your system is using the Anaconda's version of Python instead of the system's one:

{% highlight bash %}
# check PYTHONPATH
$ echo $PYTHONPATH

# unset PYTHONPATH in current shell
$ unset PYTHONPATH
{% endhighlight %}

### Installing Paramiko using Conda's Distribution
Anaconda uses its own packages, so it will not use packages installed in your previous Python installation. Suppose we want to install paramiko, then :

* Search and download paramiko Tarball from [Anaconda package repo](https://anaconda.org/anaconda/repo).

* Check the hash key and make sure it matches with the one shown in Anaconda repository and then install the package:

{% highlight bash %}
# install using conda
$ conda install paramiko-2.4.2-py37_0.tar.bz2
{% endhighlight %}

* Paramiko needs the following dependencies, so also download these from Conda repo:

{% highlight bash %}
pyasn1–0.4.5-py_0.tar.bz2
bcrypt-3.1.6-py37h7b6447c_0.tar.bz2
pynacl-1.3.0-py36h7b6447c_0.tar.bz2
{% endhighlight %}

* If somehow after installing Conda’s paramiko you still get the `ModuleNotFoundError`, then check the presence of paramiko's `site-packages` under your Anaconda's Python site-packages directory. If they do not exists, simply just copy them:

{% highlight bash %}
$ cp -R /usr/local/anaconda3/pkgs/paramiko-2.4.2-py37_0/lib/site-packages /usr/local/anaconda3/lib/python3.7/site-packages/
{% endhighlight %}

* Now you should be able to load `paramiko` in Anaconda's Python:

{% highlight bash %}
(base) [gandhi@devserver ~]$ python3
Python 3.7.3 (default, Mar 27 2019, 22:11:17)
[GCC 7.3.0] :: Anaconda, Inc. on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import paramiko
>>>
{% endhighlight %}
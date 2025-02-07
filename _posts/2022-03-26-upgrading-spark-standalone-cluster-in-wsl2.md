---
title:  "Upgrading Spark Standalone Cluster in WSL2"
seo_title: "upgrading spark standalone cluster in wsl2"
seo_description: "upgrading spark standalone cluster in wsl2"
date:   2022-03-26 00:00:00 +0700
categories:
  - Programming
tags:
  - Linux
excerpt: "Since 2019 I've been using PySpark 2.4.4 for local big data development, and my system's version of Python had moved from 3.6 into 3.8...."
---
### The Problem
Since 2019 I've been using PySpark 2.4.4 for my local big data development, and my system's Python had migrated from 3.6 into 3.8. This version drift caused me some headache when it throws the following error during the initialisation of `SparkSession` object from pyspark.sql module in my code:

{% highlight bash %}
TypeError: an integer is required (got type bytes)
{% endhighlight %}

This error was caused because PySpark version 2.4.4 does not support Python 3.8. Most of the recommendations i've found on the internet are telling me to downgrade to Python 3.7 or to upgrade Pyspark to the later version to work around the issue, for example by running `pip3 install --upgrade pyspark`.

I am using a Spark standalone cluster in my local i.e. "installing from source"-way, and the above command did nothing to my PySpark installation i.e. the version stays at `2.4.4`. There are more steps needed to be taken.

Furthermore, my WSL2 is a spaghetti-maze installation of binaries / distributions. Some challenges are:
1. There are binaries that were installed by brew and some that came by means of `sudo apt-get`
2. I forgot how I installed PySpark and Apache-Spark in the first place

Obviously doing `brew uninstall pyspark` and `apt-get remove apache-spark` did nothing. To start from a clean slate is not as easy as uninstall-and-reinstall.

### The Solution
First, I need to locate the installation path of both PySpark and Spark-Shell:

{% highlight bash %}
$ which pyspark && which spark-shell
/home/linuxbrew/.linuxbrew/bin/pyspark
/home/linuxbrew/.linuxbrew/bin/spark-shell
{% endhighlight %}

Those files are actually bash scripts used by brew to launch the application. Looking at the code of `/home/linuxbrew/.linuxbrew/bin/pyspark` we can see the following `PATH` definition:

{% highlight bash %}
# Add the PySpark classes to the Python path:
export PYTHONPATH="${SPARK_HOME}/python/:$PYTHONPATH"
export PYTHONPATH="${SPARK_HOME}/python/lib/py4j-0.10.9.2-src.zip:$PYTHONPATH"
{% endhighlight %}

Then, I need to locate which path `$SPARK_HOME` resolves to:

{% highlight bash %}
$ echo $SPARK_HOME
/opt/spark
{% endhighlight %}

I went to `/opt` and voila! That's where it was installed. What have to be done next to "upgrade" spark version is to delete the folder containing all spark-related binaries and replace it with the newer one (make sure the hadoop version tallies with your installation - I am still using `2.7` at the time of writing):

{% highlight bash %}
rm -rf /opt/spark
sudo wget https://www.apache.org/dyn/closer.lua/spark/spark-3.2.1/spark-3.2.1-bin-hadoop2.7.tgz
sudo tar xvzf spark-3.2.1-bin-hadoop2.7.tgz
sudo mv spark-3.2.1-bin-hadoop2.7 spark
{% endhighlight %}

After the above steps have been done, check your PySpark version:

{% highlight bash %}
$ pyspark --version
22/03/26 15:51:40 WARN Utils: Your hostname, PMIIDIDNL13144 resolves to a loopback address: 127.0.1.1; using 172.17.50.37 instead (on interface eth0)22/03/26 15:51:40 WARN Utils: Set SPARK_LOCAL_IP if you need to bind to another address
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.2.1
      /_/

Using Scala version 2.12.15, OpenJDK 64-Bit Server VM, 1.8.0_292
Branch HEAD
Compiled by user hgao on 2022-01-20T20:15:47Z
Revision 4f25b3f71238a00508a356591553f2dfa89f8290
Url https://github.com/apache/spark
Type --help for more information.
{% endhighlight %}

### Takeaways

1. Always be consistent with your package management. Stick to brew or apt-get but don't mix both.
2. Try to use venv to manage Python packages for different projects. Actually. this is one advice that I've heard so many times but at the moment it feels like it's too late to be implemented in my current setup.
---
title:  "Multiple Remote Repositories with Git"
seo_title: "using multiple remote repositories with git"
seo_description: "using multiple remote repositories with git"
date:   2024-08-22 00:00:00 +0700
categories:
  - Programming
tags:
  - Git
excerpt: "Sometimes, we want to push update of our projects to multiple repositories, in different SCM. In this post, I am sharing one of the ways to do that..."
toc: true
toc_label: "Table of Contents"
---

## The Problem

Sometimes, we want to push update of our projects to multiple repositories, in different SCM. In this post, I am sharing one of the ways to do that.

Check your remote repository info. In this example I have a project pushed to Github:

{% highlight bash %}
$ git remote -v show
origin  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (fetch)
origin  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (push)
{% endhighlight %}

If you check your git configuration (`PROJECT_NAME/.git/config`), you will see:

{% highlight bash %}
$ git remote -v show
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://github.com/GandhiNN/iot-vernemq-ktig-stack.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "dev"]
        remote = origin
        merge = refs/heads/dev
[http]
        sslverify = false
{% endhighlight %}

Now suppose you want to add a remote git repository on your Bitbucket project:

{% highlight bash %}
$ git remote set-url origin --add https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git
{% endhighlight %}

## The Solution

To be able to do controlled Git workflow (e.g. just pull from Github remote or just push to Bitbucket remote), we add remote repository information using git remote:

{% highlight bash %}
$ git remote add bitbucket https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git

$ git remote add github https://github.com/GandhiNN/iot-vernemq-ktig-stack.git
{% endhighlight %}

Now our project’s Git configuration contains the information of the new repositories just added:

{% highlight bash %}
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://github.com/GandhiNN/iot-vernemq-ktig-stack.git
        fetch = +refs/heads/*:refs/remotes/origin/*
        url = https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git
[branch "dev"]
        remote = origin
        merge = refs/heads/dev
[http]
        sslverify = false
[remote "bitbucket"]
        url = https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git
        fetch = +refs/heads/*:refs/remotes/bitbucket/*
[remote "github"]
        url = https://github.com/GandhiNN/iot-vernemq-ktig-stack.git
        fetch = +refs/heads/*:refs/remotes/github/*
{% endhighlight %}

In the sequence of “origin” block, Github remote takes precedence over the Bitbucket one; If you run git pull, it will still pull from only Github. If you want to make Bitbucket as the first source, you can just rearrange the sequence of “url” :

{% highlight bash %}
$ git remote -v
bitbucket       https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (fetch)
bitbucket       https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (push)
github  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (fetch)
github  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (push)
origin  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (fetch)
origin  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (push)
origin  https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (push)
{% endhighlight %}

If you want to push your updates to the Bitbucket remote, you can run `git push -u bitbucket <branch_name>`:

{% highlight bash %}
$ git push -u bitbucket master
Username for 'https://<bitbucket_url>': <bitbucket_username>
Password for 'https://<bitbucket_username>@<domain>@<bitbucket_url>':
Enumerating objects: 32, done.
Counting objects: 100% (32/32), done.
Delta compression using up to 8 threads
Compressing objects: 100% (26/26), done.
Writing objects: 100% (32/32), 12.85 KiB | 4.28 MiB/s, done.
Total 32 (delta 1), reused 0 (delta 0)
{% endhighlight %}

You can do the same with your Github target:

{% highlight bash %}
$ git push -u github master
Username for 'https://github.com': GandhiNN
Password for 'https://GandhiNN@github.com':
Enumerating objects: 32, done.
Counting objects: 100% (32/32), done.
Delta compression using up to 8 threads
Compressing objects: 100% (26/26), done.
Writing objects: 100% (32/32), 12.85 KiB | 4.28 MiB/s, done.
Total 32 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), done.
{% endhighlight %}

Let’s say that you want to set Bitbucket as the default remote for your git push / git pull activities, you need to add the Bitbucket URL as the remote “origin”:

{% highlight bash %}
$ git remote add origin https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git
{% endhighlight %}

{% highlight bash %}
$ git remote -v
bitbucket       https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (fetch)
bitbucket       https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (push)
github  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (fetch)
github  https://github.com/GandhiNN/iot-vernemq-ktig-stack.git (push)
origin  https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (fetch)
origin  https://<bitbucket_url>/scm/<bitbucket_project_name>/iot-vernemq-kafka-poc-stack.git (push)
{% endhighlight %}

## Notes

To skip SSL verification in your git workflow on your project you can use:

{% highlight bash %}
$ git config --local http.sslverify false
{% endhighlight %}

To save credentials used by your git workflow on your project you can use:

{% highlight bash %}
$ git config --local credential.helper store
{% endhighlight %}

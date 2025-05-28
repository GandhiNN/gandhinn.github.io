---
title:  "Docker Pull and Self-Signed Certificate"
seo_title: "docker pull and self-signed certificate"
seo_description: "docker pull and self-signed certificate"
date:   2025-05-27 00:00:00 +0700
categories:
  - Programming
tags:
  - Docker
  - Linux
excerpt: "Using Docker Pull behind a transparent proxy could be PITA"
toc: true
toc_label: "Table of Contents"
---
# Overview
TBC

# Problem
I was trying to pull a docker image from a docker registry but encountered the following error:

{% highlight bash %}
Error response from daemon: Get <docker registry>/v1/_ping: x509: certificate signed by unknown authority
{% endhighlight %}

# How Docker Pull Works
TBC

# Configure Insecure Docker Image Registries

The solution is to add registry domain in Docker's `insecure-registries`.

Edit or create the file `/etc/docker/daemon.json` and add the following:

{% highlight bash %}
{
        "insecure-registries": ["gcr.io", "ghcr.io"],
        "dns": ["8.8.8.8"]
}
{% endhighlight %}

Then, restart the docker daemon.

# Conclusion
<TBC>

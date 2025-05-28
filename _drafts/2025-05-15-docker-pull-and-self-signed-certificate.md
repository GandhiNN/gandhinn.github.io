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
ERROR: failed to solve: ghcr.io/cargo-lambda/cargo-lambda:latest: failed to resolve source metadata for ghcr.io/cargo-lambda/cargo-lambda:latest: failed to copy: httpReadSeeker: failed open: failed to do request: Get "https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:34556ae1a303d3cd9ef66342d829b8dc7aca21f19ab5eacdafcfff6d07219828?se=2025-05-28T07%3A40%3A00Z&sig=ro6MWL1mq0IYPeFjh%2FOd%2FULt4bKPXT9bYdX6%2FmhTSgs%3D&sp=r&spr=https&sr=b&sv=2019-12-12": tls: failed to verify certificate: x509: certificate signed by unknown authority
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

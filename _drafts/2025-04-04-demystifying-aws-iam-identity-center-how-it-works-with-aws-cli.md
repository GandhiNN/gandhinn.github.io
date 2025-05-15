---
title:  "Docker Pull and Self-Signed Certificate"
seo_title: "docker pull and self-signed certificate"
seo_description: "docker pull and self-signed certificate"
date:   2025-05-15 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
excerpt: "This post is about the workaround when pulling docker images using self-signed certificate..."
toc: true
toc_label: "Table of Contents"
---
# Overview

When running a Dockerfile, I encountered the following error logs:

{% highlight bash %}
+ cargo clean
     Removed 0 files
+ docker build -t rust-lambda .
[+] Building 24.7s (2/2) FINISHED                                                                                                                        docker:default
 => [internal] load build definition from Dockerfile                                                                                                               0.0s
 => => transferring dockerfile: 309B                                                                                                                               0.0s
 => ERROR [internal] load metadata for ghcr.io/cargo-lambda/cargo-lambda:latest                                                                                   24.6s
------
 > [internal] load metadata for ghcr.io/cargo-lambda/cargo-lambda:latest:
------
Dockerfile:2
--------------------
   1 |     # Use the cargo-lambda image
   2 | >>> FROM ghcr.io/cargo-lambda/cargo-lambda:latest
   3 |     
   4 |     # Create a directory for our application
--------------------
ERROR: failed to solve: ghcr.io/cargo-lambda/cargo-lambda:latest: failed to resolve source metadata for ghcr.io/cargo-lambda/cargo-lambda:latest: failed to copy: httpReadSeeker: failed open: failed to do request: Get "https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:34556ae1a303d3cd9ef66342d829b8dc7aca21f19ab5eacdafcfff6d07219828?se=2025-05-15T08%3A50%3A00Z&sig=8Jbz7Oc760Q0gXlrAPR2eO3D1mLZycBDJu07B4swcoA%3D&sp=r&spr=https&sr=b&sv=2019-12-12": tls: failed to verify certificate: x509: certificate signed by unknown authority
{% endhighlight %}

# Root Cause
<TBC>

# Workaround.
Add the container repository domain in the list of insecure registries in `/etc/docker/daemon.json`:

{% highlight bash %}
{
        "insecure-registries": ["gcr.io", "ghcr.io"],
        "dns": ["8.8.8.8"]
}
{% endhighlight %}

Then restart the docker daemon

{% highlight bash %}
sudo service docker restart
{% endhighlight %}

# Conclusion
<TBC>

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
I was trying to pull a docker image from a docker registry but encountered the following error:

{% highlight bash %}
ERROR: failed to solve: ghcr.io/cargo-lambda/cargo-lambda:latest: failed to resolve source metadata for ghcr.io/cargo-lambda/cargo-lambda:latest: failed to copy: httpReadSeeker: failed open: failed to do request: Get "https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:34556ae1a303d3cd9ef66342d829b8dc7aca21f19ab5eacdafcfff6d07219828?se=2025-05-28T07%3A40%3A00Z&sig=ro6MWL1mq0IYPeFjh%2FOd%2FULt4bKPXT9bYdX6%2FmhTSgs%3D&sp=r&spr=https&sr=b&sv=2019-12-12": tls: failed to verify certificate: x509: certificate signed by unknown authority
{% endhighlight %}

# How Docker Pull Works
When you are pulling the latest version of an image from a Docker registry by executing `docker pull cargo-lambda`, then Docker daemon will do the following:
1. It identifies the `cargo-lambda` image and defaults to the "latest" tag (if the tag version is not specified)
2. The Docker daemon contacts the registry.
3. If authentication is required, the user shall prompted to provide valid credentials.
4. The Docker daemon requests the image manifest for the `cargo-lambda:latest` image.
5. The container registry responds with the manifest, which contains information about the layers and configuration of the `cargo-lambda` image.
6. The Docker daemon starts downloading the image layers one by one, verifying the integrity.
7. As the layers are downloaded, Docker daemon assembles them into a complete filesystem, representing the image.
8. Once the image is fully assembled, it is ready for use as a base for running containers.
9. If the `cargo-lambda` image has shared layers with other images previously pulled, Docker daemon utilizes the cached layers, reducing download time.
10. The pulled `cargo-lambda` image is now available on the local machine and can be used to create and run containers.

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

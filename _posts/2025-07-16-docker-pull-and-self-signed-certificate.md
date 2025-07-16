---
title:  "Docker Pull and Self-Signed Certificate"
seo_title: "docker pull and self-signed certificate"
seo_description: "docker pull and self-signed certificate"
date:   2025-07-16 00:00:00 +0700
categories:
  - Programming
tags:
  - Docker
  - Linux
excerpt: "Using Docker Pull behind a transparent proxy could be a PITA..."
toc: true
toc_label: "Table of Contents"
---
# Overview
I was trying to pull a docker image from a docker registry but encountered the following error:

{% highlight bash %}
ERROR: failed to solve: ghcr.io/cargo-lambda/cargo-lambda:   
latest: failed to resolve source metadata for ghcr.io/cargo-lambda/cargo-lambda:   
latest: failed to copy: httpReadSeeker:    
failed open: failed to do request: Get "https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:<hash>?se=<date>&sig=<sig>&sp=r&spr=https&sr=b&sv=2019-12-12":    
tls: failed to verify certificate: x509: certificate signed by unknown authority   
{% endhighlight %}

# How Docker Pull Works
By default, Docker does not trust an insecure registry without a valid signed certificate. When you are working behind a corporate network, usually there will be a transparent proxy intercepting all HTTPS traffic and replaces the certificate with their own. . Sometimes, this certificate is not known by the third-party registries. Hence, the error mentioning the usage of "self-signed certificate" as shown above.

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

My solution for the above is to add registry domain in Docker's `insecure-registries`.

Edit or create the file `/etc/docker/daemon.json` and add the following:

{% highlight bash %}
{
        "insecure-registries": ["gcr.io", "ghcr.io"],
        "dns": ["8.8.8.8"]
}
{% endhighlight %}

Then, restart the docker daemon. 
{% highlight bash %}
# I am using WSL2 so systemd is not the init system
sudo service docker restart
{% endhighlight %}

Your image pull should now works.

# Conclusion

Corporate networks usually add some transparent safeguards (for good reasons) to ensure data protection for its employees. However, the default behavior that the networking services choose for some workflows could be confusing, especially to the power users within the company. One workaround for the common Docker workflow has been explained above. Feel free to follow and adapt but use it on your own risk. 
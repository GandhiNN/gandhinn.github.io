---
title:  "Calculating AWS Glue Cost using AWS SDK for Rust"
seo_title: "calculating aws glue cost using aws sdk for rust"
seo_description: "calculating aws glue cost using aws sdk for rust"
date:   2025-04-21 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
excerpt: "This post is describing how we can determine the cost of AWS Glue ETL job using AWS SDK for Rust..."
toc: true
toc_label: "Table of Contents"
---
# Overview
In my current employer, one of my day-to-day responsibilities is overseeing more than 300 data pipelines moving TBs of data daily from all of our affiliates across the globe. We choose AWS Glue as the go-to technology for data pipelines because of its fully-managed and serverless approach which free us from some management overhead. But as with many AWS services, the pricing model can be a little bit hard to understand since now we are talking about abstracted metrics i.e what AWS called as "DPU Hours". 

AWS Glue Console already provides us with a visual tool for the users to monitor their Glue Usage. This tool is called "AWS Glue Job Monitoring." However, I found that this is quite inflexible for me as a power user to control the views that I would like to have e.g. if I would like to export the statistics into a universal format i.e. CSV files for further post-processing and analytics. I decided to build a CLI tool to solve this problem by using AWS SDK for Rust, however there are some challenges that I need to consider:

# 0. How does we use the AWS SDK for Rust for Glue? What is the API to use?
TBC

# 1. How does we differentiate between Glue Standard ETL, Glue Auto-Scaling ETL, and Glue Python Shell?
TBC

# 2. How does we calculate the DPU hours based on the job run time?
TBC

## Extra: How does we handle rounding up of decimal numbers in Rust?
TBC

# 3. How does we convert the DPU hours into monetary values?
TBC

# Conclusion
TBC

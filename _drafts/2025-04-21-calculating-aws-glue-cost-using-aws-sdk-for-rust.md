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
In my workplace, one of my day-to-day responsibilities is overseeing more than 300 data pipelines moving TBs of data daily from all of our affiliates across the globe. We choose AWS Glue as the go-to technology for data pipelines because of its fully-managed and serverless approach which free us from some management overhead. But as with many AWS services, the pricing model can be a little bit hard to understand since now we are talking about abstracted metrics i.e what AWS called as "DPU Hours." Simply speaking, in Glue, we are charged for the maximum number of DPUs used, and the total usage duration, at any point during the job execution.

AWS Glue Console already provides us with a visual tool for the users to monitor their Glue Usage. This tool is called "AWS Glue Job Monitoring." However, there are some things that I would like to do which is yet to be made available by AWS. For example, there is no feature to export the job's run statistics into tabular formats for further post-processing and analytics. Hence, I decided to build a CLI tool which leverages [AWS SDK for Rust](https://aws.amazon.com/sdk-for-rust/) to solve this problem. However, there are some challenges that I need to answer:

1. What is the API to use to get the data related to AWS Glue cost?
2. How do we differentiate between Glue Standard, Standard with Auto-Scaling, and Python Shell ETL jobs?
3. How do we convert AWS Glue job runtime into AWS Glue DPU-hours?
4. How do we convert AWS Glue DPU-hours into the actual cost in USD?

In this post, I will describe my approach to answer those challenges.

# 1. What is the API to use to get the data related to AWS Glue cost?
TBC

# 2. How do we differentiate between Glue Standard, Standard with Auto-Scaling, and Python Shell ETL jobs?
One thing I noticed when working with Glue's `GetJobRun` API in AWS SDK for Rust is, we can differentiate between Glue Standard (both Spark and Python Shell variants) and Glue Auto-Scaling ETL by looking at the value of the response fields.

The Auto-Scaling ETL job will set the value of `dpu_seconds` into `Some(U64)` type and `execution_class` as `None`. On the other hand, Glue Standard Spark ETL jobs variant will set the value of `dpu_seconds` as `None` and `execution_class` as `Some(Standard)`. Finally, The Python Shell variant will have the value of both `dpu_seconds` and `execution_class` as `None`.

# 3. How does we calculate the DPU hours based on the job run time?
TBC

## Extra: How does we handle rounding up of decimal numbers in Rust?
TBC

# 4. How does we convert the DPU hours into monetary values?

Let's start by having some brief overview of the calculation for each Glue Job variant:

## Glue Spark Standard ETL Job
There are several scenarios due to the number of configuration possibilities:

### Scenario 1
Condition: AWS Glue Spark job (Glue 3.0) with G1.X worker type that runs with 6 workers for 15 minutes = 6 DPU
Fact: the price of 1 DPU-hour is $0.44 for Glue Spark job (Glue version 2.0 and above) 
Calculation: because our job ran for 15 minutes (0.25 hour) and used 6 DPUs, the bill will be = 6 DPU x 0.25 hour x $0.44 → $0.66

### Scenario 2
Condition: AWS Glue Spark job (Glue 1.0) with G.4X worker type that runs with 3 workers for 30 minutes = 12 DPU 
Fact: the price of 1 DPU-hour is $0.44 for Glue Spark Job (Glue 0.9 and Glue 1.0)
Calculation: because our job ran for 30 minutes (0.5 hour) and used 12 DPUs, the bill will be = 12 DPU x 0.5 hour x $0.44 → $2.64

## Glue Spark with Auto-Scaling Enabled ETL Job
In Glue with auto-scaling enabled, we set the maximum number of workers and let AWS Glue monitors the Spark application execution. AWS Glue then allocates more worker (executor) nodes to the cluster in near-real time after Spark requests more executors based on our workload requirements. When there are idle executors that don't have intermediate shuffle data, AWS Glue Auto Scaling removes the executors to save the cost.

However, this makes our pricing calculation to be quite non-deterministic i.e. we cannot predict the cost upfront, as the AWS Glue Spark cluster dynamically scales out and scales in the executor during job run time.

## Glue Python Shell
Glue Python Shell uses the same formula as with the Standard ETL Job variant. However, we have to handle more carefully the calculation and rounding ups of the DPU-hour because Glue Job Monitoring Console could present a false impression of the cost incurred by Glue Python Shell Jobs. For example, suppose that you have a Python Shell job with 0.0625 DPU configuration which ran for 2 minutes, then the total DPU-hours of this job would be calculated as: (2 / 60) * 0.0625 = 0.00208. The Glue Job Monitoring Console will display this as "0.00" because it handles only up to 2 decimal numbers.

# Conclusion
TBC

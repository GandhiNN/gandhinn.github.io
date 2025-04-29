---
title:  "AWS Glue Cost calculation using AWS SDK for Rust"
seo_title: "aws glue cost calculation using aws sdk for rust"
seo_description: "aws glue cost calculation using aws sdk for rust"
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
I used the `get_job_runs()` API. This API returns an array of Glue Job runs which contains many information that I can use to calculate the cost. In my case, I am starting with defining a container struct to hold the fields that I need:

{% highlight rust %}
use serde::Serialize;

#[derive(Serialize)]
pub struct Run {
    pub job_name: String,
    pub run_id: String,
    pub started_on: String,
    pub completed_on: String,
    pub duration: i32,
    pub state: String,
    pub worker_type: String,
    pub max_capacity: f64,
    pub dpu_hours: f64,
    pub cost_usd: f64,
}
{% endhighlight %}

Then I built a Glue SDK client struct. Building the client itself is out of scope of this post:

{% highlight rust %}
use aws_sdk_glue::{Client, Error, types::WorkerType};

pub struct OpClient {
    pub client: Client,
}

impl OpClient {
    pub async fn new(profile: &str, disable_stalled_stream_protection: &str, timeout: u64) -> Self {
          let credentials = load_credentials(profile);
          let config = set_config(
              profile,
              credentials,
              disable_stalled_stream_protection,
              timeout,
          )
          .await;
          let client = Client::new(&config);
          Self { client }
      }
    ...
}
{% endhighlight %}

Next is I defined a method for `OpClient` which implements the `get_job_run()` API. This method returns a result type which contains a vector of Glue Job runs, serialized to the `Run` struct I've defined earlier, and the possible error variants.

{% highlight rust %}
    ...
    pub async fn get_job_runs(&self, job_name: &str) -> Result<Vec<Run>, Error> {
        let mut job_runs = self
            .client
            .get_job_runs()
            .job_name(job_name)
            .into_paginator()
            .send();
        let mut glue_job_runs: Vec<Run> = Vec::new();
        while let Some(job_runs_output) = job_runs.next().await {
            match job_runs_output {
                Ok(v) => {
                    let runs = v.job_runs();
                    for run in runs {
                        // println!("{:#?}", run);
                        glue_job_runs.push(Run {
                            job_name: run.job_name().unwrap().to_string(),
                            run_id: run.id().unwrap().to_string(),
                            started_on: run.started_on().unwrap().to_string(),
                            completed_on: run.completed_on().unwrap().to_string(),
                            duration: run.execution_time(),
                            state: run.job_run_state().unwrap().to_string(),
                            worker_type: {
                                match run.worker_type() {
                                    Some(WorkerType::G025X) => "G025X".to_string(),
                                    Some(WorkerType::G1X) => "G1X".to_string(),
                                    Some(WorkerType::G2X) => "G2X".to_string(),
                                    Some(WorkerType::G4X) => "G4X".to_string(),
                                    Some(WorkerType::G8X) => "G8X".to_string(),
                                    Some(WorkerType::Standard) => "Standard".to_string(),
                                    Some(WorkerType::Z2X) => "Z2X".to_string(),
                                    Some(other) if other.as_str() == "NewFeature" => {
                                        "Other".to_string()
                                    }
                                    _ => "None".to_string(),
                                }
                            },
                            max_capacity: run.max_capacity().unwrap_or_default(),
                            dpu_hours: get_dpu_hours(run.clone()),
                            cost_usd: get_cost(get_dpu_hours(run.clone())),
                        })
                    }
                }
                Err(e) => println!("{:#?}", e),
            }
        }
        Ok(glue_job_runs)
    ...
{% endhighlight %}

# 2. How does we calculate the DPU hours based on the job run time?
In the above code, you can see a function called `get_dpu_hours()` which is called to get the value of the DPU hours for a particular job runs. 

Here's the function signature:

{% highlight rust %}
pub fn get_dpu_hours(r: JobRun) -> f64 {
    if r.dpu_seconds.is_none() {
        // Glue Standard (Spark and Python Shell)
        let dur = r.execution_time();
        if dur < 60 {
            let dur_rounded = mathutil::ceiling(dur as u64, 60); // round up to 60 seconds
            let d = ((dur_rounded as f64 / 60_f64) * r.max_capacity().unwrap_or_default()) / 60_f64;
            precision_f64(d, 2)
        } else {
            let d = ((dur as f64 / 60_f64) * r.max_capacity().unwrap_or_default()) / 60_f64;
            precision_f64(d, 3)
        }
    } else {
        // Handle for Glue auto-scaling (Spark)
        let dur = r.execution_time();
        if dur < 60 {
            let dur_rounded = mathutil::ceiling(dur as u64, 60); // round up to 60 seconds
            let d = (dur_rounded as f64 / 60_f64) / 60_f64; // we don't care about the max capacity as it's dynamically scaled
            precision_f64(d, 2)
        } else {
            let d = (r.dpu_seconds().unwrap() / 60_f64) / 60_f64;
            precision_f64(d, 3)
        }
    }
}
{% endhighlight %}

## Extra: How does we handle rounding up of decimal numbers in Rust?
In the above code, you can see that 
I am defining a custom math method called `mathutil::ceiling()` which has the following function definition:

{% highlight rust %}
// mathutil.rs

pub fn ceiling(num: u64, nearest: u64) -> u64 {
    if num < nearest {
        nearest
    } else {
        let mut counter = 0;
        let mut rounded = 0;
        while rounded < num {
            counter += 1;
            rounded += nearest;
        }
        counter * nearest
    }
}
{% endhighlight %}

# 3. How do we differentiate between Glue Standard, Standard with Auto-Scaling, and Python Shell ETL jobs?
One thing I noticed when working with Glue's `GetJobRun` API in AWS SDK for Rust is, we can differentiate between Glue Standard (both Spark and Python Shell variants) and Glue Auto-Scaling ETL by looking at the value of the response fields.

The Auto-Scaling ETL job will set the value of `dpu_seconds` into `Some(U64)` type and `execution_class` as `None`. On the other hand, Glue Standard Spark ETL jobs variant will set the value of `dpu_seconds` as `None` and `execution_class` as `Some(Standard)`. Finally, The Python Shell variant will have the value of both `dpu_seconds` and `execution_class` as `None`.

# 4. How does we convert the DPU hours into monetary values?
Let's start by having some brief overview of the calculation for each Glue Job variant:

|-------------+--------------+-----------------+----------+-------------------------+---------|
| Price (USD) | Pricing Unit | Billing Counter | Job Type | Minimum Billing Counter | Remarks |
|:-----------:|:------------:|:---------------:|:--------:|:-----------------------:|:-------:|  
|0.44|per DPU-hour|per second|Spark / Spark Streaming|1 minute|Glue 2.0 and above|
|0.44|per DPU-hour|per second|Spark / Spark Streaming|10 minute|Glue 0.9 / Glue 1.0|
|0.29|per DPU-hour|per second|Spark Flex|1 minute|Glue 3.0 and above|
|0.44|per M-DPU-hour|per second|Ray|1 minute|NA|
|0.44|per DPU-hour|per second|Provisioned development endpoint|10 minute|NA|
|0.44|per DPU-hour|per second|Interactive Session|1 minute|NA|
|0.44|per DPU-hour|NA|AWS Glue Data Quality|NA|NA|

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

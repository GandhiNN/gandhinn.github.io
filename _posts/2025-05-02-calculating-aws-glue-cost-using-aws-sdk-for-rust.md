---
title:  "AWS Glue Cost Calculation using AWS SDK for Rust"
seo_title: "aws glue cost calculation using aws sdk for rust"
seo_description: "aws glue cost calculation using aws sdk for rust"
date:   2025-05-02 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
excerpt: "This post is describing how we can determine the cost of AWS Glue ETL job using AWS SDK for Rust..."
toc: true
toc_label: "Table of Contents"
---
# Overview
One of my day-to-day responsibilities at work is to oversee more than 300 data pipelines moving terabytes of data daily from all of our manufacturing facilities across the globe. We choose AWS Glue as the go-to technology for data pipelines because of its fully-managed and serverless approach which free us from some management overhead. But as with many AWS services, the pricing model can be a little bit hard to understand since we are using an abstracted metric which AWS described as "DPU Hours." Simply speaking, in Glue, we are charged by the combination of compute hours and the total usage duration, at any point during the job execution.

AWS Glue Console already provides us with a visual tool for the users to monitor their usage of AWS Glue. This tool is called "AWS Glue Job Monitoring." However, there are still some "missing features." For example, there is no feature to export the job's run statistics into tabular formats for further post-processing and analytics. Hence, I decided to build a CLI tool which leverages [AWS SDK for Rust](https://aws.amazon.com/sdk-for-rust/) to solve this problem. During my development journey, there are 5 big questions that I need to answer:

1. What is the main driver for AWS Glue price?
2. What is the API to use to get the data related to AWS Glue cost?
3. How do we differentiate between Glue Standard, Standard with Auto-Scaling, and Python Shell ETL jobs?
4. How do we convert AWS Glue job runtime into AWS Glue DPU-hours?
5. How do we convert AWS Glue DPU-hours into the actual cost in USD?

This post describes my answer to these questions.

# 1. What is the main driver for AWS Glue cost?
AWS Glue cost structure is mainly driven by Data Processing Units (DPUs). DPUs provide the computation power necessary to execute ETL (Extract, Transform, Load) operations of Glue. A DPU consists of 4 vCPUs and 16 GB of memory. AWS Glue is billed on hourly usage which has an average standard rate of $0.44 per DPU-hour.

AWS Glue has many worker types where each type has its own DPU and node configuration. Bigger worker types comes with more DPU per node i.e. it provides higher performances. For example, G.2X type has 2 DPU per node, 8 vCPUs, 32 GB of RAM, and 128 GB of disk whereas G.1X's is half of those numbers. Naturally, using bigger worker types usually means that we have to pay more price, [although it's not always the case](https://aws.amazon.com/blogs/big-data/scale-your-aws-glue-for-apache-spark-jobs-with-new-larger-worker-types-g-4x-and-g-8x/). This is important especially if we are using the Glue Standard ETL Job. In this case, we have to also factor in the number of workers used during the job runtime, which will be explained in more detail later in this post. 

# 2. What is the API to use to get the data related to AWS Glue cost?
We can use the `get_job_runs()` API of AWS Glue SDK. This API returns an array of job runs for a given AWS Glue job name. This array  contains many information that we can use to calculate the cost. In my code, I defined a container struct to hold the fields that I need to be outputted at the end:

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

Then, I defined a Glue SDK client struct. Building the client itself is out of scope of this post:

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

Next, I defined a `get_job_runs()` method for `OpClient` which implements the `get_job_run()` API under the hood. This method returns a result type which contains a vector of Glue Job runs, serialized to the `Run` struct I've defined earlier. This method also returns possible error variants:

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

# 3. How do we calculate the DPU hours based on the job run time?
In the above code snippets, you can see a function called `get_dpu_hours()` which is called to get the value of the DPU hours for a particular job runs. In this function, we use some basic conditional statements to handle different type of Glue Jobs. The function is defined as follows:

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

## Extra: How do we handle rounding up of decimal numbers in Rust?
As far as my knowledge goes, Rust does not provide a standard function to round up floating point numbers to the nearest N number (ala Microsoft Excel's `=CEILING(<cell_num>, N))` built-in function). As a workaround, I defined some custom mathematical functions to achieve similar behavior.

In the `get_dpu_hours` code snippet, I am using two custom math methods:

* `mathutil::ceiling()` -> This method is used when a job runs less than 1 minute (60 seconds). This function has the following signature:


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


* `precision_f64()` -> This method is used directly when a job runs more or equal than 1 minute (60 seconds). This function has the following signature:

{% highlight rust %}
pub fn precision_f64(x: f64, decimals: u32) -> f64 {
    if x == 0. || decimals == 0 {
        0.
    } else {
        let shift = decimals as i32 - x.abs().log10().ceil() as i32;
        let shift_factor = 10_f64.powi(shift);

        (x * shift_factor).round() / shift_factor
    }
}
{% endhighlight %}
 
You may ask why there's an additional step to round up the Job run time when the job finishes less than 60 seconds. It's basically to align with how AWS Glue Job monitoring dashboard displays the DPU-hours, It displays the DPU hours as floating point numbers with 2 decimal digits, and due to the fact that when the `get_job_run()` API returns the job run time in seconds, the cost is modelled in DPU-hours, i.e. it is calculated in an hourly basis. Then, it means first you have to convert the run time into minutes and then divide it again with 60, hence the logic to round up the calculation into the nearest 60. This additional step ensures that we will have the numbers of DPU hours as close as possible with the numbers shown in the Glue Job Run Monitoring dashboard.

# 4. How do we differentiate between Glue Standard, Standard with Auto-Scaling, and Python Shell ETL jobs?
One thing I noticed when working with Glue's `GetJobRun` API in AWS SDK for Rust is, we can differentiate between Glue Standard (both Spark and Python Shell variants) and Glue Auto-Scaling ETL by looking at the value of the response fields.

The Auto-Scaling ETL job will set the value of `dpu_seconds` into `Some(U64)` type and `execution_class` as `None`. On the other hand, Glue Standard Spark ETL jobs variant will set the value of `dpu_seconds` as `None` and `execution_class` as `Some(Standard)`. Finally, The Python Shell variant will have the value of both `dpu_seconds` and `execution_class` as `None`.

# 5. How do we convert the DPU hours into monetary values?
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

In my workloads, there are 3 main use cases of AWS Glue:

## Case 1: Glue Spark Standard ETL Job
There are several scenarios due to the number of configuration possibilities, here are two of the most common ones in my daily job:

**Scenario 1**
Condition: AWS Glue Spark job (Glue 3.0) with G1.X worker type that runs with 6 workers for 15 minutes = 6 DPU
Fact: the price of 1 DPU-hour is $0.44 for Glue Spark job (Glue version 2.0 and above) 
Calculation: because our job ran for 15 minutes (0.25 hour) and used 6 DPUs, the bill will be = 6 DPU x 0.25 hour x $0.44 → $0.66

**Scenario 2**
Condition: AWS Glue Spark job (Glue 1.0) with G.4X worker type that runs with 3 workers for 30 minutes = 12 DPU 
Fact: the price of 1 DPU-hour is $0.44 for Glue Spark Job (Glue 0.9 and Glue 1.0)
Calculation: because our job ran for 30 minutes (0.5 hour) and used 12 DPUs, the bill will be = 12 DPU x 0.5 hour x $0.44 → $2.64

## Case 2: Glue Spark with Auto-Scaling Enabled ETL Job
In Glue with auto-scaling enabled, we set the maximum number of workers and let AWS Glue monitors the Spark application execution. AWS Glue then allocates more worker (executor) nodes to the cluster in near-real time after Spark requests more executors based on our workload requirements. When there are idle executors that don't have intermediate shuffle data, AWS Glue Auto Scaling removes the executors to save the cost.

The downside is this logic makes our pricing calculation to be quite non-deterministic i.e. we cannot predict the cost upfront, as the cluster dynamically scales out and scales in the executor during job run time.

## Case 3: Glue Python Shell
Glue Python Shell uses the same formula as with the Standard ETL Job variant. However, we have to handle more carefully the calculation and rounding ups of the DPU-hour because Glue Job Run Monitoring Console could present a false impression of the cost incurred by Glue Python Shell Jobs. 

For example, suppose that you have a Python Shell job with 0.0625 DPU configuration which ran for 2 minutes, then the total DPU-hours of this job would be calculated as: (2 / 60) * 0.0625 = 0.00208. The Glue Job Monitoring Console will display this as "0.00" because it handles only up to 2 decimal numbers.

# Glue Job Run cost calculation in action
Enough of theory, let's run a concrete examples. Here, I have a Glue Python Shell Job (job name and run id are redacted) which consistently finishes under 1 minute. The Glue Job Run monitoring dashboard displays the DPU hours as 0.02 for all job runs:

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-05-02-calculating-aws-glue-cost-using-aws-sdk-for-rust/img1-blogpost-20250502.png){: .align-center}

Here's the result when I queried the job run using my custom functions:

{% highlight bash %}
      | duration | state     | worker_type | max_capacity | dpu_hours | cost_usd
---------------------------------+---------------------------------------------------------------------+-----------------------------+-----------------------------+----------+-----------+-------------+--------------+-----------+-----------------------
 ********** | ********** | 2025-05-02T04:00:26.210999Z | 2025-05-02T04:01:47.026Z    | 35       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
 ********** | ********** | 2025-05-01T04:00:26.173Z    | 2025-05-01T04:02:05.4Z      | 47       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
 ********** | ********** | 2025-04-30T04:00:26.242Z    | 2025-04-30T04:02:03.282999Z | 40       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
 ********** | ********** | 2025-04-29T04:00:26.313999Z | 2025-04-29T04:01:53.619999Z | 35       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
 ********** | ********** | 2025-04-28T04:00:26.467Z    | 2025-04-28T04:02:11.681999Z | 35       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
 ********** | ********** | 2025-04-27T04:00:26.167Z    | 2025-04-27T04:01:49.9Z      | 32       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
 ********** | ********** | 2025-04-26T04:00:26.161999Z | 2025-04-26T04:02:04.845999Z | 39       | SUCCEEDED | None        | 1            | 0.017     | 0.0074800000000000005
{% endhighlight %}

We can see that although we are losing a tiny bit of precision in DPU hours (and subsequently the USD cost), we can use this logic as a foundation to do something on top e.g. if we want to build a custom monitoring dashboard with better flexibility and features than the existing dashboard provided by AWS, for example things like historical analytics with longer time frame.

# Conclusion
When using AWS Glue in your data pipelines, it's important to understand how to calculate your DPU pricing. This understanding can lead to better oversights of your spending, especially when you have a large-scale data platform and you are heavily utilizing an abstracted configuration such as Glue Auto Scaling. 

The AWS SDK for Rust provides us with extensive list of APIs to achieve this goal. While AWS already provides a visual tool out of the box for us to do the same, we can use AWS SDK for Rust to build a tool that is more tailored to our specific needs. Furthermore, the SDK is engineered to be fast, with serializers and deserializers that minimize unnecessary copies and allocations. This reduces CPU and memory utilization by the SDK, freeing up more of these resources for your downstream application.
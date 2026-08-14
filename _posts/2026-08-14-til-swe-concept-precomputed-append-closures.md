---
title:  "TIL - Pre-Computed Append Closures and its Impact on Performance of Your Code"
seo_title: "til pre-computed append closures and its impact on performance of your code"
seo_description: "today i learned how using pre-computed append closures can have good performance impact in your code"
date:   2026-08-14 00:00:00 +0700
categories:
  - Programming
tags:
  - Python
  - SWE
excerpt: "Today I learned how using pre-computed append closures approach can bring good performance impact in your code..."
toc: true
toc_label: "Table of Contents"
---

## Pre-Computed Append Closures

### The Movie Cinema Analogy

The settings: A movie cinema which has several theater types/classes: "Regular", "IMAX", or "Premiere" and a ticketing booth that is served by a human / "front desk officers".

The flow: A customer come to the front desk saying the name of the movie name he/she wants to watch -> The front desk officers map the movie name with the theater it's playing at -> The front desk officers print out the ticket and stamp it with the name of the theater -> The front desk give the stamped ticket to the customer (the **hot loop**)

#### Approach 1: Type Switch (a.k.a the slow way)

Imagine a situation where for every single order, the front desk officer walks to the back office and asks:
> "Hey, this movie, is it playing at Regular, IMAX, or Premiere (theater type/class)?"

The back office checks and points to the right theater, then the front office stamps the ticket with the correct theater placement, then he/she goes back to his/her desk. Repeats for every single customer (For the sake of simplicity, let's assume that the front desk officers does not retain any memory i.e. he/she forgots the mapping of film and theater after serving a customer.)

With 1000 customers, that's 1000 times the front office needs to walk to the back office and asking: "where does this film play?"

#### Approach 2: Pre-computed Closures (a.k.a the fast way)

Now imagine the same situation, but now the front desk officer has a cheat sheet of where each film is exactly playing:
> "The Odyssey always plays at IMAX, Spiderman is playing at Premiere, and Evil Dead is playing at Regular..."

Now, when serving the customers, the front desk officers does not need to ask anyone, he/she just looks at this cheat sheet and directly stamps the ticket with the correct theater type/class. He/she spent less time questioning, waiting, and thinking.

In this case, the cheat sheet is the **closure**. It has the answer ("film A is playing in IMAX, film B is playing in Premiere, etc") at "setup time", so the hot loop never has to figure it out again, saving a lot of time in the process.

The front desk officers will still need to check 1000 times, but the difference this time is it's much easier for them to check which theater a film is playing at (they no longer have to walk to the back office every single time they serve a customer).

### Why does this matter?

This is the idea of a pre-computed append closures: We create a function (closure) that **has less work to do every time it's being called**.

Imagine a scenario where you have to do ETL of raw data from an MS-SQL server and save the result sets into Parquet files. Having the "cheat sheet" will help the computation process ("this column is an int: use the int builder, this column is a string: use the string builder") at ETL setup time so the hot loop (ingest raw data -> casting types into proper Arrow types -> save to disk as Parquet files) never has to figure the type mapping again.

The simplest cheat sheet is having a dictionary of column and its data type: We can run a query to probe the schema once in the beginning of the ETL phase (i.e. executing query `SELECT TOP 0 * FROM (<table_name>) AS _probe` in MS-SQL and parse the result set into type mapping)

This means improved performance: CPU is the main actor on this process. CPUs work like an assembly line, they start processing the next instruction before the current one finishes (pipelining). But when they hit a decision point ("which type is this?"), they have to guess which path to take. If they guess wrong, they throw away the work and start over (pipeline stall).

- When using the **Type switch** approach: The decision changes every column (int, string, float, int, string...). The CPU guesses wrong more often -> more pipeline stalls.
- When using the **Pre-Computed Closures** approach: The only decision is checking whether the cell is null, and the answer most likely is "no". The CPU guesses correctly more often -> the pipeline is flowing smoother.

### Code

{% highlight python %}
import time

# Simulating an expensive setup work
def create_converter(schema):
    time.sleep(0.01) # simulate 10ms of work

    def converter(batch):
        return [x * 2 for x in batch]
    
    return converter

# 1. Without pre-computation

def append_without_precompute(batch):
    schema = {"type": "int"}
    converter = create_converter(schema)

    result = converter(batch)
    return result

# 2. With pre-computed closure

def make_appender():
    schema = {"type": "int"}

    # Expensive setup work which happens ONCE
    converter = create_converter(schema)

    def append(batch):
        result = converter(batch)
        return result
    
    return append

# Testing

batches = [
    list(range(100)),
    list(range(100)),
    list(range(100)),
    list(range(100)),
    list(range(100)),
]

# Without pre-computation
start = time.perf_counter()

for batch in batches:
    append_without_precompute(batch)

without_time = time.perf_counter() - start

# With pre-computation
start = time.perf_counter()

append = make_appender()

for batch in batches:
    append(batch)

with_time = time.perf_counter() - start

print(f"Without pre-computation :   {without_time:.4f} seconds")
print(f"With pre-computation    :   {with_time:.4f} seconds")

{% endhighlight %}

## Under the hood

The without pre-computation loop:

{% highlight python %}
for batch in batches:
        append_without_precompute(batch)
{% endhighlight %}

for each iteration, it does the following:

{% highlight bash %}
batch
 ↓
create_converter() # expensive operation
 ↓
convert
 ↓
return
{% endhighlight %}

So for 5 batches we will have 5 converter creations.

Now, the loop using pre-computed closure:

{% highlight python %}
for batch in batches:
    append(batch)
{% endhighlight }

we only created the converter once:

{% highlight bash %}
make_appender()
 ↓
create_converter() # one time, converter is persisted
 ↓
append(batch)
append(batch)
append(batch)
append(batch)
append(batch)
{% endhighlight %}

It executes the same number of `append(batch)` calls, but it has less work per call, because the closure is carrying within itself the prepared `converter` logic.

Now, imagine if we replace our `create_converter()` closure creator with something that has functions like: schema resolution, column mapping, encoder selection, compression selection, or writer setup. We could save on these expensive operations by carrying them within the closure that we are applying to each batches of rows. It could save some computation time, making our ETL runs faster.

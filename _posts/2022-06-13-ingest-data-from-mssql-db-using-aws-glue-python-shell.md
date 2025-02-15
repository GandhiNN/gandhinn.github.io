---
title:  "Ingest Data from MSSQL DB Using AWS Glue Python Shell"
seo_title: "ingest data from mssql db using aws glue python shell"
seo_description: "ingest data from mssql db using aws glue python shell"
date:   2022-06-13 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
  - SQL
  - Python
excerpt: "In many multinational companies, Microsoft SQL Database Server is one of the three most deployed database systems. Its compatibility with other Microsoft Products...."
---
### Background
In many multinational companies, Microsoft SQL Database Server is one of the three most deployed database systems. Its compatibility with other Microsoft Products, which still dominates the end-user OS market, is one reason why it's still prevalent.

However, as cloud transformation begins to take shape, some cross-architectural needs also arise. With cloud components being mostly Linux in nature, making its components to work with Microsoft systems proves to be something that needs more brainpower to solve.

In this article, I am sharing my experience ingesting data from Microsoft SQL Server databases using AWS Glue Python Shell.

### Rationale

Why are we using AWS Glue Python Shell?

1. **Flexibility** : This is useful in a scenario where we want to run small to medium-sized generic ETL tasks toward Microsoft SQL databases, and still be able to use all the ETL orchestration toolset that Glue provides e.g. "Trigger" and "Workflows", while not being limited by the amount of memory and runtime environment ceiling we could allocate to the job (cue : Lambda).

2. **Cost Saving** : Spark and Streaming jobs on Glue require a minimum of 2 DPU, whereas we can run Python shell jobs using 1 DPU or 0.0625 DPU. Glue Python Shell is even cheaper than a typical Lambda function invocation, about a half of the price for the same resources configured to handle similar workloads.

### The Catch
Microsoft official documentation recommends developers to use `pyodbc` library as the Python SQL driver to work with MSSQL DB.

However, `pyodbc` requires some native libs (e.g installation of unixODBC and unixODBC-devel packages, along with their shared libraries) to be available on the machine where we want to run the job. AWS Glue Python Shell is an ephemeral Amazon Linux runtime environment and it will be too hackish to install those dependencies during job runtime.

A workaround for this is to configure Glue to use `pymssql` library. It is a simple database interface for Python that builds on top of FreeTDS, a set of libraries for Unix and Linux that allows our programs to natively talk to Microsoft SQL Server and Sybase databases, to provide a Python DB-API (PEP-249) interface to Microsoft SQL Server.

Please note that at the time of this writing, Glue Python Shell version is still at version `3.6.13`, so we have to make sure that we are using the appropriate version of the `pymssql` library.

### The Steps

1. Download `pymssql` wheel for Linux build for Python 3.6 and upload the wheel to an S3 bucket. The wheel that I used can be found [in this link](https://files.pythonhosted.org/packages/bc/af/503da3351f6301ee1af5e34b10ba599afd67d20db0c70b0d05d4af56947f/pymssql-2.2.5-cp36-cp36m-manylinux_2_24_x86_64.whl).

2. Create a new Glue Python Shell job and reference the S3 URI of the previously-uploaded pymssql wheel in your newly-created Glue job.

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/pymssql_1.png){: .align-center}

3. Configure the connection to be used by the Glue Job towards the DB :

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/pymssql_2.png){: .align-center}

4. Write the code. Example as follows :

{% highlight python %}
import pandas as pd
import pymssql
import sys

def main():
    # Just initial checks
    print(sys.version) # Glue Pyshell version is 3.6
    print("hello world")

    # Connect to SQL database
    sql_password = '<your_password_here>'
    sql_db = '<your_db_name_here>'

    # Use the following notation for Windows Authentication
    conn = pymssql.connect(
        server='<your_db_host_here>:<your_db_port_here>\\<your_db_instance_here>', # syntax => {host}:{port}\\{db_instance}
        user='<your_domain_here>\\<your_user_here>', # syntax => {domain}\\{username}}
        password=sql_password,
        database=sql_db
    )
    # Open the connection to the DB
    cursor = conn.cursor()
    stmt1 = 'SELECT @@VERSION' # SQL query to be executed
    stmt2 = 'SELECT TOP 10 * from <table>' # SQL query to be executed
    cursor.execute(stmt2)
    rs = cursor.fetchall() # result set is a list of tuples

    # Convert result set to dataframe
    cols = ['col1', 'col2', 'col3']
    df = pd.DataFrame(rs, columns=cols)
    print(df.head(), df.shape)

if __name__ == '__main__':
    main()
{% endhighlight %}


If all goes well, you will see the following entries on Cloudwatch logs :

{% highlight bash %}
3.6.13 (default, Jun 23 2021, 16:06:50)
[GCC 8.3.0]
hello world
col1 col2 col3
0 aa jkt 2018-02-02 12:26:00
1 bb bnl 2018-02-02 12:26:00
2 cc nwy 2018-02-02 12:26:00
3 dd wsh 2018-02-02 12:26:00
4 ee tty 2018-02-02 12:26:00

[3 rows x 16 columns] (3, 16)
{% endhighlight %}

### Conclusion

Microsoft SQL Server Database is one of the most-deployed DB systems in PMI. Given that we are undergoing a huge transformation effort to cloud, cross-architectural issues will arise. This interaction between AWS Glue and our MS-SQL instance is one of the example.

This use case could have been easily solved if we use Glue Spark straightaway, which provides us with Glue 2.0 runtime environment, which provides us with another flexibility by utilizing the --additional-python-modules argument. But this scenario is an overkill for something that sits somewhere between "non-Lambda" and "non-Spark" workloads.

We do have other interesting cross-architectural challenges waiting to be solved. If you are interested in working with us to solve those and contribute to the advancement our our transformational effort, [join us](https://www.pmi.com/careers/explore-our-job-opportunities?departments=Information+Technology&page=1).
---
title:  "Interfacing with Database: Python and Go Implementation"
seo_title: "interfacing with database python and go implementation"
seo_description: "interfacing with database python and go implementation"
date:   2019-04-03 00:00:00 +0700
categories:
  - Programming
tags:
  - Python
  - Go
  - SQL
excerpt: "This post will explain about how I am interfacing with Database using Python and Go."
toc: true
toc_label: "Table of Contents"
---
### Overview
It goes without saying that every journey in data analytics starts by learning SQL, the most mainstream language that is used to access almost all kind of databases out there. Since I want to kickstart the journey, I picked SQLite as my practice environment because of its self-contained and zero-configuration principles, making it the most convenient DB out there for this purpose.

For the last couple of years, I am getting more acquainted with two programming languages to support me in my line of work: Python and Go. I can say that Python is a mature language to work with a DB. I have made [a simple tool to import a CSV file into SQLite database](https://github.com/GandhiNN/data-wrangling/blob/master/tools/csv2sqlite.py). Now I wanted to know how the latter compares when creating a simple API to interact with SQLite DB. As a reference point, I am using tutorials from http://www.sqlitetutorial.net/, and to add some juices along the way, I decided to apply my aforementioned programming language knowledge in following the tutorials.

So let’s get started.

### The Setup
* `sqlite3 3.24.0` provided by MacOS Mojave 10.14
* Python `3.7.2`
* go version `1.12 darwin/amd64`
* [chinook.db sample database](http://www.sqlitetutorial.net/wp-content/uploads/2018/03/chinook.zip)

There are 11 tables in the chinook sample database. To make things simple, I decided to compare how the two programming languages work the `SELECT * FROM tracks;` statement in terms of number of lines of code needed to execute aforementioned statement and deal with the returned rows afterwards.

### The Python Way
It’s simple: first you import the `sqlite3` module:

{% highlight python %}
import sqlite3
{% endhighlight %}

Then, create a Connection object that represents the database, create a Cursor object from it, and call its `execute()` method to execute SQL command:

{% highlight python %}
conn = sqlite3.connect("./chinook.db")
cursor = conn.cursor()
cursor.execute("SELECT * FROM tracks;")
{% endhighlight %}

Since the command returns matching rows, assign the return value of `fetchall()` to a variable containing list of matching rows:

{% highlight python %}
rows = cursor.fetchall()
{% endhighlight %}

If you want to print the elements to standard output, just iterate through its elements:

{% highlight python %}
for row in rows:
    print(row)
{% endhighlight %}

Lastly, don’t forget to close the DB objects. You see that all can be done in 14–15 lines of code, and we don’t have to worry that much while handling NULL-able fields compared to Go (more on that later):

{% highlight python %}
cursor.close()
conn.close()
{% endhighlight %}

### The Go Way
Same logic, we have to import the package AND the specific database drivers (which must be installed separately). This way, the driver implementation is abstracted under the hood and we have a DB-independent API to interact with our database (although there are still differences in things such as arguments provided to a method to connect to DBs may have different semantics):

{% highlight go %}
package main
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3"
    
    "fmt"
    "log"
)
{% endhighlight %}

We have to create a `db.SQL` object, a database interface abstraction to be used later for creating statements and transactions:

{% highlight go %}
db, err := sql.Open("sqlite3", "./chinook.db")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
{% endhighlight %}

Since `SELECT * FROM xxx;` most likely will return multiple rows, we use the `Query` method which will return a `*Rows` object that we can iterate and scan in a loop:

{% highlight go %}
rows, err := db.Query("SELECT * FROM tracks;")
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()
{% endhighlight %}

Here, I stumbled into the one of many major differentiators between Go and Python implementation when extracting data from the query statement; While Python’s sqlite3’s `fetchall()` provides some sort of convenience by returning query results into a list of rows, in Go we have to statically define the columns' variables and their types:

{% highlight go %}
var (
        trackid     int
        name        string
        albumid     int
        mediatypeid int
        genreid     int
        composer     *string
        milliseconds int
        bytes        int
        unitprice    float64
    )
{% endhighlight %}

Note that composer is of `*string` (pointer to string) type because I found out that field is NULL-able. If you define it as only string, then you will get the following error during runtime when it scans a NULL value under that field:

{% highlight bash %}
Scan error on column index 5, name "Composer": unsupported Scan, storing driver.Value type <nil> into type *string
{% endhighlight %}

Next, we iterate and scan through `rows` and create a simple conditional to handle the case with aforementioned NULL-able field by replacing it with `"empty_composer"` when the the code scans the NULL value:

{% highlight go %}
for rows.Next() {
		err := rows.Scan(&trackid,&name,
                    &albumid,&mediatypeid,&genreid,
                    &composer,&milliseconds,&bytes,
                    &unitprice)
		if err != nil {
			log.Fatal(err)
		}
		if composer == nil {
			temp := "empty_composer"
			composer = &temp
		}
		fmt.Println(trackid,name,albumid,
                    mediatypeid,genreid,*composer, 
                    milliseconds,bytes,unitprice)
	}
{% endhighlight %}

We can see that to accomplish the same result, we need around thrice the amount of lines of code in Golang compared to Python. It’s not that baffling considering its static type system and error-handling philosophy (we can skip it entirely of course, but it’s not idiomatic).

### Conclusion
As stated, I am not trying to benchmark these two languages in terms of performance; a lot of posts can give a better explanation regarding this. I personally find that Python has the edge in term of library support for this kind of workload and has more user-friendly feel to it. Go is a relatively new language but with its popularity continue to rise I think it will get there in time.
---
title:  "Unit Testing Bash Command Chaining in Go"
seo_title: "unit testing bash command chaining in go"
seo_description: "unit testing bash command chaining in go"
date:   2021-04-29 00:00:00 +0700
categories:
  - Programming
tags:
  - Bash
  - Go
excerpt: "Every sysadmin knows the following command chain to produce a hashed string...."
toc: true
toc_label: "Table of Contents"
---
### The Problem
Every sysadmin knows the following command chain to produce a hashed string:
{% highlight bash %}
$ echo MyBankPassword | md5sum | awk '{print $1}'
2210253a2a6e6482bb373b668d7b3e90
{% endhighlight %}

### The Solution
Out of curiosity, I decided to write a simple function in Go to try emulating the same behavior:
{% highlight go %}
package hasher

import (
	"crypto/md5"
	"encoding/hex"
)

func MD5Hash(m string) string {

	mHashed := md5.Sum([]byte(m))
	mOut := hex.EncodeToString(mHashed[:])

	return mOut
}
{% endhighlight %}

The logic of the function are:
1. Accepts a string as input and converts it to a byte slice to be consumed by Sum function of the md5 library. This method returns a byte array of 16 in length.
2. Re-convert the byte array into string by passing the whole array into EncodeToString method of the hex package.

Let’s see the result, using the same MyBankPassword as the input string:

{% highlight go %}
package main

import (
	"fmt"

	"github.com/GandhiNN/anonymizer/hasher"
)

const pswd = "MyBankPassword"

func main() {

	hashed := hasher.MD5Hash(pswd)
	fmt.Println(hashed)
}
{% endhighlight %}

Let's test the code:

{% highlight go %}
$ bashHash=`echo MyBankPassword | md5sum | awk '{print $1}'`
$ goHash=`go run main.go`
$ if [ $bashHash == $goHash ]; then echo "${bashHash} <-> ${goHash} is true"; else echo "${bashHash} <-> ${goHash} is false"; fi
2210253a2a6e6482bb373b668d7b3e90 <-> 
93d290d421543618bbfccaa7aea739d6 is false
{% endhighlight %}

Uh-oh, it yielded different hash results. Why?

**The Case of Trailing Newline**

The difference in results is caused by the implicit `\n` character at the end of line when calling echo using default arguments.

As seen in `echo.c` from GNU `coreutils` package:

{% highlight c %}
if (display_return)    
   putchar ('\n');
{% endhighlight %}

`display_return`` is a flag that is set to `true`` by default, and will only be switched to false if we use the `-n`` flag during execution, as seen in the following snippet in the same code:

{% highlight c %}
case 'n':              
    display_return = false;              
    break; 
{% endhighlight %}

To properly emulate the above behavior, I added a little modification to the `MD5Hash` function:

{% highlight go %}
func MD5HashNew(m string) string {

	mByte := []byte(m + "\n")
	mHashed := md5.Sum(mByte)
	mOut := hex.EncodeToString(mHashed[:])

	return mOut
}
{% endhighlight %}

I added one newline char in the byte array representation of the string that needs to be hashed, before it’s passed as the argument to the Sum function call, for the method to produces the same output:

{% highlight bash %}
$ goHash=`go run main.go`
$ if [ $bashHash == $goHash ]; then echo "${bashHash} <-> ${goHash} is true"; else echo "${bashHash} <-> ${goHash} is false"; fi
2210253a2a6e6482bb373b668d7b3e90 <-> 2210253a2a6e6482bb373b668d7b3e90 is true
{% endhighlight %}

**Unit Testing the Code**

The `MD5Hash` function defined above is a wrapper for the underlying functions called from the `crypto/md5`` package. In that case, there is little reason for me to write an unit test for it since I can be quite confident that the maintainers of the codebase have written good tests.

Again, just out from curiosity, I wrote an unit test for the function. I wanted to know if there is a way to check whether this function can produce the same result if I were to use run-of-the-mill bash tools:

{% highlight go %}
package hasher

import (
	"log"
	"os/exec"
	"testing"
)

func TestHash(t *testing.T) {
	var strInput1 string = "MySwissBankAccountPassword"
	expected := bashWrapper(strInput1)
	actual := MD5Hash(strInput1)
	if actual != expected {
		t.Errorf("Expected %v, got %v", expected, actual)
	}

	var strInput2 string = "MyPanamaPapersAccountPassword"
	expected = bashWrapper(strInput2)
	actual = MD5Hash(strInput2)
	if actual != expected {
		t.Errorf("Expected %v, got %v", expected, actual)
	}
}

// bashWrapper : wrapper for BashPipedCommand
func bashWrapper(strInput string) string {

	echo := exec.Command("echo", strInput)
	md5 := exec.Command("md5sum")
	awk := exec.Command("awk", `{printf $1}`) 

	output, _, err := BashPipedCommand(echo, md5, awk)
	if err != nil {
		log.Println(err.Error())
	}

	if len(output) > 0 {
		return string(output)
	}
	return ""
}
{% endhighlight %}

I defined a simple test case for hashing, using two different strings as inputs. Then I compare the result of `MD5Hash` function against the one produced by the bash command chain `echo ${strInput} | md5sum | awk '{printf $1}'`. Note that I don’t need to pass the `'` in the awk variable.

The three bash commands above are passed as arguments to `BashPipedCommand` function which itself is encapsulated inside the private function `bashWrapper`.

Now, let’s define the function `BashPipedCommand`:

{% highlight go %}
package hasher

import (
	"bytes"
	"io"
	"os/exec"
)

// BashPipedCommand : for multiple piped commands invocation
//  i.e. `echo a | md5sum | awk '{print $1}'`
func BashPipedCommand(cmds ...*exec.Cmd) (pipeOut, stdErrs []byte, pipeErr error) {

	if len(cmds) < 1 {
		return nil, nil, nil
	}

	// memory container to collect the output and stderr from the command(s)
	var output bytes.Buffer
	var stderr bytes.Buffer

	last := (len(cmds) - 1)
	for i, cmd := range cmds[:last] {
		var err error
		// stdout of prev command as input to stdin of the executed command
		cmds[i+1].Stdin, err = cmd.StdoutPipe()
		if err != nil {
			return nil, nil, err
		}
		// collect each command's stderr
		cmd.Stderr = &stderr
	}

	// we do not collect all output
	//  just collect the stdout and stderr of the last command
	cmds[last].Stdout, cmds[last].Stderr = &output, &stderr

	// start each command
	for _, cmd := range cmds {
		err := cmd.Start()
		if err != nil {
			return output.Bytes(), stderr.Bytes(), err
		}
	}

	// wait for each command to complete
	for _, cmd := range cmds {
		err := cmd.Wait()
		if err != nil {
			return output.Bytes(), stderr.Bytes(), err
		}
	}

	// return the pipeline output and the collected standard error
	return output.Bytes(), stderr.Bytes(), nil
}
{% endhighlight %}

The logic as follows :
1. The function accepts an arbitrary number of bash commands that is larger or equal to `1`, which will be stored in a `cmds` slice.
2. Two buffers to store the `stdout` and `stderr` of the bash command chain are created.
3. The code loops through the elements of the `cmdsslice`, where the `stdout` of the previous command is ‘connected’ to the `stdin` of the next command in chain.
4. The `output` buffer is used to store the `stdout` of the last command in chain, and also collect the `stderr`, if any.
5. As the previous steps are only used to define the target buffer for the command invocation, the commands themselves need to be actually executed.
6. The code then waits for each command to complete to finally store the output / errors of the commands in the buffers.
7. Finally all the defined return values are returned as byte arrays.

After all that, let’s run the test:

{% highlight go %}
Running tool: /home/linuxbrew/.linuxbrew/bin/go test -timeout 30s -coverprofile=/tmp/vscode-goHjZVHu/go-code-cover github.com/GandhiNN/anonymizer/hasher
ok      github.com/GandhiNN/anonymizer/hasher   0.006s  coverage: 53.8% of statements
{% endhighlight %}

It passes.

Note that the low code coverage is expected, since we are only interested in testing the business logic i.e. the hashing, not the whole code itself.

### Takeaways
1. Be thorough when emulating bash tools’ behavior into Go, especially if it has many options of invocations. Reading the man entry for the tool is a good start.
2. You can emulate bash command chaining in Go. If your source of truth is the output of a particular bash tool, you can check your Go code’s logic by comparing the output of your function with the output of the bash command chain, in one codebase.
3. Test the business logic. You don’t need to test a wrapper to a properly written function provided inside a stable package. As can be seen that I did not write unit tests for methods that are derived from crypto/md5 and os/exec package.
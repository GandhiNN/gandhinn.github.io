---
title:  "Simple CI/CD Pipeline in AWS: Trigger JAR Build on CodeCommit Events"
seo_title: "simple ci/cd pipeline in aws: trigger jar build on codeCommit events"
seo_description: "simple ci/cd pipeline in aws: trigger jar build on codeCommit events"
date:   2021-03-28 00:00:00 +0700
categories:
  - DevOps
tags:
  - AWS
  - Java
excerpt: "This post is a summary of how I built a simple and automated JAR package build using AWS services..."
toc: true
toc_label: "Table of Contents"
---
### Overview
This post is a summary of how I built a simple and automated JAR package build using AWS services.

AWS services used are:

1. AWS CodeCommit : A fully-managed source control service. A GitHub on AWS.
2. AWS Lambda : A serverless compute service that lets us run code without provisioning or managing servers.
3. AWS CodeBuild : A fully-managed continuous integration service that compiles source code, runs tests, and produces software packages that are ready to deploy.
4. AWS S3 : An object storage services in the cloud.

### The Steps
> **Info**
> 1. Make sure you create all resources in the same AWS region.
> 2. Repository, Lambda function, IAM role name should be globally unique.

#### 1. Setup an Empty Repository in CodeCommit
* Go to CodeCommit console, in the side tab, go to Repositories, click Create repository
* Give it a name ; I am using `ngandhi-codecommit-lambda-<programming_language>`
* Click Create

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_1.png){: .align-center}

#### 2. Create an S3 Bucket to Store the CodeBuild Output
* Go to S3 console, click Create bucket
* Give the bucket a name ; I am using ngandhi-test-pipeline-<programming_language>-output-bucket
* Click Create bucket

#### 3. Setup your local repository
Use the following project structure in your local machine (I am using a simple Java project):

{% highlight bash %}
.
├── buildspec.yml
├── pom.xml
└── src
    ├── main
    │   └── java
    │       └── MessageUtil.java
    └── test
        └── java
            └── TestMessageUtil.java
{% endhighlight %}

The content of `MessageUtil.java`:

{% highlight java %}
public class MessageUtil {
    private String message;
    public MessageUtil(String message) {
        this.message = message;
    }
    public String printMessage() {
        System.out.println(message);
        return message;
    }
    public String salutationMessage() {
        message = "Hi!" + message;
        System.out.println(message);
        return message;
    }
}
{% endhighlight %}

The content of `TestMessageUtil.java`:

{% highlight java %}
import org.junit.Test;
import org.junit.Ignore;
import static org.junit.Assert.assertEquals;
public class TestMessageUtil {
    String message = "Robert";
    MessageUtil messageUtil = new MessageUtil(message);
    @Test
    public void testPrintMessage() {
        System.out.println("Inside testPrintMessage()");
        assertEquals(message,messageUtil.printMessage());
    }
    @Test
    public void testSalutationMessage() {
        System.out.println("Inside testSalutationMessage()");
        message = "Hi!" + "Robert";
        assertEquals(message,messageUtil.salutationMessage());
    }
}
{% endhighlight %}

The content of `buildspec.yml`:

{% highlight yaml%}
version: 0.2
phases:
  install:
    runtime-versions:
      java: corretto11
  pre_build:
    commands:
      - echo Nothing to do in the pre_build phase...
  build:
    commands:
      - echo Build started on `date`
      - mvn install
  post_build:
    commands:
      - echo Build completed on `date`
artifacts:
  files:
    - target/messageUtil-1.0.jar # Path to JAR build in the packager container
{% endhighlight %}

The content of `pom.xml`:

{% highlight xml%}
<project xmlns="<http://maven.apache.org/POM/4.0.0>" 
    xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
    xsi:schemaLocation="<http://maven.apache.org/POM/4.0.0> <http://maven.apache.org/maven-v4_0_0.xsd>">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.example</groupId>
  <artifactId>messageUtil</artifactId>
  <version>1.0</version>
  <packaging>jar</packaging>
  <name>Message Utility Java Sample App</name>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>	
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.0</version>
      </plugin>
    </plugins>
  </build>
</project>
{% endhighlight %}

#### 4. Create A Lambda Function That Accepts Triggers from Code Commit
**4.1 Create the Function**
1. Go to Lambda console, click Create function
2. Choose “Author from scratch” → Give the function a name. I am using `ngandhi-codebuild-trigger-on-codecommit-events`
3. Choose the Lambda runtime (I am using Python 3.8)
4. Default for everything else (role for Lambda invocation will be created later)
5. Click "Create function"

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_2.png){: .align-center}

**4.2 The Lambda Handler Code**

> **Info**
> Lambda code editor for Python already contains boto3 library.
Write the following code in the editor and click Deploy:

{% highlight python %}
"""
A simple AWS Lambda function to trigger CodeBuild project 
on push events to CodeCommit repository
"""
import boto3
def lambda_handler(event, context):
    
    cb = boto3.client('codebuild')
    
    build = {
        'projectName': event['Records'][0]['customData'],
        'sourceVersion': event['Records'][0]['codecommit']['references'][0]['commit']
    }
    
    print(f"Starting build for project {build['projectName']} from commit ID {build['sourceVersion']}")
    cb.start_build(**build)
    print("Successfully launched build")
    
    return "Success"
{% endhighlight %}

**4.3 Setup IAM Role and Policy for Lambda**

We will need the following permissions :

1. Write log data to Amazon CloudWatch Logs
2. Invoke the AWS CodeBuild `StartBuild` API

The steps to do that:

* Go to AWS IAM Console and click Create role
* Pick “AWS Service” as trusted entity type and choose Lambda use case, click "Next"
* Attach the following policies and click “Next”:

{% highlight bash %}
AWSCodeBuildDeveloperAccess
AWSLambdaBasicExecutionRole
{% endhighlight %}

* Skip “Tags” if you want to.
* In the "Review" section, give the role a name. I am using `ngandhi-codebuild-trigger-lambda`

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_3.png){: .align-center}

* Click Create role

Attach the newly created role to your Lambda function :

* Go back to your Lambda function dashboard, click Configuration → Permissions
* In the "Execution role" panel, click "Edit"
* In the "Existing role" drop down, choose your previously created role and Click Save

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_4.png){: .align-center}

#### 5. Create A CodeBuild Project
* Go to CodeBuild console and click at "Create build project"
* Give the project a name. I am using ngandhi-codebuild-<programming_language>-project

The following properties to be used :
* Source Provider : "AWS CodeCommit"
* Repository : Your CodeCommit repository name (here: "ngandhi-codecommit-lambda-<programming_language>")
* Reference type : "Branch"

For "Environment":
* Image = "Managed Image"
* Operating system = "Amazon Linux 2"
* Runtime(s) = "Standard"
* Image = "aws/codebuild/amazonlinux2-x86_64-standard:3.0"
* Image version = "Always use the latest image for this runtime version"
* Environment type = "Linux"
* Service role = "New service role (will automatically created)"

For "Buildspec" : Default ("Use a buildspec file")
* Buildspec name = buildspec.yml

For "Artifacts":
* Type: Amazon S3
* Bucket name: Your output bucket name (here: `ngandhi-codebuild-<programming_language>-output`)
* Name: folder name. I am using "output"
* The rest, leave as is
* Click "Update artifacts"
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_5.png){: .align-center}
* Logs : Leave as is

Finally, click Create "build project"
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_6.png){: .align-center}

#### 6. Setup Trigger on CodeCommit Repository
* Go to your Lambda function dashboard and visit "Configuration" → "Triggers"
* Click "Add trigger"
* In the "Trigger configuration", choose "CodeCommit" from the drop down menu. A new setting form will be opened.
* Fill in your repository name under "Repository name" (here : `ngandhi-codecommit-lambda-<programming_language>`)
* Fill trigger name. I am using `ngandhi-trigger-codebuild`
* Under "Events", pick an event that you would like to trigger the build. You can pick multiple options. The default is "All repository events"
* In "Branch names", pick the branch that you would like to be the primary branch to trigger the build. By default it will be "All branches" (represented as empty values)
* In "Custom data — optional", fill it with the name of your CodeBuild project (here : `ngandhi-codebuild-<programming_language>-project`). This field is used to pass the name of the CodeBuild project that we want our Lambda function to invoke. This is represented in our Lambda code under the following struct :

{% highlight bash %}
build = {
        'projectName': event['Records'][0]['customData'],
        'sourceVersion': event['Records'][0]['codecommit']['references'][0]['commit']
}
{% endhighlight %}

* Click "Add" to create the trigger
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_7.png){: .align-center}

#### 7. Reviewing Trigger Configuration
* After CodeCommit trigger has been configured in the Lambda console, visit your CodeCommit repository
* Click "Settings" from the sidebar
* Click "Triggers" from the tab
* You will see your previously configured trigger there
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_8.png){: .align-center}

#### 8. Test the Trigger
* This is the fun part ; go back to your local text editor
* If you don’t have it yet, install `git-remote-codecommit` utility (needs `pip` or `pip3`) on your local

{% highlight bash %}
pip3 install git-remote-codecommit
{% endhighlight %}

* Monitor the installation process until you see a success message similar to the following message:

{% highlight bash %}
"successfully built git-remote-codecommit"
{% endhighlight %}

* Go to your local repositories and initialize git project

{% highlight bash %}
git init
git add --all
git commit -m "My first commit"
{% endhighlight %}

* Set remote origin url to CodeCommit:

{% highlight bash %}
# URL syntax
codecommit://[aws_profile_name]@[repository_name]
# example
git remote add origin codecommit://aws_profile@ngandhi-codecommit-lambda-java
{% endhighlight %}

* Push your local repo to remote branch:

{% highlight bash %}
git push -u origin master
{% endhighlight %}

* Go to your CodeBuild dashboard and see if a new build has kicked off:
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_9.png){: .align-center}

* View the AWS CodeBuild logs from the build:
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_10.png){: .align-center}

#### 9. Check the build output in S3
* Go to your previously created S3 bucket
* The JAR file will be visible inside your bucket (`<bucketName>/<artifactsOutputPath>/<objectKeys>`)
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2021-03-28-simple-cicd-pipeline-in-aws-trigger-jar-build-on-codecommit-events/aws_cc_11.png){: .align-center}

Congratulations, you have build a simple CI/CD pipeline in AWS.

### References
1. [Use AWS CodeCommit & Lambda to Trigger CodeBuild, for Fully-Automated Container Image Builds!](https://www.linkedin.com/pulse/use-aws-codecommit-lambda-trigger-codebuild-container-trevor-sullivan/)
2. [tdmalone/codebuild-trigger](https://github.com/tdmalone/codebuild-trigger/blob/master/lambda_function.py)
3. [How to deploy AWS Lambda functions using CodePipeline](https://medium.com/@sh_in/how-to-deploy-aws-lambda-functions-using-codepipeline-313207f75f53)
4. [Step 2: Create the buildspec file](https://docs.aws.amazon.com/codebuild/latest/userguide/getting-started-create-build-spec-console.html)
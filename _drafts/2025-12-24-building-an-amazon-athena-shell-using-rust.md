---
title:  "Building an Amazon Athena Shell using Rust"
seo_title: "building an amazon athena shell using rust"
seo_description: "building an interactive command-line shell for amazon athena using rust programming language"
date:   2025-12-24 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
  - Rust
  - Athena
  - S3
  - CLI
excerpt: "This post describes how to build an interactive command-line shell for Amazon Athena using Rust programming language..."
toc: true
toc_label: "Table of Contents"
---

## Overview

Amazon Athena is an interactive query service to analyze data in Amazon S3 using SQL-like syntax. I have been using this service on regular basis to check, analyze, and sometimes to pre-process data sitting in my data lake for business consumption. As a CLI person, I found that traversing AWS console to open the query editor is one too many steps for my liking. This post demonstrates my experience building a mini-shell where I can execute the same query directly from my Linux shell.

## Prerequisites

- Rust toolchain (1.70+) installed via [rustup](https://rustup.rs/)
- Valid AWS credentials
- An existing Athena database and S3 bucket for query results

## Setup

### Project Structure

We'll use the following project structure:

{% highlight bash %}
athena-shell
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── client.rs
│   ├── config.rs
│   ├── error.rs
│   ├── main.rs
│   └── repl.rs
└── target
{% endhighlight %}

Here are short explanations for each Rust source file:

- `main.rs` &rarr; Entry point for the application
- `client.rs` &rarr; AWS client wrapper that handles SDK configuration and API calls
- `config.rs` &rarr; Configuration management for AWS profiles
- `error.rs` &rarr; Custom error types and error handling
- `repl.rs` &rarr; Read-Eval-Print Loop implementation

### Dependencies

Then let's set our crate dependencies in Cargo.toml:

{% highlight yaml %}

[dependencies]
aws-config = "1.8"  
aws-runtime = "1.5"  
aws-sdk-athena = "1.97"  
aws-sdk-s3 = "1.119"  
aws-smithy-types = "1.3.5"  
aws-types = "1.3.11"  
directories = "6.0"  
inquire = "0.9.1"  
thiserror = "2.0"  
tokio = { version = "1.48", features = ["full"] }  
{% endhighlight %}

Here are short explanations of the dependencies:

- `aws-config` &rarr; Loads AWS configuration (credentials, region, etc) from env/files
- `aws-runtime` &rarr; Runtime components for AWS SDK functionality
- `aws-sdk-athena` &rarr; Official AWS SDK client for Athena service operations
- `aws-sdk-s3` &rarr; Official AWS SDK client for S3 service operations
- `aws-smithy-types` &rarr; Type definitions for AWS service communication
- `aws-types` &rarr; Common AWS types and utilities
- `directories` &rarr; Cross-platform user directory paths
- `inquire` &rarr; Interactive prompts and input validation for the shell
- `thiserror` &rarr; Derive macros for custom error types
- `tokio` &rarr; Async runtime required for AWS SDK operations

## Implementation

### Creating Custom Error Types

Let's start by defining our custom error types, since our Athena shell will encounter various types of errors such as AWS SDK failures, network issues, to invalid SQL queries and also configuration problems. We are using `thiserror` crate which provides easy-to-use macros to define our error types and it also comes with derive macros that assist in implementing `From<T>`, `Display`, and the `Error` trait.

{% highlight rust %}

// src/error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ShellError {
    #[error("Generic Athena SDK error: {0}")]
    AthenaSdkGenericError(#[from] aws_sdk_athena::Error),

    #[error("Generic S3 SDK error: {0}")]
    S3SdkGenericError(#[from] aws_sdk_s3::Error),

    #[error("Query execution failed for ID: {execution_id}")]
    QueryFailed { execution_id: String },

    #[error("Query timeout after {attempts} attempts")]
    QueryTimeout { attempts: i32 },

    #[error("Missing query execution data")]
    MissingData,

    #[error("Cannot determine home directory")]
    MissingHomeDirectory,

    #[error("Invalid timeout value: {0}")]
    InvalidTimeout(u64),

    #[error("Invalid service: {0}")]
    InvalidService(String),

    #[error("Cannot convert from UTF-8: {0}")]
    FromUtf8ConversionError(#[from] std::string::FromUtf8Error),
}

pub type Result<T> = std::result::Result<T, ShellError>;

{% endhighlight %}

Notice that we create a type alias `Result<T>`. We do this in order to simplifies error handling in our codebase. For example, instead of writing `std::result::Result<T, AthenaError>`, we can just write `Result<T>`. We also provide consistency in our code because now all the functions in our project that return results will use the same error type (`AthenaError`) by default.

### Creating AWS SDK Config Builder 

The AWS SDK for Rust requires a configuration object that contains credentials, region, and other settings to authenticate and route API calls. We'll create a reusable config builder using AWS credential file to set up the necessary configuration to build our Athena client. Then, we'll create an `AwsClient` enum that encapsulates AWS configuration loading and provides a clean interface for instantiating the clients.

Let's create a configuration builder module. We create a function which finds our AWS credentials file by first checking if we set a custom location (set from environment variables), and if not, it will look in the standard place (`$HOME/.aws/credentials`) in Linux system. We also set configuration for timeouts, retries, and protection from slow network throughput.

{% highlight rust %}
// config.rs
use crate::error::{AwsError, Result};
use aws_config::{BehaviorVersion, stalled_stream_protection::StalledStreamProtectionConfig};
use aws_runtime::env_config::file;
use std::path::PathBuf;
use std::time::Duration;

const DEFAULT_CREDENTIAL_PATH_PREFIX: &str = "./aws/credentials";

pub struct ConfigOptions {
    pub retry_attempts: u32,
    pub operation_timeout_multiplier: u64,
    pub attempt_timeout_multiplier: u64,
}

impl Default for ConfigOptions {
    fn default() -> Self {
        Self {
            retry_attempts: 5,
            operation_timeout_multiplier: 3,
            attempt_timeout_multiplier: 9,
        }
    }
}

pub fn get_credentials_path() -> Result<PathBuf> {
    // Check env variable for the path
    if let Ok(path) = std::env::var("SSO_CREDENTIAL_PATH") {
        return Ok(PathBuf::from(path));
    }
    // Fallback to home directory
    let home = directories::BaseDirs::new().ok_or(AwsError::MissingHomeDirectory)?;
    let home_dir = home.home_dir();

    Ok(home_dir.join(DEFAULT_CREDENTIAL_PATH_PREFIX))
}

pub async fn build_config(
    profile: &str,
    timeout: u64,
    no_stall_protection: bool,
) -> Result<aws_types::SdkConfig> {
    if timeout == 0 {
        return Err(AwsError::InvalidTimeout(timeout));
    }
    let config_options = ConfigOptions::default();
    let retry_config = aws_smithy_types::retry::RetryConfig::standard()
        .with_initial_backoff(Duration::from_secs(1))
        .with_max_backoff(Duration::from_secs(5))
        .with_max_attempts(config_options.retry_attempts);

    let timeout_config = aws_config::timeout::TimeoutConfig::builder()
        .connect_timeout(Duration::from_secs(timeout))
        .operation_timeout(Duration::from_secs(
            timeout * config_options.operation_timeout_multiplier,
        ))
        .operation_attempt_timeout(Duration::from_secs(
            timeout * config_options.attempt_timeout_multiplier,
        ))
        .build();

    let profile_file = file::EnvConfigFiles::builder()
        .with_file(
            file::EnvConfigFileKind::Credentials,
            get_credentials_path()?,
        )
        .build();

    let mut config_builder = aws_config::defaults(BehaviorVersion::latest())
        .profile_files(profile_file)
        .profile_name(profile)
        .timeout_config(timeout_config)
        .retry_config(retry_config);

    if no_stall_protection {
        config_builder =
            config_builder.stalled_stream_protection(StalledStreamProtectionConfig::disabled());
    }

    Ok(config_builder.load().await)
}
{% endhighlight %}

### Creating AWS Client Builder

There are 2 AWS services needed for the application: Athena and S3. We'll use an enum `AwsClient` which implements a single entry point for client creation based on service name:

{% highlight rust %}
// client.rs
use crate::config::build_config;
use crate::error::AwsError;

use aws_sdk_athena::Client as AthenaClient;
use aws_sdk_s3::Client as S3Client;

pub enum AwsClient {
    Athena(AthenaClient),
    S3(S3Client),
}

impl AwsClient {
    pub async fn new(
        service: &str,
        profile: &str,
        timeout: u64,
        no_stall_protection: bool,
    ) -> Result<Self, AwsError> {
        let config = build_config(profile, timeout, no_stall_protection).await?;

        match service.to_lowercase().as_str() {
            "athena" => Ok(Self::Athena(AthenaClient::new(&config))),
            "s3" => Ok(Self::S3(S3Client::new(&config))),
            _ => Err(AwsError::InvalidService(service.to_string())),
        }
    }
}

...
{% endhighlight %}

TBC

### Creating Basic REPL

Next, we scaffold a basic Read-Eval-Print-Loop where users can input commands to be executed towards Athena Services. Since we are working with async runtime, we have to use the non-blocking IO to handle `stdin` and `SIGINT` with the combination of `tokio::io` traits and `tokio::select!` macro.

This basic REPL has one quirk: If users cancel their input during multi line input session, they need to press ENTER twice to return to the normal prompt. I suspect this is because of the imperfect signal handling (when receiving Ctrl-C input mixed with the ongoing stdin) in my `tokio::select!` block. Let me know if you have suggestions to fix this issue!  

We will add AWS functionalities later.

{% highlight rust %}
use crate::error::Result;
use std::io::Write;
use std::io::{self};
use tokio::io::{AsyncBufReadExt, AsyncReadExt};

pub struct Repl {
    prompt: String, // prompt chars
    line_buf: String,
    input_buf: Vec<String>, // buffer to accumulate stdin input
    // holds complete command for processing
    is_in_multiline: bool, // state management of the input
}

impl Repl {
    pub fn new(profile: &str) -> Self {
        Repl {
            prompt: format!("{}> ", profile),
            line_buf: String::new(),
            input_buf: Vec::new(),
            is_in_multiline: false,
        }
    }

    pub fn print_header(&self) {
        println!(
            r#"
╔═══════════════════════════════════════╗
║           ATHENA SHELL                ║
║     AWS Query Interface v0.1.0        ║
╚═══════════════════════════════════════╝

AWS Athena Query Interface - v0.1.0
Type 'exit;' to quit
        "#
        )
    }

    pub async fn read_line(&self) -> Result<String> {
        // Create the reader
        let mut reader = tokio::io::BufReader::new(tokio::io::stdin());
        let mut buffer = Vec::new();
        let _fut = reader.read_until(b'\n', &mut buffer).await;
        Ok(String::from_utf8(buffer)?)
    }

    pub async fn repl_loop(&mut self) -> Result<()> {
        // Print header when first time entering the shell
        self.print_header();

        // Begin REPL loop
        loop {
            // Read line from stdin and flush immediately to stdout.
            // is_in_multiline is a state management flag which is true when input buffer is not empty and not terminated with a semicolon.
            // In this case it creates a new line feed prefixed with a pipe "| " char.
            // If input buffer is terminated with a semicolon, then close the input stream and process the command.
            // If return char (\n or \r\n) is fed to the buffer, then clear the input buffer.
            //
            // If the buffer is processing multi-line input, change the prompt into "|"
            if self.is_in_multiline {
                print!("| ");
            } else {
                print!("{}", self.prompt);
            }

            // Flush the output to ensure the prompt is displayed immediately
            io::stdout().flush().unwrap();

            // Wait for either stdin or sigint
            tokio::select! {
                linebuf = self.read_line() => {
                    let tmp = linebuf.unwrap();
                    self.input_buf.push(tmp.clone());
                    if tmp.contains(";") {
                        // Command complete, move and process
                        // self.line_buf = std::mem::take(&mut self.input_buf);
                        self.is_in_multiline = false;
                        // Sanitize the line_buf string logic
                        let command = self
                            .input_buf
                            .iter()
                            .map(|s| s.replace('\n', "").trim().to_string())
                            .filter(|s| !s.is_empty())
                            .collect::<Vec<String>>()
                            .join(" ")
                            .replace(" ;", ";");
                        if command == "exit;" {
                            println!("Exiting Athena Shell!");
                            return Ok(());
                        }
                        // Execute command
                        println!("{}", command); // placeholder for command execution
                        // Empty the line buffer
                        // input buffer should already empty after the take()
                        self.input_buf.clear();
                    } else if tmp == "\n" || tmp == "\r\n" { // checks for newline-only input
                        if !self.input_buf.is_empty() {
                            self.is_in_multiline = true; // keep printing "|" as prompt if input_buf contains data
                        } else {
                            self.input_buf.clear(); // regenerate prompt for newline-only input
                            self.is_in_multiline = false;
                        }
                    } else {
                        self.is_in_multiline = true;
                    }
                }
                _ = tokio::signal::ctrl_c() => {
                    println!("\nCtrl+C detected, cancelling current input...");
                    println!("Press ENTER twice to return to the prompt");
                    self.input_buf.clear();
                    self.line_buf.clear();
                    self.is_in_multiline = false;
                    // consume pending input
                    let _ = tokio::io::stdin().read(&mut [0u8; 1024]).await;
                    tokio::time::sleep(tokio::time::Duration::from_millis(250)).await;
                }
            }
        }
    }
}
{% endhighlight %}

## Usage Examples

TBC

## Conclusion

TBC

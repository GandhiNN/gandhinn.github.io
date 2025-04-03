---
title:  "Demystifying AWS CLI Login Flow"
seo_title: "demystifying aws cli login flow"
seo_description: "demystifying aws cli login flow"
date:   2025-04-04 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
excerpt: "This post details the AWS CLI login flow."
toc: true
toc_label: "Table of Contents"
---
### Overview
<TBC>

### Description
AWS CLI invocation:
1. (Primary) AWS Config file -> `~/.aws/config`:
	a. check profile and its sso profile & role name.
	b. check cli cache file -> `~/.aws/cli/cache/<cache>.json`
2. (Fallback) AWS Credentials file -> `~/.aws/credentials`:
	a. check if AWS access key ID and AWS secret access key are valid, i.e. correctness
	b. check if AWS session token is valid (i.e. correctness and "fresh")

OIDC SSO:
1. Authenticate and register device to SSO start-url to get its token.
2. Create SSO token cache for OIDC connection -> `~/.aws/sso/cache/<cachefile>.json`
3. Create AWS config file, containing the AWS profile name and its session -> `~/.aws/config`
4. Create AWS credential file -> `~/.aws/credentials`

### Overview
<TBC>

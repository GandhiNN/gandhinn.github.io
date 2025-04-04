---
title:  "Demystifying AWS CLI and AWS SSO Configuration"
seo_title: "demystifying aws cli and aws sso configuration"
seo_description: "demystifying aws cli and aws sso configuration"
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
	b. (might be invalid) check cli cache file -> `~/.aws/cli/cache/<cache>.json`
		i. content of this file can be the same with `~/.aws/credentials`
		ii. but make sure it is using CamelCase formatting for the keys!
			- use the `#[serde(rename_all = "PascalCase")]` directive for the struct.
			- and use the `#[serde(rename = "PascalCase")]` for the attribute
	c. check the token cache -> `~/.aws/sso/cache/<filename>.json`
		i. note: <filename> must be the sha1 hash version of the session name registered under the profile you are using.
		ii. for example, suppose that you have the following entry in your `~/.aws/config` file, and let's say you want to invoke the following AWS CLI command: `aws s3 ls --profile icloud-dev`:
			```
			[sso-session sso-d-9367052e24]
			sso_start_url=https://d-9367052e24.awsapps.com/start/#
			sso_registration_scopes=sso:account:access
			sso_region=eu-west-1

			[profile icloud-dev]
			sso_account_id=291751643970
			sso_session=sso-d-9367052e24 --> the cache filename must be the sha1 hash version of sso_session name!
			sso_role_name=tlz_developer
			region=eu-west-1
			```  

			Then you need to have the cache file following this syntax = `~/.aws/sso/cache/$(sha1::from("sso-d-9367052e24")).json` 
			
			i.e., `~/.aws/sso/cache/7a7d2bddd4a31e7b4be4c83fdbbe1af869b642f8.json`
2. (Fallback) AWS Credentials file -> `~/.aws/credentials`:
	a. check if AWS access key ID and AWS secret access key are valid, i.e. correctness
	b. check if AWS session token is valid (i.e. correctness and "fresh")

OIDC SSO:
1. Authenticate and register device to SSO start-url to get its token.
2. Create SSO token cache for OIDC connection -> `~/.aws/sso/cache/<cachefile>.json`
3. Create AWS config file, containing the AWS profile name and its session -> `~/.aws/config`
4. Create AWS credential file -> `~/.aws/credentials`

### Conclusion
<TBC>

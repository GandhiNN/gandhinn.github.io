---
title:  "Troubleshooting Rust AWS SDK Client Zero-Throughput Issue"
seo_title: "troubleshooting rust aws sdk client zero-throughput issue"
seo_description: "troubleshooting rust aws sdk client zero-throughput issue"
date:   2025-02-22 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
excerpt: "This post will describe the workaround to solve an issue in Rust AWS SDK client dispatch on a slow network."
---
### Background
I am developing a CLI application which has a function to register my client to AWS using Single-Sign-On on top of OIDC (OpenID Connect) protocol. Basically what it does is to interact with the AWS SSO portal to fetch all the required credentials information e.g account info, AWS access key pair, session token. 

On high level. the program will register the client id to the portal, retrieve the session token and role credentials, and finally writing those information into a file that can be loaded by my application every time it invokes any SDK APIs between a certain timeframe. 

To do these operations, I am creating a `SsoAccessTokenProvider` struct which loads an `SdkConfig` object to further build the API client which has all the functionalities to retrieve the required information to build an SSO login information:

{% highlight rust %}
....

let config: aws_types::SdkConfig = aws_config::SdkConfig::builder()
    .region(Region::new(sso_default_config.region.clone()))
    .behavior_version(BehaviorVersion::latest())
    .build();

...

pub struct SsoAccessTokenProvider {
    sso_session_name: String,
    client: Client,
    cache: AccessTokenCache,
}

impl SsoAccessTokenProvider {
    pub fn new(config: &SdkConfig, sso_session_name: &str, config_dir: &Path) -> Result<Self> {
        let sso_cache_dir = config_dir.join("sso").join("cache");
        info!(
            "Checking if SSO cache directory: {} exists...",
            sso_cache_dir.display()
        );
        if !sso_cache_dir.exists() {
            info!(
                "SSO cache directory: {} does not exists! creating...",
                sso_cache_dir.display()
            );
            fs::create_dir_all(&sso_cache_dir)?;
        } else {
            info!("SSO cache directory {} exists...", sso_cache_dir.display())
        }

        Ok(Self {
            sso_session_name: String::from(sso_session_name),
            client: Client::new(config),
            cache: AccessTokenCache::new(sso_session_name, sso_cache_dir.as_path()),
        })
    }

    pub async fn get_access_token(&self, start_url: &str) -> Result<AccessToken> {
        info!("Checking cached token...");
        let cached_token_option = self.cache.get_cached_token();
        println!("{:?}", cached_token_option);

        match cached_token_option {
            Ok(cached_token) => {
                if cached_token.is_expired() {
                    info!("Cached token expired, getting a new one...");
                    self.get_new_token(start_url).await
                } else {
                    info!("Cached token still valid, refreshing...");
                    self.refresh_token(cached_token).await
                }
            }
            Err(_) => self.get_new_token(start_url).await,
        }
    }  
{% endhighlight %}

And the following is the associated function of the `SsoAccessTokenProvider` struct which are used to interact with SSO portal to retrieve the token:

{% highlight rust %}

    ...

    async fn get_new_token(&self, start_url: &str) -> Result<AccessToken> {
        info!("Registering device client...");
        let device_client = self.register_device_client().await;
        info!("Authenticating device client...");
        self.authenticate(start_url, device_client?).await
    }

    ...
}
{% endhighlight %}

I had no issue using the application when I was connected to my office's internet, but I started to see intermittent failures when I was connected to slower networks i.e. at my home or when tethered to my mobile network.

The returned error message look like this:

{% highlight bash %}
Err(dispatch failure
 
Caused by:
    0: timeout
    1: minimum throughput was specified at 1 B/s, but throughput of 0 B/s was observed)carmakers = ["Toyota", "Mercedes Benz", "BMW", "Honda"]
{% endhighlight %}

Upon further investigation by enabling debug tracing (`RUST_LOG=debug cargo run <args>`), I found the following error logs:

{% highlight bash %}
2025-02-21T22:45:04.472644Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op:try_attempt: aws_smithy_runtime::client::http::body::minimum_throughput: current throughput: 0 B/s is below minimum: 1 B/s
2025-02-21T22:45:04.472767Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op:try_attempt: aws_smithy_runtime::client::http::body::minimum_throughput: current throughput: 0 B/s is below minimum: 1 B/s
2025-02-21T22:45:04.497850Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op:try_attempt: aws_smithy_runtime::client::http::body::minimum_throughput: current throughput: 0 B/s is below minimum: 1 B/s
2025-02-21T22:45:04.497932Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op:try_attempt: aws_smithy_runtime::client::http::body::minimum_throughput: grace period ended; timing out request
2025-02-21T22:45:04.498111Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op:try_attempt: aws_smithy_runtime::client::orchestrator: encountered orchestrator error; halting
2025-02-21T22:45:04.498299Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op: aws_smithy_runtime::client::retries::strategy::standard: not retrying because we are out of attempts attempts=1 max_attempts=1
2025-02-21T22:45:04.498339Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=9975118}:try_op: aws_smithy_runtime::client::orchestrator: a retry is either unnecessary or not possible, exiting attempt loop
{% endhighlight %}

### The SDK Guardrails for Network Issues
After reading the [official documentation](https://docs.aws.amazon.com/sdk-for-rust/latest/dg/timeouts.html), I learned that the AWS SDK for Rust has several settings to handle **API request timeouts** and **protect against stalled data streams** which may occur in the network, especially in the slower ones.

#### API Request Timeouts
The SDK provides a default timeout to protect agains transient issues in the network that could cause request attempts to take a long time or fail completely, ensuring the applications can fail fast and behave optimally.

The default timeout of the SDK is set to _3.1 seconds_.

#### Stalled Stream Protection
A stalled stream is an upload or download stream that produces no data for longer than a configured grace period, preventing the applications from hanging indefinitely and never making progress. 

The default behavior of the SDK is to enable stalled stream protection for both uploads and downloads and looks for at least _1 byte/sec of activity_ with _grace period of 20 seconds_.

### The Solution
From my perspective, the default behavior of the two guardrails are too aggressive, considering that the network quality in the places where I live can vary greatly. My solution is to:

1. Set a big-enough value for the API timeout.
2. Disable the stalled stream protection in the client.

Here's how I did it in my code:

{% highlight rust %}
...
    // Set default timeout config to 10000 seconds
    let default_timeout_config = TimeoutConfig::builder()
        .connect_timeout(time::Duration::from_secs(10000))
        .operation_timeout(time::Duration::from_secs(10000 * 3))
        .operation_attempt_timeout(time::Duration::from_secs(10000 * 3 * 3))
        .build();

    // Build config by using the timeout and disable the stalled stream protection
    let config: aws_types::SdkConfig = aws_config::SdkConfig::builder()
    .region(Region::new(sso_default_config.region.clone()))
    .behavior_version(BehaviorVersion::latest())
    .stalled_stream_protection(StalledStreamProtectionConfig::disabled())
    .timeout_config(default_timeout_config)
    .build();
...
{% endhighlight %}

Here's the result after using the adjusted configuration:

{% highlight bash %}
2025-02-22T00:30:45.382592Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:apply_configuration: aws_smithy_runtime::client::orchestrator: timeout settings for this operation: TimeoutConfig { connect_timeout: Set(10000s), read_timeout: Disabled, operation_timeout: Set(30000s), operation_attempt_timeout: Set(90000s) }
2025-02-22T00:30:45.382871Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op: aws_smithy_runtime_api::client::interceptors::context: entering 'serialization' phase
2025-02-22T00:30:45.383089Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op: aws_smithy_runtime_api::client::interceptors::context: entering 'before transmit' phase
2025-02-22T00:30:45.383244Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op: aws_smithy_runtime::client::retries::strategy::standard: no client rate limiter configured, so no token is required for the initial request.
2025-02-22T00:30:45.383280Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op: aws_smithy_runtime::client::orchestrator: retry strategy has OKed initial request
2025-02-22T00:30:45.383347Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op: aws_smithy_runtime::client::orchestrator: beginning attempt #1
2025-02-22T00:30:45.383467Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op:try_attempt: aws_smithy_runtime::client::orchestrator::endpoints: resolving endpoint endpoint_params=EndpointResolverParams(TypeErasedBox[!Clone]:Params { region: Some("eu-west-1"), use_dual_stack: false, use_fips: false, endpoint: None }) endpoint_prefix=None
2025-02-22T00:30:45.383569Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op:try_attempt: aws_smithy_runtime::client::orchestrator::endpoints: will use endpoint Endpoint { url: "https://oidc.eu-west-1.amazonaws.com", headers: {}, properties: {} }
2025-02-22T00:30:45.383867Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op:try_attempt: aws_smithy_runtime::client::identity::cache::lazy: loaded identity from cache buffer_time=10s cached_expiration=None now=SystemTime { tv_sec: 1740184245, tv_nsec: 383829579 }
2025-02-22T00:30:45.384029Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op:try_attempt: aws_smithy_runtime_api::client::interceptors::context: entering 'transmit' phase
2025-02-22T00:30:45.384142Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op:try_attempt: aws_smithy_runtime::client::http::body::minimum_throughput: no minimum upload throughput checks
2025-02-22T00:30:45.384208Z DEBUG invoke{service=ssooidc operation=CreateToken sdk_invocation_id=2200328}:try_op:try_attempt: hyper::client::pool: reuse idle connection for ("https", oidc.eu-west-1.amazonaws.com)
2025-02-22T00:30:45.384703Z DEBUG invoke{service=ssooidc operation=RegisterClient sdk_invocation_id=6539698}:try_op:try_attempt:Connection{peer=Client}: h2::codec::framed_write: send frame=Headers { stream_id: StreamId(69), flags: (0x4: END_HEADERS) }
2025-02-22T00:30:45.384803Z DEBUG invoke{service=ssooidc operation=RegisterClient sdk_invocation_id=6539698}:try_op:try_attempt:Connection{peer=Client}: h2::codec::framed_write: send frame=Data { stream_id: StreamId(69), flags: (0x1: END_STREAM) }
2025-02-22T00:30:45.997064Z DEBUG invoke{service=ssooidc operation=RegisterClient sdk_invocation_id=6539698}:try_op:try_attempt:Connection{peer=Client}: h2::codec::framed_read: received frame=Headers { stream_id: StreamId(69), flags: (0x4: END_HEADERS) }
{% endhighlight %}

From the above logs, we can see that the SDK set its timeout to 10,000 seconds and it disabled the througpuht check and was able to interact with the OIDC portal. 

### Conclusion
AWS SDK for Rust provides advanced settings that we can use to fine-tune our application's behavior when interacting with the networks. These settings are provided in every AWS SDK crates which gives greater control for the developers to further fine-tune their applications if they need tighter performance. But at the same time, we need to be extra careful when playing with the settings so they will not give detrimental side-effects.
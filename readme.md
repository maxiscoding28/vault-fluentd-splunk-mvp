# Sending Logs from Vault to FluentD and Splunk

## Introduction

### Expected Outcome
System logs from a running Vault server will be sent to Fluentd and from Fluentd to Splunk.
**Note: This tutorial describes a method for achieving a minimum viable solution and should not be used in production.**

### Prerequisites (if applicable)
- [Docker Desktop](https://docs.docker.com/get-docker/)
-  [Vault docker image](https://registry.hub.docker.com/_/vault/)
-  [Fluentd docker image](https://hub.docker.com/r/fluent/fluentd/)
- [Splunk Enterprise](https://www.splunk.com/en_us/products/splunk-enterprise.html)

## Procedure

### Setting up Splunk
1. Download and install Splunk enterprise. Open the Splunk App (should open on localhost:8000) and naviage to Settings > Data > Indexes
2. Create a New Index with the following values:
		- Index Name: `vault-sys-logs`
		- Index Data Type: `Events`
		- Leave all other vaules as default.
3. Go to Settings > Data > Data inputs. Click HTTP Event Collector.
	- Create an token with the following vaules:
		- Name: `Vault Sys Logs`
		- Allowed Indexes: `vault-sys-logs`
		- Leave all other vaules as default.
		- Copy the token and save it somewhere safe.
4. Go to Settings > Data > Data Inputs > HTTP Event Collector
	-  Click Global Settings. Uncheck Enable SSL. Click Save.

### Setting up Fluentd
1.  Create a network for the FluentD and Vault contains to communicate.
   ```
   docker network create vault-fluentd-net --subnet 192.168.211.0/24
```
2. We'll use the [fluent-plugin-splunk-hec](https://github.com/splunk/fluent-plugin-splunk-hec) plugin to send logs from fluentd to Splunk. To do this, we'll need to install in on the fluentd docker image before running the container. Build a docker image using the following Dockerfile *(note I am using `:edge-debian` because it plays nice with my arm64 machine)*. Dockerfile:
```
# Dockerfile

FROM fluent/fluentd:edge-debian
USER root
RUN buildDeps="sudo make gcc g++ libc-dev" \
&& apt-get update \
&& apt-get install -y --no-install-recommends $buildDeps \
&& sudo gem install fluent-plugin-splunk-hec \
&& sudo gem sources --clear-all \
&& SUDO_FORCE_REMOVE=yes \
apt-get purge -y --auto-remove \
-o APT::AutoRemove::RecommendsImportant=false \
$buildDeps \
&& rm -rf /var/lib/apt/lists/* \
&& rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem
```
-
```
docker build -f fluentd-splunkhec.Dockerfile -t fluentd-splunkhec .
```
3. Create a `fluentd.conf` file:
```
<system>
log_level trace
</system>

<source>
@type forward
port 24224
bind 0.0.0.0
</source>

# This <match> outputs logs to syslog in the fluentd container.
# This is for debugging purposes and can be uncommented
# for troubleshooting purposes
<match vault>
@type stdout
</match>

<match vault>
@type splunk_hec
protocol http
hec_host host.docker.internal
hec_port 8088
hec_token $SPLUNK_HEC_TOKEN_GOES_HERE <=!!
index vault-sys-logs
source vault
</match>
```
4. Run the container on the network your created using the .conf file you created:
```
docker run --name fluentd-splunkhec \
--net vault-fluentd-net --ip 192.168.211.2 \
-v <PATH TO DIRECTORY WITH CONF FILE>:/fluentd/etc \
fluentd-splunkhec -c /fluentd/etc/fluentd.conf
```

### Setting Up Vault
1. In a new terminal run the Vault docker image using the following options:
```
docker run --cap-add=IPC_LOCK --name=vault \
--log-driver=fluentd --log-opt fluentd-address=192.168.211.2:24224 --log-opt tag="{{.Name}}" \
```

### Wrapping Up
Wait a few minutes and you should start seeing system logs in splunk at `index=vault-sys-logs`.

If you still aren't seeing logs, add the following match above (the existing match) in your conf file. This will output logs to stdout on the fluentd container and allow you to troubleshoot.
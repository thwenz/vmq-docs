# Not a tuning guide

## General relation to OS configuration values

You need to know about and configure a couple of Operating System and Erlang VM configs to operate VerneMQ efficiently. First, make sure you have set appropriate OS file limits according to our [guide here](change-open-file-limits.md). Second, when you run into performance problems, don't forget to check the [settings in the `vernemq.conf` file](../configuration/introduction.md). \(Can't open more than 10k connections? Well, is the listener configured to open more than 10k?\)

## TCP buffer sizes

This is the number one topic to look at, if you need to keep an eye on RAM usage.

Context: All network I/O in Erlang uses an internal driver. This driver will allocate and handle an internal application side buffer for every TCP connection. The default size of these buffers will determine your overall RAM use in VerneMQ. The sndbuf and recbuf of the TCP socket will not count towards VerneMQ RAM, but will be used by the Linux Kernel.

VerneMQ calculates the buffer size from the OS level TCP send and receive buffers:

`val(buffer) >= max(val(sndbuf),val(recbuf))`

Those values correspond to `net.ipv4.tcp_wmem` and `net.ipv4.tcp_rmem` in your OS's sysctl configuration. One way to minimize RAM usage is therefore to configure those settings \(Debian example\):

```bash
sudo sysctl -w net.ipv4.tcp_rmem="4096 16384 32768"
sudo sysctl -w net.ipv4.tcp_wmem="4096 16384 32768"

# Nope, these values are not recommendations!
# You really need to decide yourself.
```

This would result in a 32KB application buffer for every connection. 	

If your VerneMQ use case requires the use of different TCP buffer optimisations \(per groups of clients for instance\) you will have to make sure the that the Linux OS buffer configuration, namely `net.ipv4.tcp_wmem` and `net.ipv4.tcp_rmem`allows for this kind of flexibility, allowing for small TCP buffers and big TCP buffers at the same time.

```text
Example 1 (from Linux OS config):
net.ipv4.tcp_rmem="4096 16384 32768"
net.ipv4.tcp_wmem="4096 16384 65536"
```

Example 1 above would allow VerneMQ to allocate minimal TCP read and write buffers of 4KB in the Linux Kernel, a max read buffer of 32KB in the kernel, and a max write buffer of 65KB in the kernel. VerneMQ itself would set its own internal per connection buffer to 65KB in addition.

What we just described is VerneMQ automatically configuring TCP read and write buffers and internal buffers, deriving their values from OS settings.

There are multiple additional ways to configure TCP buffers described below:

#### Setting TCP buffer sizes globally within VerneMQ:

If VerneMQ finds an `advanced.config` file, it will use the buffer sizes you have configured there for all it’s TCP listeners \(and the TCP connections accepted by those listeners\), except the Erlang distribution listeners within the cluster.

\(You'll find an example in the section below on the `advanced.config` [file](https://docs.vernemq.com/misc/not-a-tuning-guide#the-advanced-config-file)\)

#### Per protocol \(since 1.8.0\):

If VerneMQ finds a per protocol configuration \(`listener.tcp.buffer_sizes`\) in the `vernemq.conf` file, it will use those buffer sizes for the specific protocol. \(currently only MQTT or MQTTS. Support for WS/WSS/HTTP/VMQ listeners is on the roadmap\).

For `listener.tcp.buffer_sizes` you’ll always have to state 3 values in bytes: the TCP receive buffer \(recbuf\), the TCP send buffer \(sndbuf\), and the internal application side buffer \(buffer\). You should set “buffer” \(the 3rd value\) to`val(buffer) >= max(val(sndbuf),val(recbuf))`

```text
Example 2 (vernemq.conf):
listener.tcp.buffer_sizes = 4096,16384,32768
```

#### Per listener \(since 1.8.0\):

 If VerneMQ finds per listener config values \(`listener.tcp.my_listener.buffer_sizes`\), it will use those buffer sizes for all connections setup by that specific listener. This is the most useful approach if you want to set specific different buffer sizes, like huge send buffers for listeners that accept massive consumers. \(consumers with high expected message throughput\).

You would then configure a different listener for those massive consumers, and by that have the option to fine tune the TCP buffer sizes.

```text
Example 3: (vernemq.conf)
listener.tcp.my_listener.buffer_sizes = 4096,16384,32768
```

For `listener.tcp.my_listener.buffer_sizes` you’ll always have to state 3 values in bytes: the TCP receive buffer \(recbuf\), the TCP send buffer \(sndbuf\), and an internal application side buffer \(buffer\). You should set “buffer” \(the 3rd value\) to`val(buffer) >= max(val(sndbuf),val(recbuf))`

#### VerneMQ per single ClientID/or TCP connection:

This scenario would be possible with a plugin. 

## The advanced.config file

The `advanced.config` file is a supplementary configuration file that sits in the same directory as the `vernemq.conf`. You can set additional config values for any of the OTP applications that are part of a VerneMQ release. To just configure the TCP buffer size manually, you can create an `advanced.config` file:

```erlang
[{vmq_server, [
          {tcp_listen_options,
          [{sndbuf, 4096},
           {recbuf, 4096}]}]}].
```

## The vm.args file

For very advanced & custom configurations, you can add a `vm.args` file to the same directory where the `vernemq.conf` file is located. Its purpose is to configure parameters for the Erlang Virtual Machine. This will override any Erlang specific parameters your might have configured via the `vernemq.conf`. Normally, VerneMQ auto-generates a vm.args file for every boot in `/var/lib/vernemq/generated.configs/` \(Debian package example\) from `vernemq.conf` and other potential configuration sources.

{% hint style="info" %}
A manually generated `vm.args` is not supplementary, it is a full replacement of the auto-generated file! Keep that in mind. An easy way to go about this, is by copying and extending the auto-generated file.
{% endhint %}

This is how a `vm.args` might look like:

```erlang
+P 256000
-env ERL_MAX_ETS_TABLES 256000
-env ERL_CRASH_DUMP /erl_crash.dump
-env ERL_FULLSWEEP_AFTER 0
-env ERL_MAX_PORTS 262144
+A 64
-setcookie vmq  # Important: Use your own private cookie... 
-name VerneMQ@127.0.0.1
+K true
+sbt db
+sbwt very_long
+swt very_low
+sub true
+Mulmbcs 32767
+Mumbcgs 1
+Musmbcs 2047
# Nope, these values are not recommendations!
# You really need to decide yourself, again ;)
```

## A note on TLS

Using TLS will of course increase the CPU load during connection setup. Latencies in message delivery will be increased, and your overall message throughput per second will be lower.

TLS will require considerably more RAM. Instead of 2 Erlang processes per connection, TLS will use 3. You'll have a session process, a queue process, and a TLS handler process that can encapsulate quite a big state \(&gt; 30KB\).

Erlang/OTP uses its own TLS implementation, only using OpenSSL for crypto, but not connection handling. For situations with high connection setup rate or overall high connection churn rate, the Erlang TLS implementation might be too slow. On the other hand, Erlang TLS gives you great concurrency & fault isolation for long-lived connections.

Some Erlang deployments terminate SSL/TLS with an external component or with a load balancer component. Do some testing & try to find out what works best for you.

{% hint style="info" %}
The Erlang TLS implementation is rather picky on certificate chains & formats. Don't give up, if you encounter errors first. On Linux, you can find out more with the `openssl s_client` command quickly.
{% endhint %}


---
layout: "docs"
page_title: "Agent Configuration"
sidebar_current: "docs-configuration"
description: |-
  Learn about the configuration options available for the Nomad agent.
---

# Nomad Configuration

Nomad agents have a variety of parameters that can be specified via
configuration files or command-line flags. Configuration files are written in
[HCL][hcl]. Nomad can read and combine parameters from multiple configuration
files or directories to configure the Nomad agent.

## Load Order and Merging

The Nomad agent supports multiple configuration files, which can be provided
using the `-config` CLI flag. The flag can accept either a file or folder. In
the case of a folder, any `.hcl` and `.json` files in the folder will be loaded
and merged in lexicographical order. Directories are not loaded recursively.

For example:

```shell
$ nomad agent -config=server.conf -config=/etc/nomad -config=extra.json
```

This will load configuration from `server.conf`, from `.hcl` and `.json` files
under `/etc/nomad`, and finally from `extra.json`.

As each file is processed, its contents are merged into the existing
configuration. When merging, any non-empty values from the latest config file
will append or replace parameters in the current configuration. An empty value
means `""` for strings, `0` for integer or float values, and `false` for
booleans. Since empty values are ignored you cannot disable a parameter like
`server` mode once you've enabled it.

Here is an example Nomad agent configuration that runs in both client and server
mode.

```hcl
data_dir  = "/var/lib/nomad"

bind_addr = "0.0.0.0" # the default

advertise {
  # Defaults to the first private IP address.
  http = "1.2.3.4"
  rpc  = "1.2.3.4"
  serf = "1.2.3.4:5648" # non-default ports may be specified
}

server {
  enabled          = true
  bootstrap_expect = 3
}

client {
  enabled       = true
  network_speed = 10
}

plugin "raw_exec" {
  config {
    enabled = true
  }
}

consul {
  address = "1.2.3.4:8500"
}

```

~> Note that it is strongly recommended **not** to operate a node as both
`client` and `server`, although this is supported to simplify development and
testing.

## General Parameters

- `acl` `(`[`ACL`]`: nil)` - Specifies configuration which is specific to ACLs.

- `addresses` `(Addresses: see below)` - Specifies the bind address for
  individual network services. Any values configured in this stanza take
  precedence over the default [bind_addr](#bind_addr).
  The values support [go-sockaddr/template format][go-sockaddr/template].

  - `http` - The address the HTTP server is bound to. This is the most common
    bind address to change.

  - `rpc` - The address to bind the internal RPC interfaces to. Should be
    exposed only to other cluster members if possible.

  - `serf` - The address used to bind the gossip layer to. Both a TCP and UDP
    listener will be exposed on this address. Should be exposed only to other
    cluster members if possible.

- `advertise` `(Advertise: see below)` - Specifies the advertise address for
  individual network services. This can be used to advertise a different address
  to the peers of a server or a client node to support more complex network
  configurations such as NAT. This configuration is optional, and defaults to
  the bind address of the specific network service if it is not provided. Any
  values configured in this stanza take precedence over the default
  [bind_addr](#bind_addr).

    If the bind address is `0.0.0.0` then the address
  private IP found is advertised. You may advertise an alternate port as well.
  The values support [go-sockaddr/template format][go-sockaddr/template].

  - `http` - The address to advertise for the HTTP interface. This should be
    reachable by all the nodes from which end users are going to use the Nomad
    CLI tools.

  - `rpc` - The address advertised to Nomad client nodes. This allows
    advertising a different RPC address than is used by Nomad Servers such that
    the clients can connect to the Nomad servers if they are behind a NAT.

  - `serf` - The address advertised for the gossip layer. This address must be
    reachable from all server nodes. It is not required that clients can reach
    this address. Nomad servers will communicate to each other over RPC using
    the advertised Serf IP and advertised RPC Port.

- `bind_addr` `(string: "0.0.0.0")` - Specifies which address the Nomad
  agent should bind to for network services, including the HTTP interface as
  well as the internal gossip protocol and RPC mechanism. This should be
  specified in IP format, and can be used to easily bind all network services to
  the same address. It is also possible to bind the individual services to
  different addresses using the [addresses](#addresses) configuration option.
  Dev mode (`-dev`) defaults to localhost.
  The value supports [go-sockaddr/template format][go-sockaddr/template].

- `client` `(`[`Client`]`: nil)` - Specifies configuration which is specific
  to the Nomad client.

- `consul` `(`[`Consul`]`: nil)` - Specifies configuration for
  connecting to Consul.

- `datacenter` `(string: "dc1")` - Specifies the data center of the local agent.
  All members of a datacenter should share a local LAN connection.

- `data_dir` `(string: required)` - Specifies a local directory used to store
  agent state. Client nodes use this directory by default to store temporary
  allocation data as well as cluster information. Server nodes use this
  directory to store cluster state, including the replicated log and snapshot
  data. This must be specified as an absolute path.

    ~> **WARNING**: This directory **must not** be set to a directory that is
    [included in the chroot](/docs/drivers/exec.html#chroot) if you use the
    [`exec`](/docs/drivers/exec.html) driver.

- `disable_anonymous_signature` `(bool: false)` - Specifies if Nomad should
  provide an anonymous signature for de-duplication with the update check.

- `disable_update_check` `(bool: false)` - Specifies if Nomad should not check
  for updates and security bulletins.

- `enable_debug` `(bool: false)` - Specifies if the debugging HTTP endpoints
  should be enabled. These endpoints can be used with profiling tools to dump
  diagnostic information about Nomad's internals.

- `enable_syslog` `(bool: false)` - Specifies if the agent should log to syslog.
  This option only works on Unix based systems.

- `http_api_response_headers` `(map<string|string>: nil)` - Specifies
  user-defined headers to add to the HTTP API responses.

- `leave_on_interrupt` `(bool: false)` - Specifies if the agent should
  gracefully leave when receiving the interrupt signal. By default, the agent
  will exit forcefully on any signal. This value should only be set to true on
  server agents if it is expected that a terminated server instance will never
  join the cluster again.

- `leave_on_terminate` `(bool: false)` - Specifies if the agent should
  gracefully leave when receiving the terminate signal. By default, the agent
  will exit forcefully on any signal. This value should only be set to true on
  server agents if it is expected that a terminated server instance will never
  join the cluster again.

- `limits` - Available in Nomad 0.10.3 and later, this is a nested object that
  configures limits that are enforced by the agent. The following parameters
  are available:

  - `https_handshake_timeout` `(string: "5s")` - Configures the limit for how
    long the HTTPS server in both client and server agents will wait for a
    client to complete a TLS handshake. This should be kept conservative as it
    limits how many connections an unauthenticated attacker can open if
    [`tls.http = true`][tls] is being used (strongly recommended in
    production).  Default value is `5s`. `0` disables HTTP handshake timeouts.

  - `http_max_conns_per_client` `(int: 100)` - Configures a limit of how many
    concurrent TCP connections a single client IP address is allowed to open to
    the agent's HTTP server. This affects the HTTP servers in both client and
    server agents. Default value is `100`. `0` disables HTTP connection limits.

  - `rpc_handshake_timeout` `(string: "5s")` - Configures the limit for how
    long servers will wait after a client TCP connection is established before
    they complete the connection handshake. When TLS is used, the same timeout
    applies to the TLS handshake separately from the initial protocol
    negotiation. All Nomad clients should perform this immediately on
    establishing a new connection. This should be kept conservative as it
    limits how many connections an unauthenticated attacker can open if
    TLS is being using to authenticate clients (strongly recommended in
    production). When `tls.rpc` is true on servers, this limits how long the
    connection and associated goroutines will be held open before the client
    successfully authenticates. Default value is `5s`. `0` disables RPC handshake
    timeouts.

  - `rpc_max_conns_per_client` `(int: 100)` - Configures a limit of how
    many concurrent TCP connections a single source IP address is allowed
    to open to a single server. Client agents do not accept RPC TCP connections
    directly and therefore are not affected. It affects both clients connections
    and other server connections. Nomad clients multiplex many RPC calls over a
    single TCP connection, except for streaming endpoints such as [log
    streaming][log-api] which require their own connection when routed through
    servers. A server needs at least 2 TCP connections (1 Raft, 1 RPC) per peer
    server locally and in any federated region. Servers also need a TCP connection
    per routed streaming endpoint concurrently in use. Only operators use streaming
    endpoints; as of 0.10.3 Nomad client code does not. A reasonably low limit
    significantly reduces the ability of an unauthenticated attacker to consume
    unbounded resources by holding open many connections. You may need to
    increase this if WAN federated servers connect via proxies or NAT gateways
    or similar causing many legitimate connections from a single source IP.
    Default value is `100` which is designed to support the majority of users.
    `0` disables RPC connection limits. `26` is the minimum as `20` connections
    are always reserved for non-streaming connections (Raft and RPC) to ensure
    streaming RPCs do not prevent normal server operation. This minimum may be
    lowered in the future when streaming RPCs no longer require their own TCP
    connection.

- `log_level` `(string: "INFO")` - Specifies  the verbosity of logs the Nomad
  agent will output. Valid log levels include `WARN`, `INFO`, or `DEBUG` in
  increasing order of verbosity.

- `log_json` `(bool: false)` - Output logs in a JSON format.

- `log_file` `(string: "")` - Specifies the path for logging. If the path
  does not includes a filename, the filename defaults to "nomad-{timestamp}.log".
  This setting can be combined with `log_rotate_bytes` and `log_rotate_duration`
  for a fine-grained log rotation control.

- `log_rotate_bytes` `(int: 0)` - Specifies the number of bytes that should be
  written to a log before it needs to be rotated. Unless specified, there is no
  limit to the number of bytes that can be written to a log file.

- `log_rotate_duration` `(duration: "24h")` - Specifies the maximum duration a
  log should be written to before it needs to be rotated. Must be a duration
  value such as 30s.

- `log_rotate_max_files` `(int: 0)` - Specifies the maximum number of older log
  file archives to keep. If 0 no files are ever deleted.

- `name` `(string: [hostname])` - Specifies the name of the local node. This
  value is used to identify individual agents. When specified on a server, the
  name must be unique within the region.

- `plugin_dir` `(string: "[data_dir]/plugins")` - Specifies the directory to
  use for looking up plugins. By default, this is the top-level
  [data_dir](#data_dir) suffixed with "plugins", like `"/opt/nomad/plugins"`.
  This must be an absolute path.

- `plugin` `(`[`Plugin`]`: nil)` - Specifies configuration for a
  specific plugin. The plugin stanza may be repeated, once for each plugin being
  configured. The key of the stanza is the plugin's executable name relative to
  the [plugin_dir](#plugin_dir).

- `ports` `(Port: see below)` - Specifies the network ports used for different
  services required by the Nomad agent.

  - `http` - The port used to run the HTTP server.

  - `rpc` - The port used for internal RPC communication between
    agents and servers, and for inter-server traffic for the consensus algorithm
    (raft).

  - `serf` - The port used for the gossip protocol for cluster
    membership. Both TCP and UDP should be routable between the server nodes on
    this port.

    The default values are:

    ```hcl
    ports {
      http = 4646
      rpc  = 4647
      serf = 4648
    }
    ```

- `region` `(string: "global")` - Specifies the region the Nomad agent is a
  member of. A region typically maps to a geographic region, for example `us`,
  with potentially multiple zones, which map to [datacenters](#datacenter) such
  as `us-west` and `us-east`.

- `sentinel` `(`[`Sentinel`]`: nil)` - Specifies configuration for Sentinel
  policies.

- `server` `(`[`Server`]`: nil)` - Specifies configuration which is specific
  to the Nomad server.

- `syslog_facility` `(string: "LOCAL0")` - Specifies the syslog facility to
  write to. This has no effect unless `enable_syslog` is true.

- `tls` `(`[`TLS`][tls]`: nil)` - Specifies configuration for TLS.

- `vault` `(`[`Vault`]`: nil)` - Specifies configuration for
  connecting to Vault.

## Examples

### Custom Region and Datacenter

This example shows configuring a custom region and data center for the Nomad
agent:

```hcl
region     = "europe"
datacenter = "ams"
```

### Enable CORS

This example shows how to enable CORS on the HTTP API endpoints:

```hcl
http_api_response_headers {
  "Access-Control-Allow-Origin" = "*"
}
```

[`ACL`]: /docs/configuration/acl.html "Nomad Agent ACL Configuration"
[`Client`]: /docs/configuration/client.html "Nomad Agent client Configuration"
[`Consul`]: /docs/configuration/consul.html "Nomad Agent consul Configuration"
[`Plugin`]: /docs/configuration/plugin.html "Nomad Agent Plugin Configuration"
[`Sentinel`]: /docs/configuration/sentinel.html "Nomad Agent sentinel Configuration"
[`Server`]: /docs/configuration/server.html "Nomad Agent server Configuration"
[tls]: /docs/configuration/tls.html "Nomad Agent tls Configuration"
[`Vault`]: /docs/configuration/vault.html "Nomad Agent vault Configuration"
[go-sockaddr/template]: https://godoc.org/github.com/hashicorp/go-sockaddr/template
[log-api]: /api/client.html#stream-logs
[hcl]: https://github.com/hashicorp/hcl "HashiCorp Configuration Language"
Overview
========

This project is about streaming Solana account updates for a specific program
into other databases or event queues.

Having an up to date version of all account data data in a database is
particularly useful for queries that need access to all accounts. For example,
retrieving the addresses of Mango Markets accounts with the largest unrealized
PnL goes from "getProgramAccounts from a Solana node for 50MB of data and compute
locally (3-10s total)" to "run a SQL query (150ms total)".

The database could also be used as a backend for serving `getMultipleAccounts`
and `getProgramAccounts` queries generally. That would reduce load on Solana RPCt
nodes while decreasing response times.

Supported Solana sources:
- Geyser plugin (preferred) plus JSONRPC HTTP API (for initial snapshots)

Unfinished Solana sources:
- JSONRPC websocket subscriptions plus JSONRPC HTTP API (for initial snapshots)



Components
==========

- [`geyser-plugin-grpc/`](geyser-plugin-grpc/)

  The Solana Geyser plugin. It opens a gRPC server (see [`proto/`](proto/)) and
  broadcasts account and slot updates to all clients that connect.

- [`lib/`](lib/)

  The connector abstractions that the connector service is built from.

  Projects may want to use it to build their own connector service and decode
  their specific account data before sending it into target systems.

- [`connector-raw/`](connector-raw/)

  A connector binary built on lib/ that stores raw binary account data in
  PostgreSQL.

- [`connector-mango/`](connector-mango/)

  A connector binary built on lib/ that decodes Mango account types before
  storing them in PostgeSQL.


Setup Tutorial
==============

1. Compile the project.

   Make sure that you are using _exactly_ the same Rust version for compiling the
   Geyser plugin that was used for compiling your `solana-validator`! Otherwise
   the plugin will crash the validator during startup!

2. Prepare the plugin configuration file.

   [Here is an example](geyser-plugin-grpc/example-config.json). This file
   points the validator to your plugin shared library, controls which accounts
   will be exported, which address the gRPC server will bind to and internal
   queue sizes.

3. Run `solana-validator` with `--geyser-plugin-config myconfig.json`.

   Check the logs to ensure the plugin was loaded.

4. Monitor the logs

   `WARN` messages can be recovered from. `ERROR` messages need attention.

   Check the metrics for `account_write_queue` and `slot_update_queue`: They should
   be around 0. If they keep growing the service can't keep up and you'll need
   to figure out what's up.


Design and Reliability
======================

```
Solana    --------------->   Connector 
 nodes      jsonrpc/gRPC       nodes
```

For reliability it is recommended to feed data from multiple Solana nodes into
each Connector node.

The Connector service is stateless (except for some caches). Restarting it is
always safe.
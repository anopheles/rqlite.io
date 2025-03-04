---
title: "Security"
linkTitle: "Security"
"description": "How to secure your rqlite deployment"
weight: 20
---
rqlite can be secured in various way, and with different levels of control.

## SQLite security
SQLite has some [documentation on security](https://www.sqlite.org/security.html), which is worth reviewing. Much of it can be applied to rqlite, though implementing some of the practices would need you to [recompile rqlite](/docs/install-rqlite/building-from-source/).

## File system security
You should control access to the data directory that each rqlite node uses. There is no reason for any user to directly access this directory. File-level security is also very important if you decide to use _TLS Certificates and Keys_ with rqlite (see later).

You are also responsible for securing access to the SQLite database files if you enable "on disk" mode (which is not the default mode). Unless your application requires it, there is no reason for any client to directly access the SQLite file, and writing to the SQLite file may cause rqlite to fail. If your users don't need to access a SQLite database file, then don't enable "on disk" mode. This will maximize file-level security.

## Network security
Each rqlite node listens on 2 TCP ports -- one for the HTTP API, and the other for inter-node (Raft consensus) communications. Only the HTTP API port need be reachable from outside the cluster.

So, if possible, configure the network such that the Raft port on each node is only accessible from other nodes in the cluster. Alternatively run the Raft connections on a physically, or logically, different network from the network the HTTP API is connected to. There is no need for the Raft port to be accessible by rqlite clients, which only need to use the HTTP API.

If the IP addresses (or subnets) of rqlite clients is also known, it may also be possible to limit access to the HTTP API from those addresses only.

AWS EC2 [Security Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html), for example, support all this functionality. So if running rqlite in the AWS EC2 cloud you can implement this level of security at the network level.

## HTTPS API
rqlite supports secure access via HTTPS, using [_Transport Layer Security (TLS)_](https://www.cloudflare.com/learning/ssl/what-is-an-ssl-certificate/). Using TLS ensuring that all communication between clients and a cluster is encrypted. The HTTPS API also supports [Mutual TLS](https://www.cloudflare.com/learning/access-management/what-is-mutual-tls/). If mutual TLS is enabled only clients which present a trusted certificate can access rqlite.

To configure HTTPS, you set the following command-line options when launching rqlite:
```
  -http-ca-cert string
      Path to X.509 CA certificate for HTTPS.
      If not set, then the host systems CA certificate(s) will be used.
      This certificate must be set however if using mutual TLS.
  -http-cert string
      Path to HTTPS X.509 certificate
  -http-key string
      Path to corresponding HTTPS X.509 private key
  -http-no-verify
      Skip verification of HTTPS certificates when joining a cluster.
      Mostly used for testing.
  -http-verify-client
      Enable mutual TLS for HTTPS. Mutual TLS is disabled by default.
```

## Node-to-node encryption
rqlite supports authentication encryption of all inter-node traffic<sup>1</sup>. TLS is again used, and mutual TLS is also supported so you can restrict nodes to only accept inter-node connections from other authorized nodes. To use TLS each node must be supplied with the relevant SSL certificate and corresponding private key, in X.509 format. Note that every node in a cluster must operate with inter-node encryption enabled, or none at all.
```bash
  -node-ca-cert string
      Path to X.509 CA certificate for node-to-node encryption.
      If not set, then the host systems CA certificate(s) will be used
      when verifying server certificates. This certificate is required
      for verifying client certificates.
  -node-cert string
      Path to X.509 certificate for node-to-node communication
  -node-key string
      Path to X.509 private key for node-to-node communicate
  -node-no-verify
      Skip verification of any node-node certificates in each direction.
      Mostly used for testing.
  -node-verify-client
      Enable mutual TLS for node-to-node communication.
      Mutual TLS is disabled by default.
```
<sup>1</sup>There is one time nodes use the HTTP API when communicating -- during a _Join_ operation, or during bootstrapping (autoclustering). But once a node joins a cluster, all further communications takes place over the node-to-node connection.

## Basic Auth
The HTTP API supports [Basic Auth](https://tools.ietf.org/html/rfc2617). Each rqlite node can be passed a JSON-formatted configuration file, which configures valid usernames and associated passwords for that node. The password string can be in cleartext or [bcrypt hashed](https://en.wikipedia.org/wiki/Bcrypt).

Since the configuration file only controls the node local to it, it's important to ensure the configuration is identical on each node for consistent behaviour.

### User-level permissions
rqlite, via the configuration file, also supports user-level permissions. Each user can be granted one or more of the following permissions:
- _all_: user can perform all operations on a node.
- _execute_: user may access the execute endpoint.
- _query_: user may access the query endpoint.
- _load_: user may load an SQLite dump file into a node.
- _backup_: user may perform backups.
- _status_: user can retrieve node status and Go runtime information.
- _ready_: user can retrieve node readiness.
- _join_: user can join a cluster. In practice only a node joins a cluster, so it's the joining node that must supply the credentials.
- _join-read-only_: user can join a cluster, but only as a read-only node.
- _remove_: user can remove a node from a cluster.

Note that for a user to be able to access the [Unified Endpoint](/docs/api/api/#unified-endpoint), they must have **both** _execute_ and _query_ permissions.

### Example configuration file
An example configuration file is shown below.
```json
[
  {
    "username": "bob",
    "password": "secret1",
    "perms": ["all"]
  },
  {
    "username": "mary",
    "password": "$2a$10$fKRHxrEuyDTP6tXIiDycr.nyC8Q7UMIfc31YMyXHDLgRDyhLK3VFS",
    "perms": ["query", "backup", "join"]
  },
  {
    "username": "*",
    "perms": ["status", "ready", "join-read-only"]
  }
]
```
This configuration file sets authentication for three usernames, _bob_, _mary_, and `*`. It sets a password for the first two.

This configuration also sets permissions for all usernames. _bob_ has permission to perform all operations, but _mary_ can only query the cluster, as well as backup and join the cluster. `*` is a special username, which indicates that all users -- even anonymous users (requests without any BasicAuth information) -- have permission to check the cluster status and readiness. All users can also join as a read-only node. This can be useful if you wish to leave certain operations open to all accesses.

## Secure cluster example
Starting a node with HTTPS enabled, node-to-node encryption, mutual TLS disabled, and with the above configuration file. It is assumed the HTTPS X.509 certificate and key are at the paths `server.crt` and `key.pem` respectively, and the node-to-node certificate and key are at `node.crt` and `node-key.pem`
```bash
rqlited -auth config.json -http-cert server.crt -http-key key.pem \
-node-cert node.crt -node-key node-key.pem ~/node.1
```
Bringing up a second node on the same host, joining it to the first node. This allows you to block nodes from joining a cluster, unless those nodes supply a password.
```bash
rqlited -auth config.json -http-addr localhost:4003 -http-cert server.crt \
-http-key key.pem -raft-addr :4004 -join https://bob:secret1@localhost:4001 \
-node-cert node.crt -node-key node-key.pem ~/node.2
```
Querying the node, as user _mary_.
```bash
curl -G 'https://mary:secret2@localhost:4001/db/query?pretty&timings' \
--data-urlencode 'q=SELECT * FROM foo'
```

### Avoiding passwords at the command line
The above example suffer from one shortcoming -- the password for user `bob` is entered at the command line. This is not ideal, as someone with access to the process table could learn the password. You can avoid this via the `-join-as` option, which will tell rqlite to retrieve the password from the configuration file.
```bash
rqlited -auth config.json -http-addr localhost:4003 -http-cert server.crt \
-http-key key.pem -raft-addr :4004 -join https://localhost:4001 -join-as bob \
-node-cert node.crt -node-key node-key.pem \
~/node.2
```

## How can I generate certificates and keys?
There are a lot of resources available on the web explaining how to do so. The most popular tool for creating and signing certificates and keys is [OpenSSL](https://www.openssl.org/).

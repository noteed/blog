---
title: Exposing a local server through HAProxy using a reverse SSH tunnel
published: 2017-08-03
---


Probably the most frequent way to write a server is to
run it on a laptop and access it directly, using `localhost` or
`127.0.0.1`. It works and is enough for simple settings. Sometimes though, such
a setup is a bit _too_ simple. For instance you might want your server to check
the content of the Host header, or let coworkers and webhooks reach it.

A possible solution is to use a service like [ngrok](https://ngrok.com/).
Another solution when you're already using HAProxy somewhere in the cloud, is
to expand its configuration to include a new subdomain using a new backend
(your laptop), accessible through a reverse SSH tunnel.

Among the reasons to re-use an existing HAProxy deployment, I see sharing a
common namespace, making it easier for people to remember which hostname is
used for who or which service. E.g. imagine having:

- `alice.example.com` points to Alice's laptop
- `bob.example.com` points to Bob's workstation
- `alpha.example.com` points to a development machine in the cloud

Even when working alone, Alice can use the realistic `alice.example.com` domain,
and share URLs easily with Bob later.

Another reason to re-use HAProxy is to share parts of its configuration: for
instance require a client certificate to access some subdomains.

In practice, using HAproxy to expose a developer computer means a few things:

- Run your server on your machine. Here we assume it runs within a container
  with IP address `172.17.0.12` on the port `8008`.
- Assuming you own `example.com`, configure your DNS to point `alice.example.com`
  to the machine running HAProxy.
- Create the reverse SSH tunnel, from your machine to the HAProxy server,
  opening, say, the port `10008`. This means that HAProxy will be able to use
  `127.0.0.1:10008` in a new backend configuration.
- Configure HAProxy to route `alice.example.com` to the new backend.

Instead of having SSH create a listening port on the HAProxy server, it can
also, since version 6.7, create a UNIX domain socket. This is useful if HAProxy
cannot use the loopback address to access the tunnel (e.g. because it runs
within a container). Instead of having SSH listen on `0.0.0.0`, you can share
the socket path. (Listening on `0.0.0.0` would expose the port to the outside
world and defeat the use of client certificates when routing the subdomain to
your laptop.)


# HAProxy configuration and `ssh` command:

Using a UNIX domain socket is possible in HAProxy: simply give the path instead of an `addr:port` pair:

```
server alice /forwarder/alice.sock cookie alice
```

On the laptop, running the tunnel looks like:

```
> ssh -NT -R /forwarder/alice.sock:172.17.0.12:8008 -l alice example.com
```


# Troubleshooting

In the next sections, I show some possible problems you might encounter.


## Using cURL with UNIX domain sockets

On the HAProxy machine, you can check if your server is indeed accessible
through the forwaded UNIX domain socket with a command like:

```
> curl --unix-socket /forwarder/alice.sock http://alice
```

## Deleting an existing socket when a client tries to establish a reverse tunnel

You can read an error looking like the following:

```
Warning: remote port forwarding failed for listen path /forwarder/alice.sock
```

It might be because the file already exists. If the error disappears when you
delete it, you can configure you SSH server to reuse existing paths: in
`sshd_config`, set `StreamLocalBindUnlink yes`.


## Allow the forwarded port to listen to a public address.

If you use a regular port instead of a UNIX domain socket, you should know that
by default the SSH server will make forwarded ports listen only on the loopback
address. If HAProxy should connect to another address for some reason (e.g. it
is running in a container and the SSH server is in another), in `sshd_config`,
set `GatewayPort yes`.

Be carefull if your `address:port` combination is accessible from the outside
world.


## HAProxy access right

Make sure the HAProxy process can read the socket (i.e. can read the file and
traverse the necessary directories).

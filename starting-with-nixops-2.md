---
title: Starting with NixOps (and thus Nix and NixOS), part 2
published: 2017-08-11
---

# Starting with NixOps (and thus Nix and NixOS), part 2

In part 1, I showed how to write a very basic Nix expression to describe a
machine to be deployed on Digital Ocean using NixOps, and the few commands
necessary to deploy and destroy it.

I also showed that it was possible to SSH into it. Acutally this is not
something you really want to do. Instead we will update the Nix expression to
refine the setup of our droplet and let NixOps do the rest. (SSHing into a
machine should only be used to investigate problems, not to change the state of
a machine.)


## .envrc

Having to specify the `DIGITAL_OCEAN_AUTH_TOKEN` environment variable is
tiresome (and possibly insecure since it is visible is various places, e.g. the
shell history), so I'm using a `.envrc` file:

```
$ echo 'export DIGITAL_OCEAN_AUTH_TOKEN=xxxx' > .envrc
$ direnv allow
```

In the reminder of the blog post, I will thus no longer prefix my commands with
the Digital Ocean token.


## Nginx

As a first modification to our deployment, let's add Nginx. Since Nginx is
already part of NixOS, we don't have much to do:

- add a firewall rule to expose the port 80,
- enable Nginx.

This translates to the two following lines (to be added in the `machine-1`
description):

```
    networking.firewall.allowedTCPPorts = [ 80 ];
    services.nginx.enable = true;
```

After deployment, you should be able to get an answer from your server:

```
$ nixops deploy -d do-rem
$ curl http://185.14.186.74
```

(The answer is a 404 since we need to further configure Nginx.)

As a reminder, in addition to retrieving the IP address from the Digital Ocean
web interface, you can also use `nixops info`.

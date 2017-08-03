---
title: Starting with NixOps (and thus Nix and NixOS)
published: 2017-08-10
---

# Starting with NixOps (and thus Nix and NixOS)

While learning the Nix ecosystem and trying to use it, I found it a bit more
harder than I thought to achieve what I wanted. In this post, I'm documenting
what I learned, partly for myself, partly to share with other people that would
like to follow the same path.

The prerequesite to follow along this post is to install Nix and NixOps.


## Basics

The basics of Nix are actually very well documented elsewhere; in particular:

- [The Nix manual](http://nixos.org/nix/manual/)
- [Nix by example](https://medium.com/@MrJamesFisher/nix-by-example-a0063a1a4c55)
- [The Nix pills series](http://lethalman.blogspot.be/2014/07/nix-pill-1-why-you-should-give-it-try.html)

As as a starting point, here I give a `do.nix` file suitable for the `nixops`
executable. I will use it in the rest of the post as a running example:

```
{
  network.description = "Some machines (actually just one)";

  machine-1 = { config, pkgs, ... }: {
    deployment.targetEnv = "digitalOcean";
    deployment.digitalOcean.region = "ams2";
    deployment.digitalOcean.size = "512mb";

    networking.firewall.allowedTCPPorts = [ 80 ];

    services.nginx = {
      enable = true;

    }; # service.nginx

  }; # machine-1
}
```

That file is enough to get a machine up and running on Digital Ocean. (It seems
the support for the AWS environment is much more mature but for what this post
is about, Digital Ocean support is fine.)

Here how you instruct NixOps to use that file to spin up a machine a provision
it.

When you want to update the machine, following changes to its configuration
file, the same `deploy` command is used again.


## XXX

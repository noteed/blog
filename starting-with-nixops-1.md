---
title: Starting with NixOps (and thus Nix and NixOS), part 1
published: 2017-08-04
---

# Starting with NixOps (and thus Nix and NixOS), part 1

While learning the Nix ecosystem and trying to use it, I found it a bit more
harder than I thought to achieve what I wanted. In this post and the next one,
I'm documenting what I learned, partly for myself, partly to share with other
people that would like to follow the same path.

The prerequesite to follow along this post is to install Nix and NixOps.


## Basics

The basics of Nix are actually very well documented elsewhere; in particular:

- [The Nix manual](http://nixos.org/nix/manual/)
- [Nix by example](https://medium.com/@MrJamesFisher/nix-by-example-a0063a1a4c55)
- [The Nix pills series](http://lethalman.blogspot.be/2014/07/nix-pill-1-why-you-should-give-it-try.html)


## Running example

As as a starting point, here I give a `do.nix` file suitable for the `nixops`
executable. I will use it in the rest of the post as a running example:

```
$ cat do.nix
{
  network.description = "Some machines (actually just one)";

  resources.sshKeyPairs.ssh-key = {};

  machine-1 = { config, pkgs, ... }: {
    deployment.targetEnv = "digitalOcean";
    deployment.digitalOcean.region = "ams2";
    deployment.digitalOcean.size = "512mb";

  }; # machine-1
}
```

That file contains a single Nix expression and is enough to get a machine up
and running on Digital Ocean. (It seems the support for the AWS environment is
much more mature but for what this post is about, Digital Ocean support is
fine.)

Here is how you instruct NixOps to use that file to spin up a machine and
provision it (you can create API tokens at
https://cloud.digitalocean.com/settings/api/tokens):

```
$ nixops create -d do do.nix
$ nixops list
+----------------+------+------------------------+------------+--------------+
| UUID           | Name | Description            | # Machines |     Type     |
+----------------+------+------------------------+------------+--------------+
| c94aa2ee-7954- | do   | Unnamed NixOps network |          0 |              |
+----------------+------+------------------------+------------+--------------+
$ nixops info -d do
Network name: do
Network UUID: c94aa2ee-7954-11e7-8947-0242c5c1eaab
Network description: Some machines (actually just one)
Nix expressions: /home/thu/projects/web/nixops/do.nix

+-----------+---------------+---------------------+-------------+------------+
| Name      |     Status    | Type                | Resource Id | IP address |
+-----------+---------------+---------------------+-------------+------------+
| machine-1 | Missing / New | digitalOcean [ams2] |             |            |
+-----------+---------------+---------------------+-------------+------------+
$ DIGITAL_OCEAN_AUTH_TOKEN=xxxx nixops deploy -d do
machine-1> creating droplet ...
...
```

This creates a "deployment" using our expression: a mean for NixOps to track
the state associated with our machines and name that deployment "do" (which can
be used instead of its UUID) then actually deploy it to Digital Ocean. (For
good measure I call also `nixops list` and `nixops info` to demonstrate those
two commands.)

After a while (a long while when using the Digital Ocean target), you should be
able to SSH into the machine:

```
...
machine-1> activation finished successfully
do> deployment finished successfully
$ nixops ssh -d do machine-1
[root@machine-1:~]#
```

For good measure again:

```
$ nixops list
+----------------+------+-----------------------------------+------------+--------------+
| UUID           | Name | Description                       | # Machines |     Type     |
+----------------+------+-----------------------------------+------------+--------------+
| c94aa2ee-7954- | do   | Some machines (actually just one) |          1 | digitalOcean |
+----------------+------+-----------------------------------+------------+--------------+
$ nixops info -d do
Network name: do
Network UUID: c94aa2ee-7954-11e7-8947-0242c5c1eaab
Network description: Some machines (actually just one)
Nix expressions: /home/thu/projects/web/nixops/do.nix

+-----------+-----------------+--------------+-------------+----------------+
| Name      |      Status     | Type         | Resource Id | IP address     |
+-----------+-----------------+--------------+-------------+----------------+
| machine-1 | Up / Up-to-date | digitalOcean |             | 188.226.174.95 |
| ssh-key   | Up / Up-to-date | ssh-keypair  |             |                |
+-----------+-----------------+--------------+-------------+----------------+
```


## Deploying changes

When you want to update the machine, following changes to the Nix expression
describing it, the same `deploy` command is used again. (You can already do it
again with no changes and it should complete much more quickly.)


## Destroying the machine

You can destroy the droplet with the following command:

```
$ DIGITAL_OCEAN_AUTH_TOKEN=xxxx nixops destroy -d do
machine-1> destroying droplet 57418645
```


## Next

[Part 2 is here.](starting-with-nixops-2.md)

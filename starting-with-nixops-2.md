---
title: Starting with NixOps (and thus Nix and NixOS), part 2
published: 2017-08-15
---

# Starting with NixOps (and thus Nix and NixOS), part 2

In [part 1](starting-with-nixops-1.md), I showed how to write a very basic Nix
expression to describe a machine to be deployed on Digital Ocean using NixOps,
and the few commands necessary to deploy and destroy it.

I also showed that it was possible to SSH into it. Acutally this is not
something you really want to do. Instead we will update the Nix expression to
refine the setup of our droplet and let NixOps do the rest. (SSHing into a
machine should only be used to investigate problems, not to change the state of
a machine.)

In this post, I'll show how to add a handful of things to our deployment.


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


## Static site

In this section we prepare a simple one-file static site. We turn it into a Nix
expression that we use in our deployment.

We create a directory (this could be a complete Git repository) to contain the
site andd associated scripts (e.g. a static site generator that would generate
a `_site` directory):

```
$ mkdir -p static_site/_site
$ echo "Hello." > static_site/_site/index.html
```

We add a `default.nix` file to build the site:

```
with import <nixpkgs> {};
{
  static_site = stdenv.mkDerivation {
    name = "static_site";
    src  = ./.;
    installPhase = ''
      mkdir -p "$out/"
      cp -a ./_site "$out/"
    '';
  };
}
```

The `default.nix` file uses the standard Nix machinery, nothing specific to
NixOS. All it does is copying the `_site` directory. In a more realistic setup,
it would probably first build it.

Now to use that package in our deployment, we:

- import it
- use it in the Nginx configuration

At the top of our `do.nix` file we add
```
let
  static_site = (import ./static_site).static_site;
in
```

We modify the Nginx part with:

```
    services.nginx = {
      enable = true;
      virtualHosts = {
        "example.com" = {
          root = "${static_site}/_site";
        };
      };
    };
```

After deploying again:

```
$ curl http://185.14.186.74
Hello.
```


## Users

Adding users look like this:

```
    users.mutableUsers = false;
    users.extraUsers.toto = {
      uid = 1000;
      isNormalUser = true;
      home = "/home/toto";
      description = "The Toto User";
      extraGroups = [ "wheel" ];
      openssh.authorizedKeys.keys = [ "ssh-rsa xxxx toto@somewhere" ];
    };
```

Stating `mutableUsers = false` basically means that existing users and
passwords are configured by the deployment instead of by login into the machine
then changing things.

Now that we hase a user, we can provision some data directory:

```
    system.activationScripts.toto =
      ''
        echo Creating toto directories...
        mkdir -m 0755 -p /home/toto/toto
        chown toto:users /home/toto/toto
      '';
```

We can confirm all is well:

```
$ nixops ssh -d do-rem machine-1
[root@machine-1:~]# su toto
[toto@machine-1:/root]$ cd
[toto@machine-1:~]$ ls
toto
```


## Cron


We the above user and directory in place, we add a cron job to fill that
directory:

```
    services.cron = {
      enable = true;
      systemCronJobs = [
        "0 5 * * * toto cd /home/toto/toto && date > date.log"
      ];
    };
```


## Packages

If you need a pakcage that is not automatically installed (i.e. it is not yet a
dependency of your deployment), you can specify it by itself. Here we add
`wget`:

```
    environment.systemPackages = [
      pkgs.wget
    ];
```


## Imports

Instead of having everything in the same Nix expression (beside the static
site), it is possible to use the `imports` feature of NixOS.

We remove the `extraUsers`,`activationScripts` and `systemCronJobs` parts and
move them to a new file, `toto.nix`:

```
{ config, pkgs, ... }: {
  users.extraUsers.toto = {
    uid = 1000;
    isNormalUser = true;
    home = "/home/toto";
    description = "The Toto User";
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [ "ssh-rsa xxxx toto@somewhere" ];
  };

  system.activationScripts.toto =
    ''
      echo Creating toto directories...
      mkdir -m 0755 -p /home/toto/toto
      chown toto:users /home/toto/toto
    '';

    services.cron.systemCronJobs = [
        "35 * * * * toto cd /home/toto/toto && echo Hello > date.log"
    ];
}
```

In their place, we simply import the new file:

```
    imports = [ ./toto.nix ];
```

NixOS will merge similar records to create the deployment.

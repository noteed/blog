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

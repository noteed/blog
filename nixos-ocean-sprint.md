# NixOS Ocean Sprint (December 2021)

I've been lucky to participate to the [NixOS Ocean
Sprint](https://oceansprint.org/) this December on Lanzarote, and also to
sponsor it (t-shirts, yeah!). It both made a nice change from the Belgian
weather, and allowed me to actually meet great people from the NixOS
community.

In particular, it was nice to talk with people that knew weird things such as
not-os or NAR files.

Productivity-wise, I could worked a bit the first day with Marjan to turn the
NixOS testing code (written as a single Python file) into a proper [Python
package](https://github.com/NixOS/nixpkgs/pull/149329), both from the point of
vue of Python itself, and Nix (using `buildPythonApplication`). The second day,
I helped Jonas to create a smaller version of `nixos/modules/module-list.nix`.
It lists about 1200 modules and I've been able to reduce it at about [80
modules] that still result in a bootable QEMU image.

[80 modules]: https://github.com/noteed/nixpkgs/blob/minimal-modules-list/nixos/modules/module-list-edited.nix

After that, I tried a bit to actually reduce the toplevel closure size of that
image (reducing the module list doesn't change the image, only the amount of
work Nix has to do to evaluate its configuration). By default, that image is
about 1GB.

To do that I mostly used `nix-store -qR result`, which lists the Nix store path
used in the closure, and `nix why-depends` to try to find packages that could
be removed from the closure. Actually removing them can be done by disabling
some options, or trying to expose new options. Although I'm using QEMU to try
if the image boots, I tried disabling its Guest Agent, which results in a nice
size decrease (indeed, by default the Guest Agent has support for things like X
or PulseAudio). I'm at 550MB right now.

---
layout: post
title:  "The DOs and DON'Ts of nixpkgs overlays"
date:   2017-11-03 16:17:00 +0100
---
One presentation at [NixCon 2017][nixcon2017] that especially drew my attention
was [Nicolas Pierron][nbpname]'s talk about **Nixpkgs overlays**
([video][video], [slides][slides]). I'd like to give a quick summary here for
future reference. All the credits go to Nicolas, of course.

## What are overlays?

Overlays are the new standard mechanism to customize nixpkgs. They replace
constructs like `packageOverride` and `overridePackages`.

## How do I use overlays?

Put your overlays into *~/.config/nixpkgs/overlays/*. All overlays share a basic
structure:

{% highlight nix %}
self: super: {
  …
}
{% endhighlight %}

Arguments:

* **self** is the result of the fix point calculation. Use it to access packages
  which could be modified somewhere else in the overlay stack.
* **super** one overlay down in the stack (and base nixpkgs for the first
  overlay). Use it to access the package recipes you want to customize, and
  for library functions.

## Good examples

### Add default proxy to Google Chrome

{% highlight nix %}
self: super:
{
  google-chrome = super.google-chrome.override {
    commandLineArgs =
      ''--proxy-server="https=127.0.0.1:3128;http=127.0.0.1:3128"'';
  };
}
{% endhighlight %}

From [Alexandre Peyroux](https://spof.px.io/~alex/posts/2017-05-15-nixos-overlay.html).

### Fudge Nix store location

{% highlight nix %}
self: super:
{
  nix = super.nix.override {
    storeDir = "${<nix-dir>}/store";
    stateDir = "${<nix-dir>}/var";
  };
}
{% endhighlight %}

From [Yann Hodique](http://yann.hodique.info/blog/nix-in-custom-location/).

### Add custom package

{% highlight nix %}
self: super:
{
  VidyoDesktop = super.callPackage ./pkgs/VidyoDesktop { };
}
{% endhighlight %}

From [Mozilla](https://github.com/mozilla/nixpkgs-mozilla/).

## Bad examples

### Recursive attrset

{% highlight nix %}
self: super: rec {
  fakeClosure = self.writeText "fake-closure.nix" ''
    …
  '';
  fakeConfig = self.writeText "fake-config.nix" ''
    (import ${fakeClosure} {}).config.nixpkgs.config
  '';
}
{% endhighlight %}

Two issues:

* Use `super` to access library functions.
* Overlays should not be recursive. Use `self`.

### Surplus arguments

{% highlight nix %}
{ python }:
self: super:
{
  "execnet" =
    python.overrideDerivation super."execnet" (old: {
      buildInputs = old.buildInputs ++ [ self."setuptools-scm" ];
    });
}
{% endhighlight %}

The issue:

* Overlays should depend just on `self` and `super` in order to be composable.

### Awkward nixpkgs import

{% highlight nix %}
{ pkgs ? import <nixpkgs> {} }:
let
  projectOverlay = self: super: {
    customPythonPackages =
      (import ./requirements.nix {
        inherit pkgs;
      }).packages;
  };
in
  import pkgs.path {
    overlays = [ projectOverlay ];
  }
{% endhighlight %}

Two issues:

* Unnecessary double-import of nixpkgs. This might break cross-system builds.
* *requirements.nix* should use `self` not `pkgs`.

#### Improved version

*shell.nix*:
{% highlight nix %}
{ pkgsPath ? <nixpkgs> }:
import pkgsPath {
  overlays = [ import ./default.nix; ];
}
{% endhighlight %}

*default.nix*:
{% highlight nix %}
self: super:
{
  customPythonPackages =
    (import ./requirements.nix { inherit self; }).packages;
}
{% endhighlight %}

### Incorrectly referenced dependencies

{% highlight nix %}
self: super:
let inherit (super) callPackage;
in {
  radare2 = callPackage ./pkgs/radare2 {
    inherit (super.gnome2) vte;
    lua = super.lua5;
  };
}
{% endhighlight %}

The issue:

* Other packages should be taken from `self` not `super`. This way they
  can be overridden by other overlays.

### Overridden attrset

{% highlight nix %}
self: super:
{
  lib = {
    firefoxVersion = … ;
  };
  latest = {
    firefox-nightly-bin = … ;
    firefox-beta-bin = … ;
    firefox-bin = … ;
    firefox-esr-bin = … ;
  };
}
{% endhighlight %}

The issue:

* Other attributes present in `lib` and `latest` from down the overlay stack are
  erased.

#### Improved version

Always extend attrsets in overlays:
{% highlight nix %}
self: super:
{
  lib = (super.lib or {}) // {
    firefoxVersion = … ;
  };
  latest = (super.latest or {}) // {
    …
  };
}
{% endhighlight %}

## Summary

I hope this condensed guide helps you to write better overlays. For
in-depth discussion, please go watch Nicolas' [talk][video] and read the
[Nixpkgs manual][manual-overlays].  Many thanks to Nicolas for putting this
together!


[nixcon2017]: http://nixcon2017.org/
[nbpname]: https://twitter.com/nbpname
[slides]: http://nbp.github.io/slides/NixCon/2017.NixpkgsOverlays/
[video]: https://youtu.be/6bLF7zqB7EM?t=39m50s
[manual-overlays]: https://nixos.org/nixpkgs/manual/#chap-overlays

<!--
cover image:
https://www.flickr.com/photos/thedailyenglishshow/3909355644 (preferred)
https://www.flickr.com/photos/indi/4259120807/
-->

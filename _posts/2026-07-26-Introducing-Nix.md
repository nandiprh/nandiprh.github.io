---
layout: post
title: "Comprehensive guide to Nix on non-nixos systems"
date: 2026-07-26
categories: networking security
---

So lately I've been getting more interested in android, its layout and its handling of packages. 
I remember 2 years back when I stepped into the worlld of programming, I was enthusiastic about OS, and eventually I discovered NixOS. The fact being I really liked the idea of the OS being declarative, reproducible and upgrades being atomic.
But I ran into serious learning curve quite early, with problems being most of the projects I tried running had their own compiling instructions, and those expected shared libraries via UNIX FHS(File hierarchy system).
Well the Nix community had the answers in the form of a dedicated flake file that described the entire build environment along with the nixpkgs required.
But well it works great if had I been fluent in Nix and the template it followed. 


Eventually I ditched NixOS and distrohopped a lot, in this process I grasped some of the Linux specific concepts, along with multitudes of applications that bundled with the Linux kernel to make it usable as a distribution.
So I adopted a hybrid approach, using Nix in Gentoo !
The fact being all the system level packages being installed by portage via Gentoo repo and user level applications via Nixpkgs. This let me access nix shell, nix build and nix develop, but also the tradeoff being loosing the Nixos generation rollbacks and a good level of declarativity that was only possible in NixOS.
But the plus point being gaining access to the FHS again.
For the very fact I switched from Openrc to systemd to gain more of nix automation of declaring daemons, as nix being tightly coupled with systemd as an init system.


For a multi user installation, it is important for every user to have access to a main nix conf which is situated in /etc/nix/nix.conf , further user level changes can be invoked in ~/.config/nix for a single user.
Also nix never gives a user permit to build nixpkgs or a better term would be nix derivations. It is handled by nix builders, the number of nix builders can be defined in nix.conf to tightly couple it with available cpu cores, or else the default builders remain 8 in number.
Unlike Gentoo's portage whose historial behaviour has been compiling from source, nix uses cachix to rather install nixpkgs via cache.nixos.org .

Well a very important distinction is to separate the fact that rather than treating nix as a package manager it is foremost a buil system in my opinion, and thus nix doesn't really have a concept of package distribution via repositories. Rather the conventional method is to use nix channels and define one of either the current release of NixOS or the unstable one.
For users who choose to proceed via flakes (has been experimental since a decade, but around 70%(rough estimate, source: trust me bro!) of users utilise this feature), which introduces a great concept of lock file along with flake registry to decide the input and output.
The lock file makes reproducibility pinned via version locking with the help of flake.lock (auto generated via flake.nix, and must be treated as a readonly file.), well this was inspired by Rust's cargo , and the way it pinned versions of crates in a toml file.


The nix.conf file I use in my system.

```nix
build-users-group = nixbld
experimental-features = nix-command flakes

max-jobs = auto                                         # use all cores for parallel builds
cores = 0                                               # 0 = use all cores per job
#build-dir = /tmp/nix-build                             # build on tmpfs if you have RAM to spare

builders-use-substitutes = true

min-free = 1073741824                                   # trigger GC when < 1GB free (in bytes)
max-free = 5368709120                                   # stop GC when 5GB is free
auto-optimise-store = true                              # hardlink identical files in store

sandbox = true
sandbox-fallback = false                                # don't silently disable sandbox on failure
extra-sandbox-paths = /etc/ssl/certs /etc/resolv.conf   # network builds need these

allow-import-from-derivation = false                    # stricter, catches impure eval patterns
pure-eval = false                                       # set true per-command if you want strictness
restrict-eval = false

log-lines = 50                                          # show more context on build failure
show-trace = true                                       # full eval trace on errors (very useful)

keep-outputs = true                                     # keep build deps (useful for dev shells)
keep-derivations = true                                 # keep .drv files for debugging

ssl-cert-file = /etc/ssl/certs/ca-certificates.crt

```

But this file just cannot handle every nix feature, the ni ecosystem has to offer. Sure this is enough to invoke and install packages via the traditional method of nix-env -iA <pkgname> or using nix profile, but we need is a mechanism that allows it tobe declarative.
On non-nixos system the best way to handle this is home-manager module, but before that we need to setup is a flake registry, that defines the sources from where the nix expressions would be allowed for a user.

In my system I prefer to define both the flake registry and home-manager file in a single directory, which is ~/.config/home-manager, though it can be defined in any directory the user decides to.

My complete flake registry

```nix
{
  description = "Home Manager configuration of Pratyush";

  inputs = {
    # Specify the source of Home Manager and Nixpkgs.
    nixpkgs.url = "github:nixos/nixpkgs/nixpkgs-unstable";

    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    nixgl = {
      url = "github:nix-community/nixGL";
      inputs.nixpkgs.follows = "nixpkgs";
    };

     nur = {
      url = "github:nix-community/NUR";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    caelestia-shell = {
      url = "github:caelestia-dots/shell";
      inputs.nixpkgs.follows = "nixpkgs";
    };

  };

  outputs =
    inputs@{ nixpkgs, home-manager, nixgl, nur, caelestia-shell, ... }:
    let
      pkgs = import nixpkgs {
        system = "x86_64-linux";
        #pkgs = nixpkgs.legacyPackages.${system};
        overlays = [ nixgl.overlay ];
      };
    in
    {
      homeConfigurations.honken = home-manager.lib.homeManagerConfiguration {
        inherit pkgs;

        # Specify your home configuration modules here, for example,
        # the path to your home.nix.
        modules = [ ./home.nix ];

        extraSpecialArgs = { inherit inputs; };
      };
    };
}

```

One of the special inputs thats defined here is it defines unstable channel as the nixpkg input source. The home manager module and NUR(nix user repository), and nixgl.
The biggest caveat most of the standalone nix users face is that cli applications run flawlessly like any native package, but the gui applications abort due to non availability of openGL related sources, that are utilised via shared libs in a traditional distriubution.Since every nixpkg is isolated in /nix, there isn't a streamlined way to share them among each other in the nix sandboxed environment, without creating a flake that defines a particular environment.
The NixGL project works as a wrapper to this gui applications to let the nixpkgs access the graphics driver of the system. This can be defined as a generic graphics driver or particularly for Intel via the mesa-opengl, nvidia and even AMD.

More info here: https://github.com/nix-community/nixgl

In a nix shell, the wrapper can be passed along with a gui application say xyz, in the form of nixGL xyz (For applications utilising openGL) or nixVulkan xyz (for modern applications utilising vulkan).

But the issue being home manager being declarative, and usually doesn't allow direct bindings as such, thus it requires a wrapper script be defined.

```nix
{ config, pkgs, inputs,... }:

let
  nixGLWrap = pkg: pkgs.runCommand "${pkg.name}-nixgl" {} ''
    mkdir -p $out/bin $out/share/applications
    for bin in ${pkg}/bin/*; do
      binname=$(basename $bin)
      cat > $out/bin/$binname << EOF
#!/bin/sh
exec ${pkgs.nixgl.nixGLIntel}/bin/nixGLIntel $bin "\$@"
EOF
      chmod +x $out/bin/$binname
    done
    if [ -d ${pkg}/share/applications ]; then
      for d in ${pkg}/share/applications/*.desktop; do
        sed "s|Exec=${pkg}/bin/|Exec=$out/bin/|g" $d \
          > $out/share/applications/$(basename $d)
      done
    fi
  '';
in

{
  home.username = "honken";
  home.homeDirectory = "/home/honken";
  home.stateVersion = "26.05";

  nixpkgs.config.allowUnfree = true;

  home.packages = with pkgs; [
    ripgrep
    fd
    btop
    htop
    thunderbird
    (nixGLWrap librepods)
    gimp
    opencode
    weechat
    aria2
    elinks
    dconf
    qutebrowser
    yt-dlp
    gallery-dl
    (nixGLWrap brave)
    alpaca
    blanket
    libreoffice
    (nixGLWrap jellyfin-desktop)
    baobab
    showtime
    decibels
    (nixGLWrap chromium)
    proton-pass
    gnome-firmware
    gnome-calendar
    gnome-clocks
    gnome-system-monitor
    gnome-weather
    (nixGLWrap sushi)
    (nixGLWrap obs-studio)
    (nixGLWrap vlc)
    gowall
   # fdroidserver
    android-studio
    halloy
  ];

  # Git
  programs.git = {
    enable = true;
    settings.user = {
        name = "Pratyush";
        email = "nandipratyush1917@gmail.com";
        };
  };

  # Shell
  programs.zsh = {
      enable = true;
      autosuggestion.enable = true;
      syntaxHighlighting.enable = true;

      oh-my-zsh = {
      enable = true;
      plugins = [ "git" "docker" ];
      theme = "darkblood";
      };
  };

  # Enable fzf and automatically hook it into Zsh
  programs.fzf = {
    enable = true;
    enableZshIntegration = true;
  };

  wayland.windowManager.sway = {
      systemd.enable = true;
      xwayland = true;
  };

  programs.home-manager.enable = true;
  targets.genericLinux.enable = true;
  xdg.enable = true;
  xdg.mime.enable = true;
  targets.genericLinux.gpu.enable = true;
  targets.genericLinux.nixGL.defaultWrapper = "mesa";
  targets.genericLinux.nixGL.vulkan.enable = true;
  systemd.user.sessionVariables = config.home.sessionVariables;

  # For electron apps to use wayland server instead of xwayland
  home.sessionVariables = {
  NIXOS_OZONE_WL = "1";
  };

  qt.enable = true;
  gtk.enable = true;

}

```

To apply changes

```bash
home-manager switch --flake ~/.config/home-manager#honken
```
The standard command is "home-manager switch --flake <directory name(can be relative or absolute)>#<user>"
Note: Read the output of home-manager switch properly, you may also choose to backup the file with the -b parameter. During home manager switching a lot of times adding newer gui applications requires the graphics driver to be updated.
That is relayed by home-manager which is usually "sudo <path to nix store>".

```bash
┌[honken@Pratyush-PC] [/dev/pts/3]
└[~]> home-manager switch --flake ~/.config/home-manager#honken
warning: Git tree '/home/honken/.config/home-manager' is dirty
evaluation warning: 'system' has been renamed to/replaced by 'stdenv.hostPlatform.system'
Starting Home Manager activation
Activating checkExistingGpuDrivers
GPU drivers require an update, run
  sudo /nix/store/cpj89f3jpz68dg8yrj89qgf2k2jr4mli-non-nixos-gpu/bin/non-nixos-gpu-setup
Activating checkFilesChanged
Activating checkLinkTargets
Please do one of the following:
- In standalone mode, use 'home-manager switch -b backup' to back up files automatically.
- When used as a NixOS or nix-darwin module, set either
  - 'home-manager.backupFileExtension', or
  - 'home-manager.backupCommand',
  to move the file to a new location in the same directory, or run a custom command.
- Set 'force = true' on the related file options to forcefully overwrite the files below. eg. 'xdg.configFile."mimeapps.list".force = true'
Existing file '/home/honken/.config/user-dirs.dirs' would be clobbered
┌[honken@Pratyush-PC] [/dev/pts/3] [1]
└[~]>  sudo /nix/store/cpj89f3jpz68dg8yrj89qgf2k2jr4mli-non-nixos-gpu/bin/non-nixos-gpu-setup
```

As a particular note, unlike NixOS, on non-nixos systems it cannot replicate system level fuctionalities, because other users cannot access the nixpkgs installed by another user, which is a part of nix security framework.
Though every package resides in /nix, these are hashed, alongwith the fact that in NixOS, /etc is just a symlink to a file in defined in the nix store.
Therefore it limits functionalities like a full fledged desktop environment or running systemd service via elevated privileges.

A rather common example is ssl/tls cert error.
In my Gentoo system, after a long time update, ssl and tls certs need manual update,as it gives choices to preserve either the old certs, merge them or completely discard them.
A better way of dealing with such errors are:

An example error - 
```bash

└[~]> nix shell nixpkgs#mpv
warning: unable to download 'https://channels.nixos.org/flake-registry.json': Problem with the SSL CA cert (path? access rights?) (77) error adding trust anchors from file: /etc/ssl/certs/ca-certificates.crt; using cached version
error:
       … while fetching the input 'https://channels.nixos.org/nixpkgs-unstable/nixexprs.tar.xz'
       error: Failed to open archive (Source threw exception: error: unable to download 'https://channels.nixos.org/nixpkgs-unstable/nixexprs.tar.xz': Problem with the SSL CA cert (path? access rights?) (77) error adding trust anchors from file: /etc/ssl/certs/ca-certificates.crt).  

```

To resolve this:
1. check if the newer certs exist and if they aren't empty
2. list all the certs available
3. get info about the certificates
4. Check if the certs work via curl

```bash
[honken@Pratyush-PC] [/dev/pts/0]
└[~]> ls -la /etc/ssl/certs/ca-certificates.crt
wc -l /etc/ssl/certs/ca-certificates.crt
-rw-r--r-- 1 root root 0 Jul 26 15:44 /etc/ssl/certs/ca-certificates.crt
0 /etc/ssl/certs/ca-certificates.crt
┌[honken@Pratyush-PC] [/dev/pts/0]
└[~]> equery list ca-certificates
 * Searching for ca-certificates ...
[IP-] [  ] app-misc/ca-certificates-20260223.3.112.4-r1:0
┌[honken@Pratyush-PC] [/dev/pts/0]
└[~]> stat /etc/ssl/certs/ca-certificates.crt
  File: /etc/ssl/certs/ca-certificates.crt
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: 0,37    Inode: 33467139    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-07-11 21:49:58.032469032 +0530
Modify: 2026-07-26 15:44:46.262194686 +0530
Change: 2026-07-26 15:44:46.262194686 +0530
 Birth: 2026-07-11 21:49:58.032469032 +0530
┌[honken@Pratyush-PC] [/dev/pts/0]
└[~]> curl -v https://channels.nixos.org/flake-registry.json -o /dev/null
* Host channels.nixos.org:443 was resolved.
* IPv6: 2a04:4e42:25::347
* IPv4: 151.101.209.91
* HTTPS-RR: 0 .
*   Trying [2a04:4e42:25::347]:443...
* ALPN: curl offers h2,http/1.1
} [5 bytes data]
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
} [1564 bytes data]
* SSL Trust Anchors:
* error adding trust anchors from file: /etc/ssl/certs/ca-certificates.crt
* SSL Trust Anchors:
* error adding trust anchors from file: /etc/ssl/certs/ca-certificates.crt
* closing connection #0
curl: (77) error adding trust anchors from file: /etc/ssl/certs/ca-certificates.crt
```

In gentoo linux, the upstream solution is to update ca-certificates manually or use dispatch-conf to edit changes in /etc . In general dispatch-conf method is bound to fail here as these certificates are distriburted by Mozilla.
So to check if the certs are actually installed 

```bash
ls /usr/share/ca-certificates/mozilla/ | wc -l
```
If the above number is non-zero that means the certs exist, and are available for the system.

```bash
sudo update-ca-certificates
```
Then the existence and working can be verified via

```bash
[~]> wc -l /etc/ssl/certs/ca-certificates.crt
curl -sI https://channels.nixos.org/flake-registry.json
3532 /etc/ssl/certs/ca-certificates.crt
HTTP/2 200
cache-control: max-age=300
content-security-policy: default-src 'none'; style-src 'unsafe-inline'; sandbox
content-type: text/plain; charset=utf-8
etag: "1aafb6e6c07c1dd62e0e90683d9a5b9ac410cb9628e1cca8f4c8fd275e48e875"
strict-transport-security: max-age=31536000
x-content-type-options: nosniff
x-frame-options: deny
x-xss-protection: 1; mode=block
x-github-request-id: 16E6:261A5D:2A575D:6C7756:6A65FBBD
via: 1.1 varnish, 1.1 varnish
x-timer: S1785068479.888953,VS0,VE216
cross-origin-resource-policy: cross-origin
x-fastly-request-id: d951300431378566cb74e4bebad996b6f8698dee
expires: Sun, 26 Jul 2026 12:26:19 GMT
source-age: 0
access-control-allow-origin: *
accept-ranges: bytes
age: 0
date: Sun, 26 Jul 2026 12:21:19 GMT
x-served-by: cache-bom-vanm7210037-BOM, cache-bom-vanm7210051-BOM
x-cache: MISS, MISS
x-cache-hits: 0, 0
vary: Authorization,Accept-Encoding
content-length: 9653
```

Thankyou !

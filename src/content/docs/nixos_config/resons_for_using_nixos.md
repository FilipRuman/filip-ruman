---
title: Why you should use NixOS
description: Features that makes NixOS the best linux distribution and my story trying linux.
---

If you are thinking about switching to NixOS than you are in the right place.

## Best NixOS Features

### Updates

NixOS uses [two separate main channels](https://nixos.wiki/wiki/Nix_channels)
which allows it to satisfy two different groups of users, users that favor
stability over everything else and users that want bleeding edge features.

> Stable channels (nixos-25.11) provide conservative updates for fixing bugs and
> security vulnerabilities, but do not receive major updates after initial
> release. New stable channels are released every six months.
>
> Unstable channels (nixos-unstable, nixpkgs-unstable) correspond to the main
> development branch (unstable) of Nixpkgs, delivering the latest tested updates
> on a rolling basis.

### Stability

From my almost 3 year experience, with daily driving NixOS on the unstable
branch, the commonest issue with the NixOS is that sometimes on the unstable
branch you can't compile one of the packages. This means that if you don't feel
like helping nixos team with solving the issue than you can just ignore it most
of the time and try updating your OS the next week. Most of the time the issue
will be solved within a one week.

Of course there are sometimes issue with the newer versions of packages. Not
that long ago I had an issue with [SDDM](https://github.com/sddm/sddm) not
working at all. But NixOS has an easy way to solve these kinds of problems- you
can just rollback to the previous generation. When you encounter regression in
any package that you use, just submit an issue to the repo of that package and
than rollback to the previous generation and use this package like nothing has
happened.

### Documentation

The [NixOS wiki](https://nixos.wiki) is a great learinging resource. It has
hight quality information for most of the popular programs. NixOS is popular
enought that a lot of the linux programs have information sections dedicated for
Nixos. You can always use information made for other distros,
[arch wiki](https://wiki.archlinux.org/title/Main_page) a great learining
resource for distros other than arch.

### Packages

[NixOS has the biggest packages repository in the world](https://repology.org/repository/nix_unstable).
If this wasn't enough you can install flatpaks declaratively with
[Nix-flatpak](https://filip-ruman.pages.dev/nixos_config/usefull_flakes/#nix-flatpak)
or run standard Linux executables without any issues with steam-run.

### Easy Customization

To install something like [KDE plasma](https://kde.org/plasma-desktop/) with all
additional apps you can just copy this configuration form the NixOS wiki.

```nix
services = {
  desktopManager.plasma6.enable = true;
  displayManager.sddm.enable = true;
  displayManager.sddm.wayland.enable = true;
};

environment.systemPackages = with pkgs; [
  # KDE Utilities
  kdePackages.discover # Optional: Software center for Flatpaks/firmware updates
  kdePackages.kcalc # Calculator
  kdePackages.kcharselect # Character map
  kdePackages.kclock # Clock app
  kdePackages.kcolorchooser # Color picker
  kdePackages.kolourpaint # Simple paint program
  kdePackages.ksystemlog # System log viewer
  kdePackages.sddm-kcm # SDDM configuration module
  kdiff3 # File/directory comparison tool
  
  # Hardware/System Utilities (Optional)
  kdePackages.isoimagewriter # Write hybrid ISOs to USB
  kdePackages.partitionmanager # Disk and partition management
  hardinfo2 # System benchmarks and hardware info
  wayland-utils # Wayland diagnostic tools
  wl-clipboard # Wayland copy/paste support
  vlc # Media player
];
```

Uninstalling it is as simple as commenting out this or not importing module with
this configuration. Installing and customizing most of the apps on NixOS is as
easy. You can also use
[Home-Manager](https://nix-community.github.io/home-manager/) to manage all the
dot-configs on all of your systems easily and declaratively.

### Easy System Backups

NixOS allows you to avoid most of the issues that would make your system
unusable by storing previous generations so if you were to encouter a fatal
regression you can just move to the previous generation. You can also setup
[btrfs](https://nixos.wiki/wiki/Btrfs) to store snapshots of your system like
you should on other distros. I personally think that this isn't needed on NixOS.

:::warn

Remember to always follow the 3,2,1 rule when handling any important data.

The 3-2-1 backup rule is a data storage strategy that recommends keeping three
copies of your data, on two different types of media, with one copy stored
off-site.

:::

### Quick Setup on a New Machine

[I've written about installing installing my whole configuration with one command in here](https://github.com/nix-community/nixos-anywhere).\
This process is so fast that I can install NixOS form scratch using a live ISO
on my desktop within 20 minutes. NixOS makes it so easy because I just need to
clone my config from public GitHub repo(I don't need to even log into anything)
and rebuild my system. You can even do this remotely through ssh with
[nixos-anywhere](https://github.com/nix-community/nixos-anywhere)

## My Story with Using Linux

I have been using nixos as my daily driver on my stationary pc and laptop for
almost 3 years. Before that I was using windows with a window managers
([whim](https://github.com/dalyIsaac/Whim) being the best working one)
customized terminal and neovim as an editor(before that I was using vs-code with
vim pugin). I was runing this windows setup for a year or so before I started
trying linux. I realy wanted to try out linux because I had many small anoying
poblems with my os. I hoped that I wouldn't have them when runnign linux. So
I've installed linux on vm at first. It was ok but very laggy(virtual box), but
the kde plasma de was looking so good and eaveryone was saying that linux is so
great that I really wanted to install it on the real hardware. I heard from the
[Chris Titus](https://www.youtube.com/@ChrisTitusTech) that the only 3 good
distros are Arch, Debian, and Mint. The Arch was the must fancy and there were
many hyprland setups for it so that's what I've chosen.

At first I was following a guide that was installing arch with commands instead
of just using the arch-install script. Ofcourse I had some weird error that I
could propably debug right now with my linux expirience but back then I wasn't
expirienced at all. The next distro that I've tried was debian but it had
another issue with installing that I honestly can't really remember, I've
debugged it and It installed normally. But I had an issue with an ethernet cable
dying on me and I was debugging my internet for like 2 days straight. It worked
perfectly fine but for some kind of reson when I've switched to linux it decided
to die. I was just annoyed and went back to good old windows for a bit.

Later I've dicovered that there was a simple script for installing arch linux -
arch-install. I used it and It worked like a charm. Happy that I finally managed
to install arch (I think that I had used linux mint on one device at some point
before that and it was working ok). Than I started to install software that I
was using on windows. Most of it installed but with some of them I had a lot of
issues. I also had problem with installing vs-code(I was using it back than) and
other software that I just dont remember anymore. Annoyed with all of this linux
shit I just wanted to use a working system for a bit and moved back to windows
for a year or so. While on windows I found a lot of usefull software and made my
setup almost look like linux: tiling window managers, unity(Fuck unity, to this
day they don't have simple features like ui scaling implementd in their editor
on linux), neovim, configured terminal, etc. This setup was honestly really good
but after using linux windows just felt slugish and window managers that I was
using back than were a lot worse than the linux versions. Also the winget wasn't
as great as packman.Also I wanted to try something new on my pc because the
previous linux experiments were really fun in the end.

So once again I've chosen to use arch so I've run the arch-install but it didn't
work. I think that the issue was that I had windows installed on a second
partition on the same disk and the arch-install script didn't know what to do.
At that point I was really annoyed with arch so I've tried to use debian. It
also couldn't install! At that point really anoyed I remember that thre was
another really popular distro that I heard of, 'NixOS'. I geve linux the last
chance and put NixOS iso on my thubdrive and started installing it, and... It
worked! NixOS was the only distro that installed without any issues this time.
I've started using it and imidietly I was amazed by the amout of packages and
the quality of the documentation. I've configured hyprland on it and was using
nixos for a week or two, but I was just developing a project that needed to be
tested on the pc platform that has the biggest user base - Windows. Another
issue was that thre was a lot of patrst that I didin't fully understand in nixos
so I had many issues with my configuration. Installing hyprland this fast was
also an issue because it needed some software and configuration that I didin't
know about and it lacked some basic functionallity. This wouldn't be an issue if
I were using kde plasma because it is a complete desktop environment.

So I've switched back to windows for about 2-3 months. In this time I've
discovered great windows software like [whim](https://github.com/dalyIsaac/Whim)
or [UniGetUI](https://github.com/Devolutions/UniGetUI). The configuration that I
was using was so good that I didin't reallisticly have any reason to switch back
to linux, but I just wanted to do this because the windows was just getting
worse and worse.

The next time that I've swithced to NixOS I had a configuration that was a mess
but working in the way that I've wanted. This allowed me to research how to
improve it while having a preety nice setup. From that point I haven't got any
reason to switch back to windows. I've been slowly improving my configuration
and making my setup so much better than anything that windows could ever offer.
Having switched to linux alos improved my privacy becuase my os doesn't send any
data and it is really easy to setup an encrypted partition that makes data on
your pc 100% safe(if you have a strong password ) while you are away form it.
Switching to linux also allowed me to discover software that made my workflow a
lot better- yazi, zoxide, etc.

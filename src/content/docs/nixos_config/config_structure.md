# Config Structure

Generally speaking it doesn't matter how your config is structured. It's the
same thing as with programming, if you have a small project than the structure
doesn't matter. But with time your config will certainly grow larger and larger.
Especially if you have multiple machines.

The config structure that I recommend the most is called "Dendritic pattern" It
will allow us to remove most of the boiler plater and write reusable configs
that we can use on all machines.

I really recommend those sources if you want to know more about this:

- [Vimjoyer's vid](https://www.youtube.com/watch?v=-TRbzkw6Hjs)
- [Dendritic design GitHub repo](https://github.com/Doc-Steve/dendritic-design-with-flake-parts)

## Basic Setup

To setup a very basic dendritic pattern project we will use this command:

`nix flake init -t github:vic/flake-file#dendritic` I recommend you to make it
inside a git directory under `/etc/nixos/`. Using git you will be able to easily
sync your config with other devices, and store it safely.

:::Warning

Most 'dumb/random' errors that I had when working with nix config, were because
of git. If files in a subdirectory are not 'visible' when you rebuild your
config, It probably is caused by this directory not being included inside your
git tree. I'm a lazy person so I use
[lazy git](https://github.com/jesseduffield/lazygit) for this.

:::

```bash
.
├── flake.nix
└── modules
    ├── audio.nix
    ├── fish.nix
    ├── hyprland.nix
    ├── otherExampleMoudule.nix
    └── hosts
        └── $host
            ├── $host.nix
            └── hwConfig.nix
```

This Is a basic structure that we will be developing right now Core of it is
created by the previous command. All the nix files will be imported
automatically by the [import tree flake](https://github.com/vic/import-tree)
unless you put "_name" as the name of a file or directory.

### Host Modules

under the "modules" directory create a "hosts" subdirectory.\
We will put there config that will allow specific machines behave in specific
ways, while utilizing the same project as a config.

::: Example

Your homelab doesn't have to have hyprland installed on it(you ssh onto it), but
your laptop does. The same thing will be happenign with many other things like:
hardware config display config(different resoulution) or even mouse sensitivity.

:::

Create another directory with a name of your host(laptop, desktop, homeLab,
etc.) I will reference it with "$host" through this tutorial.

Than in a $host.nix file:

```nix
### ./modules/host/$host/$host.nix

{
  inputs,
  self,
  lib,
  ...
}: {
  flake = {
    # Declear  $host machine
    nixosConfigurations.$host = inputs.nixpkgs.lib.nixosSystem {
      modules = with self.nixosModules; [
        $host
        #hyprland
        otherExampleMoudule
        fish
      ];
    };

    # module for additiona configuration 
    nixosModules.$host = {pkgs, ...}: {
      environment.systemPackages = with pkgs; [
        blender
      ];
    };
  };
}
```

We need to create `nixosConfigurations.$host = inputs.nixpkgs.lib.nixosSystem`
this will allow us to run:
`sudo nixos-rebuild switch --upgrade --flake .#$host"` to use certain host file
as a config on this $host machine.
Than inside of the `inputs.nixpkgs.lib.nixosSystem`, we will be able to import any modules that we will create.
Example of this is the  `flake.nixosModules.$host` module created in the
previous snippet.

#### Hardware Config

We need to copy your hardware config file from the
'/etc/nixos/hardware-config.nix' to the './modules/host/$host/hw-config.nix'
We need to do this so that our config can mout filesystems right and has needed kernel support and more.
The only change we need to do is wrap it in a flake parts module: "flake.nixosModules.$host"
it will have the same name as the module inside $host.nix. This will mean that
flake-file will automatically merge their contents so we don't have to import
those two modules separately.

This is how my hw-config looks:

```nix
{
  flake.nixosModules.desktop = {
    config,
    lib,
    modulesPath,
    ...
  }: {
    imports = [
      (modulesPath + "/installer/scan/not-detected.nix")
    ];

    boot.initrd.availableKernelModules = ["nvme" "xhci_pci" "ahci" "usbhid" "usb_storage" "sd_mod"];
    boot.initrd.kernelModules = [];
    boot.kernelModules = ["kvm-amd"];
    boot.extraModulePackages = [];

    fileSystems."/" = {
      device = "/dev/disk/by-uuid/bac8161a-fcc7-465d-9944-549efd76e1a0";
      fsType = "ext4";
    };

    boot.initrd.luks.devices."luks-9c6c352a-9681-4e52-885f-3a33e9ce92fe".device = "/dev/disk/by-uuid/9c6c352a-9681-4e52-885f-3a33e9ce92fe";

    fileSystems."/boot" = {
      device = "/dev/disk/by-uuid/2BB5-71EB";
      fsType = "vfat";
      options = ["fmask=0077" "dmask=0077"];
    };
    environment.etc."crypttab".text = ''
      data_crypt UUID=e30f59fc-0341-4b86-8cfe-8084ff2a8c1d /root/keys/data.key luks
    '';

    fileSystems."/data" = {
      device = "/dev/mapper/data_crypt";
      fsType = "ext4";
    };

    # swapDevices =
    #   [ { device = "/dev/disk/by-uuid/f923d65d-a077-43e4-9d5e-b9bb33be36f7"; }
    #   ];

    # Enables DHCP on each ethernet and wireless interface. In case of scripted networking
    # (the default) this is the recommended approach. When using systemd-networkd it's
    # still possible to use this option, but it's recommended to use it in conjunction
    # with explicit per-interface declarations with `networking.interfaces.<interface>.useDHCP`.
    networking.useDHCP = lib.mkDefault true;
    # networking.interfaces.enp14s0.useDHCP = lib.mkDefault true;
    # networking.interfaces.wlp13s0.useDHCP = lib.mkDefault true;

    nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
    hardware.cpu.amd.updateMicrocode = lib.mkDefault config.hardware.enableRedistributableFirmware;
  };
}
```

::: Warning

Do not copy **MY** config, it will **100% MAKE YOUR OS UN BOOTABLE**!

:::

### Example Modules

Now I will show how some example NixOS modules would look like, they are taken
straight from my personal config:

#### 1. Enable NVIDIA GPU Support

```nix
### ./modules/nvidia.nix
{
  flake.nixosModules.nvidia = {
    # Enable OpenGL
    hardware.graphics = {
      enable = true;
    };

    #For nixos-unstable, they renamed it
    services.xserver.enable = true;
    services.xserver.videoDrivers = ["nvidia"];

    hardware.nvidia = {
      modesetting.enable = true;
      powerManagement.enable = true;
      # Fine-grained power management. Turns off GPU when not in use.
      # Experimental and only works on modern Nvidia GPUs (Turing or newer).
      powerManagement.finegrained = false;

      # Enable the Nvidia settings menu,
      # accessible via `nvidia-settings`.
      nvidiaSettings = true;

      open = true;
    };
  };
}
```

#### 2. Docker with NVIDIA GPU Pass thru Support

```nix
### ./modules/docker.nix
{
  flake.nixosModules.docker = {pkgs, ...}: {
    hardware.nvidia-container-toolkit.enable = true;
    environment.systemPackages = with pkgs; [
      nvidia-container-toolkit
    ];
    virtualisation.docker = {
      enable = true;
      enableNvidia = true;
      daemon.settings.features.cdi = true;
      enableOnBoot = false; # enable by hand later with the command from hyprland
    };
  };
}
```

As explained before you can import those modules by just putting their name(eg.
docker, nvidia) inside your host module:

```nix
nixosConfigurations.$host = inputs.nixpkgs.lib.nixosSystem {
  modules = with self.nixosModules; [
      nvidia
      docker
      # anythign other 
  ];
}
```

### Adding External Flake Input with 'Flake-File'

Sometimes we need to an external flakes. Example of use case for this is
installing some programs that don't have a normal package or the package doesn't
have all features of a flake. One of the programms that I use a external flake
for is [Zen Browser](https://zen-browser.app/), broser that I mainly use and
highly recommend.

```nix
{inputs, ...}: {
  flake-file.inputs.zen-browser = {
    url = "github:youwen5/zen-browser-flake";
    inputs.nixpkgs.follows = "nixpkgs";
  };
  flake.nixosModules.zen = {pkgs, ...}: {
    environment.systemPackages = [
      inputs.zen-browser.packages.${pkgs.stdenv.hostPlatform.system}.default
    ];
  };
}
```

Flake-file allows us to just place `flake-file.inputs.$AnyNameForThisFlake` in
any module that we want to add this flake's exported properties in our config.\
Than we can use it anywhere by using `inputs.$AnyNameForThisFlake`. We can than
import our module normal as we would with any other module. We can allso
override any part of the flake how we could normally do(eg. set the
`inputs.nixpkgs.follows`).

Than we need to just remember to run: `sudo nix run .#write-flake` before we
rebuild our config, to add any newly added flakes to our source
flake(`./flake.nix`). I've myself added this command to run always when I
rebuild my config, so I don't forget about it.

### Home Manager Modules

If you don't yet know what
[Home-Manager](https://github.com/nix-community/home-manager) is, you hosestly
need to change it right now. Home-Manager is in my opinion the bes nixos
'extension'. It allows you to relayably manage your dotfiles that configure all
of your programs.
[Vimjoyer's video about Home-Manager](https://www.youtube.com/watch?v=a67Sv4Mbxmc)

We can define home manage modules the same as we define NixOS modules with
flake-parts. `flake.homeModules.name` I personally like split home-manager
modules into just 2 types.

- General - needed on all of the machines: `flake.homeModules.general`
- Machine specific - needed on a specific machine `flake.homeModules.$host`

They will than all merge into one module that I can import inside my config, so
that I have only like 3 home manager modules(for me: desktop, laptop, general)

Here is an example home manager flake:

```nix
{
  flake.homeModules.general = {lib, ...}: {
    programs.ghostty.enable = true;
    programs.ghostty.settings = lib.mkForce {
      "background-opacity" = 0.9;
      "background-blur" = true;
      "clipboard-paste-protection" = false;
      "mouse-hide-while-typing" = true;
      "shell-integration" = "fish";
      "font-family" = "MesloLGM Nerd Font";
      "command" = "fish --login --interactive";
      "copy-on-select" = "clipboard";
    };
  };
}
```

#### Making Home Manager Work

Now we need to change our $host module

```nix
### ./modules/host/$host/$host.nix

{
  inputs,
  self,
  lib,
  ...
}: {
  flake-file.inputs.home-manager = {
    url = lib.mkDefault "github:nix-community/home-manager";
    inputs.nixpkgs.follows = "nixpkgs";
  };

  imports = [
    inputs.home-manager.flakeModules.home-manager
  ];
  flake = {
    nixosConfigurations.$host = inputs.nixpkgs.lib.nixosSystem {
            ## nothing changed
    };

    nixosModules.$host = {pkgs, ...}: {

      imports = [
        inputs.home-manager.nixosModules.default
      ];

      home-manager.users.f = {
        imports = with self.homeModules; [
          $host
          general
        ];
        home = {
          stateVersion = "25.11";
          username = "$username";
          homeDirectory = "/home/$username";
        };
      };

      home-manager.backupFileExtension = "home-managebak";
    };
  };
}
```

This will import the home-manager flake and set it up so that it imports all the
home-manager modules.

### Actually Using This Config

I really like having one command to do many things at once. So I've setup
commands for updating my OS. This is really nice, I just rune this command once
a week and I'm suere that eaverything on my pc is up to date and I don't have to
worry about auto updated(Looking at you Macroslop).

- rebuild - quick rebuild of my config. usefull when testing new config /
  installing new app.

TODO: Rephrase this:

- updateNix - updates all things on my machine, also removes old packages for
  backup system versions with: `nix-collect-garbage`, also runs any commands
  that I put inside `/etc/nixos/onUpdate.sh`( usefull for things like updating
  docker containers or doing other not nix related things).

To make my update commands work on all of my machines, I put my host name inside
`/etc/nixos/host.txt` so that my commands may read it.

```nix
# for fish shell
onUpdate = "sudo /etc/nixos/onUpdate.sh";
readHost = "set -g host (cat /etc/nixos/host.txt)";
rebuild = "readHost ; cd /etc/nixos/NNC/ ; sudo nix run .#write-flake ; sudo nixos-rebuild switch --upgrade --flake .#$host";
updateNix = "cd /etc/nixos/NNC/; git pull ; rebuild ; sudo nix flake update ; flatpak update -y ; onUpdate ; cleanup";
cleanup = "sudo nix-collect-garbage --delete-older-than 14d";
```

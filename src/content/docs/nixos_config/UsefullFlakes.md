# Useful Flakes

## nix-flatpak

[Nix-Flatpak](https://github.com/gmodena/nix-flatpak) is a great tool that
allows us to install flatpaks decleratively. We can just write and import a
simple module like this:

```nix
{inputs, ...}: {
  flake-file.inputs.nix-flatpak = {
    url = "github:gmodena/nix-flatpak";
  };

  flake.nixosModules.flatpak = {pkgs, ...}: {
    imports = [inputs.nix-flatpak.nixosModules.nix-flatpak];

    services.flatpak = {
      enable = true;
    };

    systemd.services.flatpak-repo = {
      wantedBy = ["multi-user.target"];
      path = [pkgs.flatpak];
      script = ''
        flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
      '';
    };
  };
}
```

And than we can just import any flathub package at any place like this.

```nix
services.flatpak.packages = [
  "io.ente.auth"
  "com.brave.Browser"
  "com.discordapp.Discord"
  "org.gimp.GIMP"
];
```

`services.flatpak.packages` is a list so we can append it at many places and all
inputs will be merged. You can find IDs of all
[flathub flatpaks in here](https://flathub.org/en) To update all flatpaks on
your system run: `flatpak update -y`

## Steam-Config-Nix

[This flake](https://github.com/different-name/steam-config-nix) allows for easy
management of
[compatibility tools and launch arguments](https://github.com/different-name/steam-config-nix/blob/master/options.md)
for steam games.

```nix
{inputs, ...}: {
  flake-file.inputs.steam-config-nix = {
    url = "github:different-name/steam-config-nix";
    inputs.nixpkgs.follows = "nixpkgs";
  };
  flake.nixosModules.steam = {pkgs, ...}: {
    imports = [inputs.steam-config-nix.nixosModules.default];

    programs = {
      gamemode.enable = true;
      steam = {
        extraCompatPackages = with pkgs; [
          # proton-cachyos-x86_64-v4
          # proton-dw
          # proton-em
          proton-ge-bin
        ];
        protontricks.enable = true;
        package = pkgs.steam.override {
          extraProfile = ''
            export PROTON_ENABLE_WAYLAND=1
            export PROTON_ENABLE_HDR=1
          '';
        };

        gamescopeSession.enable = true;
        enable = true;
        localNetworkGameTransfers.openFirewall = true; # Open ports in the firewall for Steam Local Network Game Transfers
        config = {
          enable = true;
          closeSteam = true;
          defaultCompatTool = "GE-Proton";

          apps = {
            cyberpunk-2077 = {
              id = 1091500;
              compatTool = "GE-Proton";
              launchOptions = {
                env.WINEDLLOVERRIDES = "winmm,version=n,b";
                args = [
                  "--launcher-skip"
                  "-skipStartScreen"
                ];
              };
            };
          };
        };
      };
    };
  };
}
```

## NVF

[as author says](https://github.com/NotAShelf/nvf/tree/main):

> nvf is a highly modular, configurable, extensible and easy to use Neovim
> configuration in Nix. Designed for flexibility and ease of use, nvf allows you
> to easily configure your fully featured Neovim instance with a few lines of
> Nix.

You can check it out by running`nix run github:notashelf/nvf`

This is the nvim distro that I use myself. I've migrated from using basic nvim
config with lua to this because of these 2 main resons:

- Easy package management - nixos is great with package management and lazy &
  mason are not.
- Easy lsp setup - this was especially an issue before nvim added native lsp
  settings.

Enabling support for almost any lang is as simple as putting this in your nvf
config:

```nix
vim.languages.nix.enable = true;
```

If there is no support baked in for any feature you can easily hack it togheder
by installing a normal nixpkg for it and writing some lua code inside:

```nix
vim.luaConfigPre = '' lua code'';
```

You can check its documentation [in here](https://nvf.notashelf.dev/). You can
see my config [in here](https://github.com/FilipRuman/NNC/tree/main/modules/nvf)

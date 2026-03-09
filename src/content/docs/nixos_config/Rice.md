# Rice

I really like to make programs that I use, look and behave in the exact way that
I want. This includes: keyboard, whole desk setup, text editor and OS as a
whole. But I have time constrains and I don't want this customization to take a
lot of my precious time.

On this page I will describe how I have achieved my current configuration
without wasting too much time.

## Stylix

As readme on [their repo](https://github.com/nix-community/stylix) says:

> Stylix is a theming framework for NixOS, Home Manager, nix-darwin, and
> Nix-on-Droid that applies color schemes, wallpapers, and fonts to a wide range
> of applications.
>
> Unlike color scheme utilities such as base16.nix or nix-colors, Stylix goes
> further by applying themes to supported applications, following the "it just
> works" philosophy.

I personally use it mainly to apply sensible settings on most of the apps, by
using the 'autoEnable'. This will enable customiztation for all programs that
have been enabled in your home manager config. You can later easily disable them
in `stylix.targets` in home manager module. Some programs like grub or plymoth
need to be disabled inside the nixos module instead of the home manager one.

You can also easily set a cursor package and size through this

```nix
{inputs, ...}: {
  flake-file.inputs.stylix.url = "github:danth/stylix";

  flake.nixosModules.stylix = {pkgs, ...}: let
    fonts = import ./fonts/_consts.nix {inherit pkgs;};
  in {
    imports = [inputs.stylix.nixosModules.stylix];
    stylix = {
      enable = true;
      # base16Scheme = "${pkgs.base16-schemes}/share/themes/ayu-dark.yaml";
      # base16Scheme = "${pkgs.base16-schemes}/share/themes/tokyo-night.yaml";
      base16Scheme = "${pkgs.base16-schemes}/share/themes/tokyodark-terminal.yaml";

      cursor = {
        size = 16;
        package = pkgs.bibata-cursors;
        name = "Bibata-Modern-Classic";
      };
      autoEnable = true;
      targets = {
        grub.enable = false;
        plymouth.enable = false;
      };
      fonts = {
        sizes = {
          applications = 12;
          terminal = 15;
          desktop = 10;
          popups = 10;
        };
        monospace = {
          package = fonts.codePkg;
          name = fonts.codeName;
        };
        sansSerif = {
          package = fonts.sansSerifPkg;
          name = fonts.sansSerifName;
        };

        emoji = {
          package = fonts.emojiPkg;
          name = fonts.emojiName;
        };
      };
    };
  };

  flake.homeModules.general = {
    stylix.targets = {
      hyprlock.enable = false;
      nvf.enable = false;
    };
  };
}
```

## Waypaper

I really like my wallpapers to change, even if they often don't match completely
match my system theme. This makes my system feel fresh eaverytime I turn on my
pc. I can do this because I have a library of tens of absolute banger 4k+
wallpapers(No I will not share them, finding them by hand is a part of the
experience).

Using waypaper with swaybg as a backend I can eayly make my wallpapers be chosen
randomly eaverytime I open hyprland. I just add to exec-once "waypaper --random"
inside the hyprland's config. You also need to remember to set right setting
inside the waypaper's gui.

## Plymouth

TODO: add gif to my theme

Plymouth allows to display nice looking animation during startup & shutdown of
your pc. This looks nice especially if you have encrypted your disk(as you
should) with a very strong password, so it doesn't look too ugly when you type
your 30 symbol pass.

I myself use rings animation from
[plymoth themes](https://github.com/adi1090x/plymouth-themes?tab=readme-ov-file)

```nix
{
  flake.nixosModules.plymouth = {pkgs, ...}: {
    boot = {
      plymouth = {
        enable = true;
        theme = "rings_2";
        themePackages = with pkgs; [
          (adi1090x-plymouth-themes.override {
            selected_themes = ["rings_2"];
          })
        ];
      };

      # Enable "Silent boot"
      consoleLogLevel = 3;
      initrd.verbose = false;
      kernelParams = [
        "quiet"
        "udev.log_level=3"
        "systemd.show_status=auto"
        "video=3840x2160"
      ];

      # Hide the OS choice for bootloaders.
      # Spam space when booting to show it anyway
      loader.timeout = 0;
    };
    boot.initrd.systemd.enable = true;
  };
}
```

## Hyprland

I use Hyprland as a window manager, mainly because it is really stable and looks
really good. Stylix has support for it but it only sets the window borders
anyway.

My config is really simple, I have no creazy shortcuts. It is just a set of
settings that I've gathered thruout my journey with NixOS, the base was from
some kind of config that I've translated from the native hyrpland config. It is
really bare bone, just some basic rounded borders, small gaps, snap transition
animations.

This part handles inputs that control audio on your system:

```nix
bindel = [
  ", XF86AudioRaiseVolume, exec, wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+"
  ", XF86AudioLowerVolume, exec, wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-"
];
bindl = [
  ", XF86AudioNext,  exec, playerctl next"
  ", XF86AudioPause, exec, playerctl play-pause"
  ", XF86AudioPlay,  exec, playerctl play-pause"
  ", XF86AudioPrev,  exec, playerctl previous"
  ", XF86AudioMute, exec, wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle"
];
```

`xwayland.force_zero_scaling = true;` This is needed because I have a 1.5
scalling on my 4K monitor, so that text is visible clearly. This works great
with wayland apps but things that run on x11 aren't scaling properly and have a
really low resolution, so I have to disable scaling on them and scale them in
their builtin settings.

For 'movement' I use Super + key form home row. This is a lot more ergonomic for
my keyboard layout. To move a window to a workspace I just add SHIFT modifier.

```nix
        "Super, s , workspace, 1"
        "Super SHIFT, s, movetoworkspace, 1"

        "Super, d , workspace, 2"
        "Super SHIFT, d, movetoworkspace, 2"

        "Super, f , workspace, 3"
        "Super SHIFT, f, movetoworkspace, 3"

        "Super,  j , workspace, 4"
        "Super SHIFT, j , movetoworkspace, 4"

        "Super, k , workspace, 5"
        "Super SHIFT, k , movetoworkspace, 5"

        "Super, l , workspace, 9"
        "Super SHIFT, l , movetoworkspace, 9"
```

my whole generic config

```nix
{
  flake.homeModules.general = {
    wayland.windowManager.hyprland.enable = true;
    services.hyprpolkitagent.enable = true;
    wayland.windowManager.hyprland.settings = {
      cursor = {
        no_hardware_cursors = true;
        no_warps = false;
      };
      input.kb_layout = "pl";
      xwayland.force_zero_scaling = true;
      exec-once = [
        "Hypridle"
        "waybar"
        "waypaper --random"
      ];

      bindm = [
        "Super, mouse:272, movewindow"
        "Super, mouse:273, resizewindow"
      ];
    # stylix modifies the qt theme by the qtct by default
      env = [
        "QT_QPA_PLATFORMTHEME,qt6ct"
      ];
      general = {
        gaps_in = 3;
        gaps_out = 3;
        border_size = 2;
        resize_on_border = true;
        allow_tearing = false;
        layout = "master";
      };
      # Handle some multimedia key inputs
      bindel = [
        ", XF86AudioRaiseVolume, exec, wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+"
        ", XF86AudioLowerVolume, exec, wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-"
      ];
      bindl = [
        ", XF86AudioNext,  exec, playerctl next"
        ", XF86AudioPause, exec, playerctl play-pause"
        ", XF86AudioPlay,  exec, playerctl play-pause"
        ", XF86AudioPrev,  exec, playerctl previous"
        ", XF86AudioMute, exec, wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle"
      ];
      decoration = {
        rounding = 2;
        active_opacity = 1.0;
        inactive_opacity = 0.9;
        shadow.enabled = true;
        blur.enabled = true;
      };

      animations = {
        enabled = true;
        bezier = "ease,0.4,0.02,0.21,1";
        animation = [
          "windows, 1, 3.5, ease, slide"
          "windowsOut, 1, 3.5, ease, slide"
          "border, 1, 6, default"
          "fade, 1, 3, ease"
          "workspaces, 1, 0.5, ease"
        ];
      };
      bind = [
        "Super, Space, togglefloating,"
        "Super SHIFT Control_L, Q, exit,"
        "Super SHIFT, C, killactive,"
        "Super Control_L, E, exec,shutdown now"
        "Super Control_L, S, exec,systemctl suspend"
        "Super SHIFT Control_L, R, exec, reboot"

        "Super, G, exec, ghostty"
        "Super, H, exec, rofi -show drun"
        "Super, X, exec, hyprlock"
        "Super, T, exec, /etc/nixos/NNC/utils/record/start.bash"
        "Super SHIFT, T, exec, /etc/nixos/NNC/utils/record/end.bash"
        "Super, O, exec, grimblast -n copy area"
        "Super Control_L, O, exec, grimblast -n edit area"
        ", Print, exec, grimblast copy area"

        "Super , R, fullscreen,"

        "Super, A , workspace, 0"
        "Super SHIFT, A, movetoworkspace, 0"

        "Super, s , workspace, 1"
        "Super SHIFT, s, movetoworkspace, 1"

        "Super, d , workspace, 2"
        "Super SHIFT, d, movetoworkspace, 2"

        "Super, f , workspace, 3"
        "Super SHIFT, f, movetoworkspace, 3"

        "Super,  j , workspace, 4"
        "Super SHIFT, j , movetoworkspace, 4"

        "Super, k , workspace, 5"
        "Super SHIFT, k , movetoworkspace, 5"

        "Super, l , workspace, 9"
        "Super SHIFT, l , movetoworkspace, 9"
      ];
    };
  };
}
```

## Hyprpanel

![My Hyprpanel config](./hyprpanel.png)

I've recently moved from waybar to the hyprpanel. I think that it is a lot less
hacky and works really great out of the box, but at the same time it can be
pretty easily customized. It also works great with stylix.

It comes in with really good looking notifications handler and all buttons on
bar are clickable with custom menus. It generally achieves the main goal that I
have - great looking rice without any effort. It also comes with great working
GUI settings screen.

The only issue I have is customizing it on nixos. To change some settings with
home manager you need to do:

1. Disable homemanager configuration for hyprpanel - otherwise you will not be
   able to see the changed configuration.
2. Rebuild your system to apply the previous change.
3. Restart the hyprpanel.
4. Open GUI setting of hyprpanel and make changes that you want to do in your
   homemanager config.
5. Export the config using a button in setting. TODO: ADD Photo
6. Open the exported config and move it's contents to your .nix config
7. Translate the json to nix and renable the homemanager configuration for
   hyprpanel.
8. Rebuild your system to apply changes.
9. See if you achived desierd results, if not repeat.

my config:

```nix
{
  flake.homeModules.general = {
    programs.hyprpanel = {
      enable = true;
      settings = {
        terminal = "ghostty";
        menus = {
          clock.time.military = false;
          clock.time.hideSeconds = true;
          power.lowBatteryNotification = true;
        };
        bar = {
          launcher.icon = "";
          battery.hideLabelWhenFull = true;
          clock.showTime = true;
          clock.format = "%m %d  %H:%M";
          layouts = {
            "0" = {
              left = ["dashboard" "workspaces" "windowtitle"];
              middle = [];

              right = [
                "volume"
                "network"
                "bluetooth"
                # "systray"
                "clock"
                "notifications"
                "battery"
              ];
            };
          };
        };
        theme.bar = {
          transparent = true;
          margin_sides = "0.5em";
          floating = true;
        };
      };
    };
  };
}
```

## Hyprlock

[Hyprlock](https://github.com/hyprwm/hyprlock/)

> Hyprland's simple, yet multi-threaded and GPU-accelerated screen locking
> utility.

I will not make a screenshot for my config because idk how.

I use it with a combination of [hypridle](https://github.com/hyprwm/hypridle) to
lock my screen when I'm away form my pc. I also have a Super + X shortcut for
quick screen locking. Of course this is not as safe as turning off your PC -
especially if your disk is encrypted.

My Hyprlock config:

```nix
    programs.hyprlock = {
      enable = true;
      settings = {
        general = {
          disable_loading_bar = true;
          hide_cursor = true;
          no_fade_in = false;
          grace = 0;
        };

        background = [
          {
            path = "/home/f/Walpapers/...";
            blur_passes = 0;
            blur_size = 0;

            contrast = 0.8916;
            brightness = 0.8172;
            vibrancy = 0.1696;
            vibrancy_darkness = 0.0;
          }
        ];

        input-field = [
          {
            monitor = "";
            size = "600, 80";
            outline_thickness = 3;
            dots_size = 0.4; # Scale of input-field height, 0.2 - 0.8
            dots_spacing = 0.4; # Scale of dots' absolute size, 0.0 - 1.0
            dots_center = true;
            outer_color = "rgba(0, 0, 0, 0)";
            inner_color = "rgba(255, 255, 255, 0.1)";
            font_color = "rgb(200, 200, 200)";
            fade_on_empty = false;
            font_family = "SF Pro Display Bold";
            placeholder_text = ''<i><span foreground="##ffffff99"> Enter Pass </span></i>'';
            hide_input = false;
            position = "0, -510";
            halign = "center";
            valign = "center";
          }
        ];

        # Time-Hour
        label = [
          {
            monitor = "";
            text = ''cmd[update:1000] echo "<span>$(date +"%I")</span>"'';
            color = ''rgba(255, 255, 255, 1)'';
            font_size = 250;
            font_family = "StretchPro";
            position = "-80, 230";
            halign = "center";
            valign = "center";
          }

          # Time-Minute
          {
            monitor = "";
            text = ''cmd[update:1000] echo "<span>$(date +"%M")</span>"'';
            color = "rgba(147, 196, 255, 1)";
            font_size = 250;
            font_family = "StretchPro";
            position = "0, -20";
            halign = "center";
            valign = "center";
          }

          # Day-Month-Date
          {
            monitor = "";
            text = ''cmd[update:1000] echo -e "$(date +"%d %B, %a.")"'';
            color = ''rgba(255, 255, 255, 100)'';
            font_size = 45;
            font_family = ''Suisse Int'l Mono'';
            position = ''20, -180'';
            halign = ''center'';
            valign = ''center'';
          }

          # USER
          {
            monitor = "";
            text = "    $USER";
            color = "rgba(216, 222, 233, 0.80)";
            outline_thickness = 2;
            dots_size = 0.3; # Scale of input-field height, 0.2 - 0.8
            dots_spacing = 0.3; # Scale of dots' absolute size, 0.0 - 1.0
            dots_center = true;
            font_size = 50;
            font_family = "SF Pro Display Bold";
            position = "0, -400";
            halign = "center";
            valign = "center";
          }
        ];
      };

      settings.general = {
        fractional_scaling = 1;
      };
    };
```

My Hypridle config:

```nix
{
  flake.homeModules.general = {
    services.hypridle.enable = true;
    services.hypridle.settings = {
      general = {
        after_sleep_cmd = "hyprctl dispatch dpms on";
        before_sleep_cmd = "loginctl lock-session"; # lock before suspend.
        ignore_dbus_inhibit = false;
        lock_cmd = "hyprlock";
      };

      listener = [
        {
          timeout = 300;
          on-timeout = "hyprlock";
        }
        {
          timeout = 150; # 2.5min.
          on-timeout = "hyprctl dispatch dpms off"; # screen off when timeout has passed
          on-resume = "hyprctl dispatch dpms on && brightnessctl -r"; # screen on when activity is detected after timeout has fired.
        }
      ];
    };
  };
}
```

## Rofi

[my rofi config](./rofi.png)

> A window switcher, Application launcher and dmenu replacement.

I know that there are many modern alternatives but I just have a working config
for it and, it just works for me. The only mode that I use is dmenu anyway.

## SDDM

[my sddm config](./sddm.png)

[SDDM](https://github.com/sddm/sddm)

> SDDM is a modern display manager for X11 and Wayland sessions aiming to be
> fast, simple and beautiful.

SDDM is a display manager of my choice, mainly because of it's looks and how
popular it is. It is the default display manager on KDE Palsma. I chose the
[sddm-astronaut](https://github.com/Keyitdev/sddm-astronaut-theme)'s blackhole
theme for it, but there are a lot of other good ones. I use this to override the
theme type.

```nix
custom = pkgs.sddm-astronaut.override {
  embeddedTheme = "black_hole";
};
```

If you are curious how I've figured this out, because you can't find anything
about this anywhere: I've opened the sddm-astronaut github repo thru
search.nixos nix packages, than looked in the
[source code of the nix package](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/pkgs/by-name/sd/sddm-astronaut/package.nix#L53).
Than found this part:

```nix
{
  lib,
  stdenvNoCC,
  fetchFromGitHub,
  kdePackages,
  formats,
  themeConfig ? null,
  embeddedTheme ? "astronaut",
}:
```

And thought that the `embeddedTheme` seems like the right setting.

My whole config:

```nix
{
  flake.nixosModules.sddm = {pkgs, ...}: let
    custom = pkgs.sddm-astronaut.override {
      embeddedTheme = "black_hole";
    };
  in {
    environment.systemPackages = [
      custom
    ];

    services.displayManager.sddm = {
      theme = "sddm-astronaut-theme";
      extraPackages = [custom];
      wayland = {
        enable = true;
        compositor = "kwin";
      };
      enable = true;
    };
  };
}
```

::: NOTE[Pro TIP]

If you ever have a problem with sddm or other software that doesn't allow you to
boot into your normal desktop enviorment do following: Press Ctrl + LAlt +
F2/F3/F... to go to another tty. This will show you a 'terminal' window that
allows you to log in and use your pc in a terminal mode. Than you can just
update your config or do anything else to save your pc.

I had before porblem with sddm not working so I had to start hyprland by hand
thru this method. Thankfully this issue was resolved after about 2weeks with a
new update.

:::

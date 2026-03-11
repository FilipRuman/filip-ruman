---
title: 5. Other Tips
description: Other Tips
---

# Other Tips

## Recording

For screen recording I use
[gpu-screen-recorder](https://github.com/BrycensRanch/gpu-screen-recorder-git-copr)
with this command

```bash
gpu-screen-recorder \
        -w screen \
        -f 60 \
        -o "$HOME/Videos/recording_$(date +%F_%H-%M-%S).mp4" &
```

and to end the recording i use: `pkill -f gpu-screen-recorder`. I have binded
those commands to shortcuts inside Hyprland.

## Screenshots

I use [grimblast](https://github.com/hyprwm/contrib/tree/main/grimblast) with
shortcuts like this:

```nix
"Super, O, exec, grimblast -n copy area"
"Super Control_L, O, exec, grimblast -n edit area"
", Print, exec, grimblast copy area"
```

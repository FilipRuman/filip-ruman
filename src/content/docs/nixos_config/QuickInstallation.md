# Quick Installation

Ability to quickly and easily install your config is really important for
obvious reasons. To acomplish this we want to be able to copy configuration,
remove unnecesery files and create additiona data with as few steps as we can.\

You can also use config straight from github:
`sudo nixos-rebuild switch --flake github:<owner>/<repo>#<hostname>`. I don't
recommend this because it makes it harder to modify your config. You would have
to push a new commit every time you try a new change inside of your
configuration.

The easiest way to do this will be storing your config on a public github repo.
This will allow us to clone your config without logging into any account. Also
than you can just put any commands necessary for installation, inside the
readme. If you don't want to use GitHub, you may aswell accomplish this easily
in many other ways.

First of all we need to ensure that we use right shell. Another thing will be
specifying hostname, if you have a similar architecture to me- your rebuild
commands read host name form a file.

snippet from my installation process:

```bash
bash ; export host=<desktop/laptop/server>
```

next we need to clean any unnecessary files inside of the `/etc/nixos/`. To do
this we can just do a simple:

```bash
sudo rm -rf /etc/nixos 
mkdir /etc/nixos/
```

Than if you need we can just paste the `$host` data into the
`/etc/nixos/host.txt` file.

I also need to create a `onUpdate.sh` file, it is needed for running various
tasks when I update my system.

Next thing will be cloning the repo that contains your config. For me it will
be: `sudo git clone https://github.com/FilipRuman/NNC.git`

Than we can just switch to the new config
using:`sudo nixos-rebuild switch --upgrade --flake ".#$host"`

snippet from my installation process:

```bash
sudo rm -rf /etc/nixos 
mkdir /etc/nixos/
cd /etc/nixos || exit
sudo echo "$host" >./host.txt
# On update file
touch onUpdate.sh
sudo chmod +x onUpdate.sh
sudo git clone https://github.com/FilipRuman/NNC.git
cd ./NNC/ || exit
sudo nixos-rebuild switch --upgrade --flake ".#$host"
```

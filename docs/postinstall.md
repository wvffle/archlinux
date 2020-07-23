# Snapshots
```sh
# Install cronie
yay -S cronie
sudo systemctl enable --now cronie

# Install snapper
yay -S snapper snap-pac
sudo snapper -c root create-config /
```

# Network
```sh
sudo systemctl enable --now NetworkManager

# Enable and configure wired/wifi connection
nmtui
```

### NordVPN
```sh
yay -S nordvpn-bin wireguard-tools wireguard-dkms

sudo systemctl enable --now nordvpnd
nordvpn login

nordvpn set technology nordlynx
nordvpn set dns 1.1.1.1

nordvpn connect
```

### DNS over HTTPS, DNSSEC
I'm not using IPv6 but it can be configured in dnscrypt-proxy easily
```sh
yay -S dnscrypt-proxy

sudo -e /etc/dnscrypt-proxy/dnscrypt-proxy.toml

# Configure following values
server_names = ['cloudflare']
listen_addresses = ['127.0.0.1:53']

sudo -e /etc/NetworkManager/conf.d/dns-servers.conf
[global-dns-domain-*]
servers=127.0.0.1

# If using NordVPN
nordvpn set dns 127.0.0.1

sudo systemctl enable --now dnscrypt-proxy
sudo systemctl restart NetworkManager
```

# OhMyZsh and SpaceVim
```sh
# Install oh-my-zsh 
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install SpaceVim
curl -sLf https://spacevim.org/install.sh | bash
```
After installing SpaceVim you should open `nvim` once or twice to install all plugins

### umask
```sh
Prepend `.zshrc` with  `umask 077`
```

# sway
```sh
yay -S sway
```

Now, add following to your `.zshrc`
```sh
# ~/.zshrc

# Wayland flags
function sway() {
  export QT_QPA_PLATFORM=wayland-egl
  export QT_WAYLAND_FORCE_DPI=physical
  export QT_WAYLAND_DISABLE_WINDOWDECORATION=1

  export ECORE_EVAS_ENGINE=wayland_egl
  export ELM_ENGINE=wayland_egl

  export SDL_VIDEODRIVER=wayland

  export _JAVA_AWT_WM_NONREPARENTING=1

  export MOZ_ENABLE_WAYLAND=1

  /usr/bin/sway
}
```

# Nodejs
I've decided to switch to `pnpm` as it is currently only package manager that can store packages in another dir and uses reflinks to link modules. Combine that with btrfs and you have the modules installed only once!
```sh
yay -S pnpm nvm
```

### nvm
I'm using Node Version Nanager for some projects that must use specific node version
```sh
nvm install 14
nvm use 14

mv ~/.nvm ~/.local/share/nvm
```

Fetch chpwd-hook.zsh
```sh
curl -sL https://git.io/JfCQQ > ~/.local/share/nvm/chpwd-hook.zsh
```

Now, add following to your `.zshrc`
```sh
# ~/.zshrc
export NVM_DIR="$HOME/.local/share/nvm"
plugins=(... nvm)
source $NVM_DIR/chpwd-hook.zsh


source ~/.zshrc
```


### pnpm
```sh
yay -S pnpm
mkdir -p /data/pnpm ~/.config/pnpm

echo 'store-dir="/data/pnpm"' > ~/.config/pnpm/npmrc.ini
ln -s ~/.config/pnpm/npmrc.ini ~/.npmrc

pnpm add -g pnpm
yay -Rnscu pnpm
```

### yarn
Alternatively to have all of the modules on the ssd, one can use yarn for that purpose
```sh
npm install -g yarn
yarn global add yarn

rm ~/.yarn
mkdir -p ~/.local/share/yarn/bin

ln -s ~/.config/yarn/global/node_modules/.bin/yarnp ~/.local/share/yarn/bin/yarn
ln -s ~/.config/yarn/global/node_modules/.bin/yarnpkg ~/.local/share/yarn/bin/yarnpkg

Now, add following to your `.zshrc`
```sh
# ~/.zshrc
export PATH="$HOME/.local/share/yarn/bin:$PATH"
plugins=(... yarn)

source ~/.zshrc

npm remove -g yarn
```

# AppArmor
```sh
sudo -e /etc/sbupdate.conf
# Add `apparmor=1 audit=1 lsm=lockdown,yama,apparmor` to the CMDLINE_DEFAULT

yay -S apparmor audit

sudo -e /etc/apparmor/parser.conf

# Uncomment following:
write-cache
Optimize=compress-small

sudo groupadd -r audit
sudo gpasswd -a waff audit

sudo -e /etc/audit/auditd.conf

# Set this value
log_group = audit

sudo systemctl enable apparmor
sudo systemctl enable auditd

echo 'exit aa-notify -p -s 1 -w 60 -f /var/log/audit/audit.log' >> ~/.config/sway/config

reboot
```

# Mumble
Since [mumble#3675](https://github.com/mumble-voip/mumble/pull/3675) will be available since mumble 1.4.0, I had to build mumble-git myself. To do this I've edited PKGBUILD of [mumble-git](https://aur.archlinux.org/packages/mumble-git/) and added `no-jackaudio` to the `CONFIG+=`.

Then after installation of `mumble-git` I've added following bindsyms to the `~/.config/sway/config`
```sh
bindsym --whole-window button9 exec dbus-send --session --type=method_call --dest=net.sourceforge.mumble.mumble / net.sourceforge.mumble.Mumble.startTalking                                                     
bindsym --whole-window --release button9 exec dbus-send --session --type=method_call --dest=net.sourceforge.mumble.mumble / net.sourceforge.mumble.Mumble.stopTalking
```

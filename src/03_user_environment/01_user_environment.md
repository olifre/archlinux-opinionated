# Set up user environment

## Set up shell
```
yay -S zsh
chsh
```
Choose `/usr/bin/zsh`.

Then, install `oh-my-zsh` (see (upstream docs)[https://ohmyz.sh/#install]).

Also, install from AUR:
```
yay -S autojump
```

In `.zshrc`, activate these plugins:
```
plugins=(git github git-extras gitignore svn lol catimg compleat wakeonlan battery autojump colorize web-search dirhistory screen rand-quote hitchhiker)
```

FIXME: May want to adjust theme and plugins and such!

## Keep extended ZSH history
Create `~/.oh-my-zsh/custom/history.zsh` with content:
```
HISTFILE=~/.histfile
HISTSIZE=2000000
SAVEHIST=2000000

## Extended history.
## Instead of just a list of commands, append it with this:
## `:&lt;beginning time since epoch&gt;:&lt;elapsed seconds&gt;:&lt;command&gt;'.
setopt extended_history
```
You may want to save ZSH history periodically, e.g. via Syncthing, useful for easier recovery or to search through history from other machines you use.

Run `crontab -e` as user, and add a line like:
```
*/30   *  * * * cp -a /home/olifre/.histfile /home/olifre/Sync/Sync/ZSH-History/histfile-myhostname
```

## Syncthing setup
Enable the `syncthing-tray-qt6` widget by enabling the tray icon, which automatically causes it to autostart and also starts the setup wizard. 

You might also want to do (should in principle be done by the tray widget):
```
systemctl --user --now enable syncthing
```

## Set up `ssh-agent`
```
systemctl enable --user ssh-agent
```
Edit `~/.zshrc`, set:
```
export SSH_AUTH_SOCK=$XDG_RUNTIME_DIR/ssh-agent.socket
```
Also, edit `~/.config/plasma-workspace/env/ssh-agent.sh` (may need to create directory) and add the same line.

Re-login after this.

## WireGuard VPN
```
yay -S wireguard-tools
nmcli conn import type wireguard file somefilewithoutspaces.conf
```
Note that you may want to adapt the config in NetworkManager graphically afterwards, as VPNs imported this way are autoconnect / always-on by default.

## Other things you might want to do
* Firefox plugins such as:
  * [FoxyProxy](https://addons.mozilla.org/de/firefox/addon/foxyproxy-standard/)
  * [Greasemonkey](https://addons.mozilla.org/de/firefox/addon/greasemonkey/)
  * [Stylus](https://addons.mozilla.org/de/firefox/addon/styl-us/)
  * [KeePassXC-Browser](https://addons.mozilla.org/de/firefox/addon/keepassxc-browser/)
  * [Plasma Integration](https://addons.mozilla.org/de/firefox/addon/plasma-integration/)
  * [Privacy Badger](https://addons.mozilla.org/de/firefox/addon/privacy-badger17/)
  * [uBlock Origin](https://addons.mozilla.org/de/firefox/addon/ublock-origin/)
  * [User-Agent Switcher](https://addons.mozilla.org/de/firefox/addon/user-agent-switcher-revived/)
  * [FoxReplace](https://addons.mozilla.org/de/firefox/addon/foxreplace/)
  * [DownThemAll!](https://addons.mozilla.org/de/firefox/addon/downthemall/)
* Firefox configuration:
  * [User Scripts](https://github.com/olifre/userscripts)
  * [User Styles](https://github.com/olifre/userstyles)
  * May want to set up opening of last opened tabs, disable search results before history results, set as default browser.

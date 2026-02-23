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

In `~/.zshrc`, activate these plugins:
```
plugins=(git github git-extras gitignore svn lol catimg compleat wakeonlan battery autojump colorize web-search dirhistory screen rand-quote hitchhiker)
```

In case you are using `ssh-agent` via `systemd` for `KeepassXC` for example, add to `~/.zshrc`:
```
export SSH_AUTH_SOCK=$XDG_RUNTIME_DIR/ssh-agent.socket
```
This should trigger the user unit via `ssh-agent.socket`. 

You may also want to copy over more things from `~/.oh-my-zsh/custom` of one of your existing machines, e.g. themes etc.

You might also want to add to your `.zshrc`:
```
export SYNCTEX_EDITOR="emacsclient --no-wait +%{line} %{input}"

export LESSCOLORIZER='pygmentize -O style=solarized-dark'
```

You might also want to install:
```
yay -S zsh-syntax-highlighting zsh-autosuggestions
```
and activate these with:
```
cd ~/.oh-my-zsh/custom
ln -s /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.plugin.zsh .
ln -s /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh .
```
and configure by creating `~/.oh-my-zsh/custom/zsh-autosuggestions-config.zsh` with content:
```
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=60'
#ZSH_AUTOSUGGEST_USE_ASYNC=true
ZSH_AUTOSUGGEST_BUFFER_MAX_SIZE=40
```

You may also want to create `~/bin` and copy over files you already have there, and then add to `.zshrc`:
```
export PATH=$PATH:~/bin/
```

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
  * May also want to activate Passkey support in the KeepassXC browser plugin.
  * If you are working with the LHC computing grid, you may want to copy over `~/.globus` and add the certificate to Firefox, and enable a master password there. 
  * You may want to disable password saving in Firefox (in case you use KeepassXC). 
* In KDE/Plasma, you may want to:
  * Create more virtual desktop, e.g. 4 in total. 
  * Edit window decoration => window title bar buttons, add "always on top" and "on all desktops".
  * In keyboard setup, enable the compose key, e.g. re-use the "menu" key for that.

## Set up Git
Install `git-lfs`:
```
yay -S git-lfs
git lfs install
```
You will likely want to use a `~/.gitconfig` like this:
```
[user]
        email = freyermuth@physik.uni-bonn.de
        name = Oliver Freyermuth
		signingkey = PUTYOURGPGKEYHERE
[color]
        ui = auto
        diff = auto
        branch = auto
        interactive = auto
        status = auto
[credential]
        helper = cache --timeout=3600
[core]
        compression = 9
        bigfilethreshold = 180m
        whitespace = blank-at-eof,blank-at-eol,cr-at-eol,indent-with-non-tab,space-before-tab
[push]
        default = simple
        followTags = true
[merge]
[grep]
        lineNumber = on
[diff "bin"]
        textconv = hexdump -v -C
[format]
        pretty = fuller
[filter "lfs"]
        required = true
        clean = git-lfs clean -- %f
        smudge = git-lfs smudge -- %f
        process = git-lfs filter-process
```

## Set up fonts
```
yay -S adobe-source-code-pro-fonts
```
I use this for example in Emacs. 

## Set up `zathura`
Create `~/.config/zathura/zathurarc` with content:
```
#set recolor-keephue true
set recolor-darkcolor "#93A1A1"
set recolor-lightcolor "#002B36"
#set recolor-reverse-video true
#set recolor true
#set render-loading false
set render-loading true
#set render-loading-bg #000000
#set render-loading-fg #000000
set page-cache-size 300
set page-store-threshold 300
#set smooth-scroll true
set font "Source Code Pro normal 7"
set first-page-column 1:2
```

## Configure `yt-dlp`
Create `~/.config/yt-dlp/config` with content:
```
--mtime
```

## Set up `conky`
Install `conky`:
```
yay -S conky
```
Restore your favourite configuration to `~/.config/conky`, then create `~/.config/autostart/conky.desktop` with content:
```
[Desktop Entry]
Type=Application
Name=conky
Exec=conky --daemonize --pause=5
StartupNotify=false
Terminal=false
```
I am using [this theme repo](https://github.com/olifre/conky-themes). 

## Set up Konsole
You may want to copy the built-in profile, then set up infinite history, and also set a transparency of 20 % for the terminal.

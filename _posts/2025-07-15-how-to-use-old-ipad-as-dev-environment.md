---
layout: post
title: "How to use old iPad as a dev environment"
---

### Hello!

This post is the inception of our blog - our first post, and a pretty niche one at that. I'll walk you through how I salvaged my old iPad as a development environment for writing this very post. I hope this will be useful for someone someday.

![ipad](/assets/ios-dev-1.png)

### Why

Very frequently asked question among my friends, and, I have sane reasons why I did this.

1. My main dev environment (MacBook) is currently kidnapped by an Apple Authorized Service Provider. They told me my precious MacBook would be repaired and shipped in 2-3 weeks - no news since then, and the provider didn't offer a loaner laptop.
2. I don't have money for an old machine to code on temporarily, such as old ThinkPad models.
3. The only laptop I have on hand is an "Acer Nitro 5" gaming laptop which lives for approximately 20 nanoseconds when not plugged into the wall.

So I needed a portable solution. The iPhone 15 is the most modern device I have, but the screen is too cramped. The only device left is a 2015 model iPad Pro, which is so old that I can't even upgrade to iOS 17. So, how can we write code on this device as performantly as possible? With a server!

### Prerequisites

- A VPS with decent enough specs - for my setup, tmux + mosh + ssh takes only about 50 MB of RAM. Depending on the coding you do, you'll be fine with the cheapest specs.
- An old device - this can be an iPad, ThinkPad, ancient Babylonian computer that you got as a gift back in 2003, anything with an internet connection.

### wait wait wait, isnt it laggy?

No! Even with cheapest specs, mosh will carry you.

*what is mosh?*
> Remote terminal application that allows roaming, supports intermittent connectivity, and provides intelligent local echo and line editing of user keystrokes.
> Mosh is a replacement for interactive SSH terminals. It's more robust and responsive, especially over Wi-Fi, cellular, and long-distance links.
> Mosh is free software, available for GNU/Linux, BSD, macOS, Solaris, Android, Chrome, and iOS.

*but why mosh over ssh?*

Did you know that Turkey's average internet speed is about 45 Mbps and I'm using 24 Mbps internet? If you didn't know, now you know how I suffer with internet slower than a Citroen Ami. Besides speed, the internet is not stable at all, leading to a complete mess with SSH.

With mosh, when I type in nvim, I don't expect a response back from the server - it's echoed locally. If my internet connection drops, I can continue working, and whenever I reconnect, mosh will fast-forward my work to the server.

So without mosh, this would be unbearable, but right now, I have a decent mobile setup.

### okay... go on.

Okay, great! Now you need a terminal that supports true 256 colors - meaning any modern terminal except Mac's own terminal.

For Linux and Mac, you can't go wrong with Kitty. For iOS (which is my setup), Blink Shell is your best bet. It has a 1-week trial, and the annual subscription is very reasonably priced.

BUT. I'm Cansu and something always goes wrong with me. My old iPad cannot update to iOS 17 and above, and Blink Shell requires iOS 17+. Well, I fixed this temporarily by downloading the application from my iPhone first, then downloading from iPad within the Purchases tab (App Store downloads the latest compatible version for you). This went horribly wrong, as the one-year-outdated version of Blink Shell was crashing at the slightest provocation. So I went with Termius! The free tier is more than enough, and if all you do is mosh-ing, you don't need the pricey tier.

So:
- Unix/Linux/Mac → Kitty
- iOS → Blink Shell if you can, or Termius
- Windows → Windows Terminal

After this, considering you have a fresh Ubuntu/Debian/apt-based server in hand, we'll go from zero.

### Setting up SSH and Mosh

First, let's set up SSH access to your server. Most VPS providers give you root access via SSH keys or password. Once you have basic SSH access:

```bash
# Update your server
sudo apt update && sudo apt upgrade -y

# Install mosh
sudo apt install -y mosh

# Install fail2ban for security (optional but recommended)
sudo apt install -y fail2ban
```

Now, from your iPad terminal, connect using mosh instead of SSH:

```bash
# Instead of: ssh user@your-server-ip
# Use:
mosh user@your-server-ip
```

Mosh uses UDP ports 60000-61000, so make sure your VPS firewall allows these ports. Most providers have this open by default, but if you're having connection issues, check your firewall settings.

### Configuring UFW (Uncomplicated Firewall)

For better security, let's configure UFW to properly manage our server's firewall:

(much needed edit here: do not allow all IP's to connect to these ports. my IP is constantly changing, I am traveling places, so I allowed all connecting IP's.
if you can, while allowing the ports, whitelist only one IP address.)

```bash
# Install UFW (usually pre-installed on Ubuntu)
sudo apt install -y ufw

# Reset UFW to default settings
sudo ufw --force reset

# Set default policies (deny incoming, allow outgoing)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (port 22) - IMPORTANT: do this before enabling UFW
sudo ufw allow ssh

# Allow mosh (UDP ports 60000-61000)
sudo ufw allow 60000:61000/udp

# If you're running a web server, allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable UFW
sudo ufw enable

# Check status
sudo ufw status verbose
```

**Important UFW commands:**
- `sudo ufw status` - Check firewall status
- `sudo ufw allow [port]` - Allow specific port
- `sudo ufw deny [port]` - Deny specific port
- `sudo ufw delete [rule]` - Delete a rule
- `sudo ufw reload` - Reload firewall rules

**Security tip:** Always ensure SSH access is allowed before enabling UFW, or you might lock yourself out of your server!

Great! After you have access to the server with mosh, let's install our tools:
```bash
sudo apt update
```
We'll need:
- Neovim (our text editor)
- Tmux (session management)
- Lazygit 

And any additional dev tools you need. We'll focus on setting up the text editor.

```bash
sudo apt install -y neovim tmux lazygit
```

After installation, start by running:
```bash
tmux
```

### Wait! I'm not familiar with tmux and I don't wanna learn keybinds!

Fair! And you don't even need to learn all the keybinds! All you need is:

(Prefix stands for Ctrl+B by default, just like how the leader key stands for Space in vim)

- Prefix + C for new window
- Prefix + N for next window  
- Prefix + D for detaching from the session

That's it! You can later come back to the same session after detaching with `tmux a` or `tmux at` or `tmux attach`.

### But I still don't wanna use tmux!

Fair! But it's highly suggested for a setup like this. If your internet connection is unstable to the point that mosh can't save you, tmux will always save your sessions. In case of an abrupt disconnect, you can always come back to your previous session with `tmux attach`.

And! If you're a Termius user, or this may also happen with other terminal emulators, you need tmux for nvim colorschemes to work. Some terminals (such as Termius) have weirdly fragile color handling, and without tmux, the terminal emulator's theme will always override the nvim theme, leading to a black/white look without colorscheme.

Plus, it's just really cool to use. All you need is 3 keybinds - you don't even need to configure tmux, trust me!

### okay... go on.

Great! Now after running tmux, you'll see a really ugly default theme at first, like this:
![tmux](https://sysaix.com/wp-content/uploads/2022/12/image-5.png)

So... we need to configure tmux first! But to configure tmux, we need to configure nvim first.

You can follow the instructions of LazyVim, NvChad, LunarVim, AstroVim, any distro you want, but I'm only using the mini.nvim suite. It's a one-file config, just shy of 150 lines of code, and satisfies all my needs. You can access the configuration [right here](https://github.com/caner-cetin/.config/blob/main/nvim/init.lua).

If you aren't following distros and want to use my config:
```bash
# Create the nvim config directory if it doesn't exist
mkdir -p ~/.config/nvim

# Edit the config file
nano ~/.config/nvim/init.lua
```

Copy paste my init.lua, restart nvim, wait for a bit, and then nvim will be configured automatically.

### What's in my nvim configuration?

My configuration is built around the mini.nvim ecosystem, which provides a comprehensive set of plugins in a lightweight package:

**Core functionality:**
- **mini.basics**: Sensible defaults for vim options
- **mini.statusline**: A clean status line
- **mini.pick**: File picker (alternative to telescope for basic needs)
- **mini.completion**: Auto-completion
- **mini.icons**: File icons
- **mini.bufremove**: Safe buffer deletion
- **mini.tabline**: Buffer tabs at the top

**Editor enhancements:**
- **mini.pairs**: Auto-close brackets and quotes
- **mini.surround**: Easily change surrounding characters
- **mini.comment**: Toggle comments with ease
- **mini.indentscope**: Visual indent guides
- **mini.jump2d**: Jump to any position with two keystrokes
- **mini.cursorword**: Highlight word under cursor
- **mini.animate**: Smooth scrolling animations
- **mini.sessions**: Session management

**Language support:**
- **LSP setup**: Go, Python, TypeScript, Lua, and Harper (grammar checking)
- **Treesitter**: Better syntax highlighting
- **Conform**: Auto-formatting on save
- **Trouble**: Better diagnostics display
- **Go.nvim**: Enhanced Go development

**Additional tools:**
- **Telescope**: Advanced file searching
- **Neo-tree**: File explorer
- **Which-key**: Keybinding helper
- **Rainbow CSV**: CSV file highlighting
- **Render-markdown**: Better markdown rendering

**Theme:** Dayfox colorscheme, not too light, not too dark, pleasant for eyes. 

The configuration auto-installs everything on first run, so you don't need to manage plugins manually.

Now you're good to go for nvim, tmux time!

### Installing TPM (Tmux Plugin Manager)

First, we need to install TPM to manage tmux plugins:

```bash
# Clone TPM repository
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

Now let's configure tmux:

```bash
# Edit tmux config (note: it's .tmux.conf, not .tmux.config)
nano ~/.tmux.conf
```

Here's my minimal tmux configuration:

```bash
# TPM (Tmux Plugin Manager)
set -g @plugin 'tmux-plugins/tpm'

# Sensible tmux defaults
set -g @plugin 'tmux-plugins/tmux-sensible'

# Gruvbox theme for tmux
set -g @plugin 'egel/tmux-gruvbox'
set -g @tmux-gruvbox 'dark'

# Fix PATH for tmux
set-environment -g PATH "/usr/local/bin:/bin:/usr/bin"

# Initialize TPM (keep this at the very bottom)
run '~/.tmux/plugins/tpm/tpm'
```

After saving the config, restart tmux or source the config:

```bash
# Either restart tmux or source the config
tmux source-file ~/.tmux.conf
```

Then install the plugins by pressing `Prefix + I` (that's Ctrl+B then I). The plugins will install automatically, and you'll have a beautiful gruvbox-themed tmux with sensible defaults!

### Troubleshooting

**Connection issues:**
- Make sure your VPS allows UDP ports 60000-61000 for mosh
- Check if your local network blocks these ports
- Fall back to SSH if mosh doesn't work: `ssh user@your-server`

**Tmux colors not working:**
- Make sure your terminal supports 256 colors
- Try adding `export TERM=xterm-256color` to your shell profile
- Restart tmux after config changes

**Plugin installation fails:**
- Check internet connection on your server
- Manually clone plugins if TPM fails
- Ensure git is installed: `sudo apt install git`

### Conclusion

And that's it! You now have a fully functional development environment running on your old iPad. The combination of mosh for stable connections, tmux for session management, and a well-configured nvim setup gives you a surprisingly capable coding environment that rivals traditional setups.

Happy coding!

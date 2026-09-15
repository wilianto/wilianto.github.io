---
layout: post
title: "Minimalist Mac Setup as Backend Engineer"
date: 2026-09-16 00:06:36 +0800
categories: macos, developer-tools
permalink: minimalist-mac-setup-as-backend-engineer
---

I recently set up a new MacBook. My goal was not to install every tool available, but to have a small setup that is comfortable for backend development: a useful terminal, containers, Git with SSH signing, and a clean workspace.

My guiding principle is to keep things simple. If I lost my laptop tomorrow and had to buy a new one, I want to be productive again in less than an hour. That means keeping the setup small, repeatable, and limited to tools I actually use. My tool is probably not the most up to date one, but still does the job well.

## Essential applications checklist

- [Google Chrome](https://www.google.com/chrome/) for browsing and testing web applications.
- [Visual Studio Code](https://code.visualstudio.com/download) as my editor.
- [Go](https://go.dev/dl/) for backend development.
- [Obsidian](https://obsidian.md/download) for notes and documentation.

## Homebrew

This is where all started.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

brew version
```

## Zsh and terminal improvements

macOS ships with Zsh. I use [Oh My Zsh](https://ohmyz.sh/) for a small amount of shell customization, plus command suggestions from my history.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
brew install zsh-autosuggestions
code ~/.zshrc
```

After installing `zsh-autosuggestions`, enable it in `.zshrc` by adding it to the plugin list:

```zsh
plugins=(git zsh-autosuggestions)
```

Reload the configuration when finished:

```bash
source ~/.zshrc
```

## macOS defaults

I prefer normal key repeat behavior when holding a key in an editor:

```bash
defaults write -g ApplePressAndHoldEnabled -bool false
```

I also make Finder a little more useful by showing hidden files and the current POSIX path in its title bar:

```bash
defaults write com.apple.finder AppleShowAllFiles YES
defaults write com.apple.finder _FXShowPosixPathInTitle -bool true
killall Finder
```

## Containers and basic tools

For local containers, I use Docker's CLI with Colima. Colima runs the Linux virtual machine, which means I do not need Docker Desktop for a minimal local setup.

```bash
brew install colima docker htop

colima start
docker ps
```

`htop` is not essential, but it is useful when I need a quick view of CPU and memory usage.

## SSH key for GitHub

I use an ED25519 SSH key to authenticate with GitHub.

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh

ssh-keygen -t ed25519 -C "your-email@example.com"
eval "$(ssh-agent -s)"

pbcopy < ~/.ssh/id_ed25519.pub
```

The last command copies the public key. Add it to GitHub in **Settings → SSH and GPG keys**. 

## Git identity and SSH commit signing

Next, configure Git identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

I also use the same SSH key to sign my commits automatically:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

To verify signatures locally, Git needs an allowed-signers file. Create it and add one line containing your identity and **public** key. 

```bash
touch ~/.ssh/allowed_signers
chmod 600 ~/.ssh/allowed_signers
code ~/.ssh/allowed_signers
```

Example format:

```text
your-email@example.com ssh-ed25519 AAAAC3...redacted
```

Tell Git where to find it:

```bash
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

Now create a commit and verify it:

```bash
git log --show-signature
```

Example output:

```text
commit a30d74b1ab99d4554e2a43fe3604bb9915d0948f (HEAD -> main, origin/main, origin/HEAD)
Good "git" signature for Your Name with ED25519 key SHA256:[redacted]
Author: Your Name <[redacted]>
Date:   Wed Sep 16 00:15:08 2026 +0800

    Test commit signing
```

The `Good "git" signature` line confirms that Git could verify the commit signature.

That is enough for my initial setup. I prefer adding tools only after I have a real need for them; it keeps a new machine easy to understand and maintain.

---
layout: post
tags: ["Blog","Shell","Virtualization","Linux"]
title: "Bare-minimum Linux System Setup"
date: 2024-06-02T16:01:17-07:00
summary: My bare-minimum setup script for Linux systems
math: false
draft: false
---


I've gotten a lot of feedback from friends asking about my process for the initial setup of linux servers, and figured I'd write up the essentials to put the information in one place. 

I run a version of this config script on basically every machine I use regularly. This is my "bare minimum" config; just the essentials that I would otherwise have to enter manually to be able to effectively use the system. 

I created a shortcut to make running the script as easy as possible. To use it, just run: 
```sh
sudo su
wget bootstrap.jacobbokor.com -o bootstrap.sh && bash bootstrap.sh
```

Or, without the shortcut: 
```sh
wget https://gist.githubusercontent.com/0xjmux/74adafeb729662564f7c29e549567be4/raw/cfe76546b6bea67881b980eac173862d4a3a9759/base-bootstrap.sh -o bootstrap.sh
sudo bash bootstrap.sh
```
##### [base-bootstrap.sh](https://gist.githubusercontent.com/0xjmux/74adafeb729662564f7c29e549567be4/raw/cfe76546b6bea67881b980eac173862d4a3a9759/base-bootstrap.sh)

```sh
#!/bin/bash
# bootstrap.sh is a basic startup script for Vagrant VMs and other
# development type machines. 
# jbokor, 11-2023

basePackages="tmux vim curl htop gnupg2 git tree software-properties-common lsof"
HOMEDIR="/root"

echo "export PS1='\[\e[31m\]\u@\h:\w\$ \[\e[0m\]'" >> $HOMEDIR/.bashrc
echo "alias sl='ls $LS_OPTIONS'" >> $HOMEDIR/.bashrc
echo "alias ll='ls $LS_OPTIONS -l'" >> $HOMEDIR/.bashrc
echo "alias la='ls $LS_OPTIONS -la'" >> $HOMEDIR/.bashrc
echo "export VISUAL=vim" >> $USER_HOME/.bashrc
echo "export EDITOR=vim" >> $USER_HOME/.bashrc
source $HOMEDIR/.bashrc

# vim setup
touch $HOMEDIR/.vimrc
echo "\" Basic vimrc " >> $HOMEDIR/.vimrc
# spaces instead of tabs
echo "set tabstop=4 shiftwidth=4 expandtab" >> $HOMEDIR/.vimrc
# syntax highlighting
echo "syntax on" >> $HOMEDIR/.vimrc
# fixes coloring issues that occur sometimes in ssh
echo "set background=dark" >> $HOMEDIR/.vimrc
# enables line numbering
echo "set number" >> $HOMEDIR/.vimrc
# prevents possible security issue
echo "set nomodeline" >> $HOMEDIR/.vimrc
echo "root .bashrc and .vimrc updated"

apt update -y
apt upgrade -y
apt install $basePackages -y
```

### Script Explanation
```sh
USER_HOME="/root"
echo "export PS1='\[\e[31m\]\u@\h:\w\$ \[\e[0m\]'" >> $USER_HOME/.bashrc
echo "alias sl='ls $LS_OPTIONS'" >> $USER_HOME/.bashrc
echo "alias ll='ls $LS_OPTIONS -l'" >> $USER_HOME/.bashrc
echo "alias la='ls $LS_OPTIONS -la'" >> $USER_HOME/.bashrc
echo "export VISUAL=vim" >> $USER_HOME/.bashrc
echo "export EDITOR=vim" >> $USER_HOME/.bashrc
source $USER_HOME/.bashrc
```
This makes the root account prompt red and sets up some basic aliases for `ls`. It also sets my default text editor and viewer to vim - there's little more annoying than trying to debug a crash on a new machine, opening up system logs, and having to deal with `nano`. 

The red root prompt is essential - I want to be instantly reminded when I'm running commands as root to prevent any mistakes. The colored prompt also helps delineate between commands and command output, quickly indicating where the output from each command begins and ends.


```sh
# root vim setup
touch $USER_HOME/.vimrc
echo "\" Basic vimrc " >> $USER_HOME/.vimrc
# spaces instead of tabs
echo "set tabstop=4 shiftwidth=4 expandtab" >> $USER_HOME/.vimrc
# syntax highlighting
echo "syntax on" >> $USER_HOME/.vimrc
# fixes coloring issues that occur sometimes in ssh
echo "set background=dark" >> $USER_HOME/.vimrc
# enables line numbering
echo "set number" >> $USER_HOME/.vimrc
# prevents possible security issue
echo "set nomodeline" >> $USER_HOME/.vimrc
```
This is my bare-minimum `.vimrc`; the comments above each line should be plenty explanation as to what it's doing. 


```sh
basePackages="tmux vim curl htop gnupg2 git tree software-properties-common lsof"
# ...
apt update -y
apt upgrade -y
apt install $basePackages -y
```
Installs essential packages, using `apt`. This is the only part of the script that's distro specific; everything else will work on any mostly POSIX-compliant system. Other than `software-properties-common`, the package names for `dnf`/`yum` are the same.

This script is very short, and that's by design. When it comes to setup scripts for things like VMs, less is more; I want the script to have completed by the time I login to the VM over ssh.

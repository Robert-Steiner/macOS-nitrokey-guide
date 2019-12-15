# macOS Nitrokey Pro 2  Guide

This guide shows the setup of a Nitrokey Pro 2 with GnuPG on macOS.

**Content**

- [Prerequisites](#prerequisites)
- [Nitrokey App](#nitrokey-app)
- [Set up GnuPG](#set-up-gnupg)
- [SSH](#ssh)
- Git _(Coming Soon)_
- GitHub / GitLab _(Coming Soon)_
- [Thunderbird](#thunderbird)
- [Git-Tower / Transmit](#git-tower-and-transmit)

## Prerequisites

During the guide you will need to install some additional tools. You will use Homebrew to install 
all of these. If you don't have Homebrew installed on your system yet, follow the 
link below.

[Homebrew | brew.sh](https://brew.sh/index_de)

## Nitrokey App 

If it is your first Nitrokey, you will need to install the Nitrokey App to change the user as well 
as the admin PIN. By default, the user PIN has the value `123456`, and the admin PIN has the value 
`12345678`. 

**Please change the PIN's before you continue!**

**Install Nitrokey App**

```shell
brew tap homebrew/cask-drivers
brew cask install nitrokey
open /Applications/Nitrokey App v1.4.app
```

## Set up GnuPG

GnuPG implements the OpenPGP standard and can be seen here as a middleware between your applications
like Thunderbird or git and your Nitrokey.

**Install gnupg2**

```shell
brew install gnupg2
```

You can test the installation of GnuPG by running:

```shell
gpg --help
```

If you run gpg for the first time, gpg will create the folder `.gnupg` in your home directory.

**Create OpenPGP keys and move them onto your Nitrokey**

The Nitrokey website already provides a very good tutorial about how to create OpenPGP keys and move
them onto your Nitrokey. You can find the tutorial here: 
[OpenPGP Key Generation With Backup | nitrokey.com](https://www.nitrokey.com/de/documentation/openpgp-create-backup)

**Install pinentry-mac**

Next, you need to install [pinentry-mac | github.com](https://github.com/GPGTools/pinentry-mac). 
GnuPG already comes with a `pinentry-program` but this is for terminal use only. Later in this 
tutorial we want to integrate GnuPG with other applications like Thunderbird. In this case you don't
have access to a terminal to enter your user PIN. Therefore you need `pinentry-mac`. `pinentry-mac` 
pops up as an application window where you can enter your PIN.

```shell
brew install pinentry-mac
```

**Create gpg.conf**

Now you have to change the gpg.conf. The config below was taken from
[drduh | github.com](https://github.com/drduh/config/blob/master/gpg.conf) and is a good point to
start with. If you already use your own personal config, please make sure that it contains the 
options `use-agent` and `no-tty` at least. `use-agent` is required for 
[SSH support | nitrokey.com](https://www.nitrokey.com/documentation/applications#os:mac&a:ssh-for-server-administration&p:nitrokey-pro)  and `no-tty` is required by 
[Git-Tower | git-tower.com](https://www.git-tower.com/help/mac/integration/gpg) if you want to use 
gpg within Git-Tower.

```
# https://github.com/drduh/config/blob/master/gpg.conf
# https://www.gnupg.org/documentation/manuals/gnupg/GPG-Configuration-Options.html
# https://www.gnupg.org/documentation/manuals/gnupg/GPG-Esoteric-Options.html
# Use AES256, 192, or 128 as cipher
personal-cipher-preferences AES256 AES192 AES
# Use SHA512, 384, or 256 as digest
personal-digest-preferences SHA512 SHA384 SHA256
# Use ZLIB, BZIP2, ZIP, or no compression
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
# Default preferences for new keys
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
# SHA512 as digest to sign keys
cert-digest-algo SHA512
# SHA512 as digest for symmetric ops
s2k-digest-algo SHA512
# AES256 as cipher for symmetric ops
s2k-cipher-algo AES256
# UTF-8 support for compatibility
charset utf-8
# Show Unix timestamps
fixed-list-mode
# No comments in signature
no-comments
# No version in signature
no-emit-version
# Long hexidecimal key format
keyid-format 0xlong
# Display UID validity
list-options show-uid-validity
verify-options show-uid-validity
# Display all keys and their fingerprints
with-fingerprint
# Display key origins and updates
#with-key-origin
# Cross-certify subkeys are present and valid
require-cross-certification
# Disable caching of passphrase for symmetrical ops
no-symkey-cache
# Disable putting recipient key IDs into messages
throw-keyids
use-agent
# Make sure that the TTY (terminal) is never used for any output. 
no-tty
```

Move the config to `~/.gnupg/`:

```shell
mv gpg.conf ~/.gnupg/
```

## SSH

In this section you will prepare the gpg-agent for ssh support and configure your shell so 
that it will take the gpg-agent as SSH identity agent instead of the default macOS ssh-agent. 

**Create gpg-agent.conf**

Let's start by creating a gpg-agent config. You can use the config below or create your own one.
It is important that your config contains the option `enable-ssh-support` for ssh support. 
At the same time you can already set the `pinentry-program` to `/usr/local/bin/pinentry-mac` if you 
want to integrate gpg in Thunderbird or Git-Tower. 

```
default-cache-ttl 600
max-cache-ttl 7200
enable-ssh-support
pinentry-program /usr/local/bin/pinentry-mac
```

Move the agent config to `~/.gnupg/`:

```shell
mv gpg-agent.conf ~/.gnupg/
```

Reload the gpg-agent to apply the changes:

```shell
gpgconf --reload gpg-agent
```

**Configure your profile**

```shell
unset SSH_AGENT_PID
if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
	export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
fi
```

**Create sshcontrol**

If you have more than one authentication key but you only want to allow a specific one to be used for 
ssh, you can put this key into the `sshcontrol` file. The `sshcontrol` file contains all gpg 
authentication keys that are allowed to be used for ssh by the gpg-agent.

First, we need to get the keygrip of your authentication key:

```shell
gpg -K --with-keygrip 
```

Output:
```
/home/alice/.gnupg/pubring.kbx
-------------------------------
sec>  rsa2048/CB2F38F25B491A54 2019-11-24 [SC]
      Keygrip = D4DF0C35D3E22FA6AC37DA2E54FB03F73616A3CB
      Kartenseriennr. = XXXX XXXXXXXX
uid                [ ultimativ ] Alice <alice@example.org>
ssb>  rsa2048/04BB7F8FDEC5E5D9 2019-11-24 [E]
      Keygrip = 21B2EDF018D7CAF0B45644FDB753DD42307C4425
ssb>  rsa2048/BBB6B86627C2D43A 2019-11-24 [A]
      Keygrip = 2E149DA9C5E46E0DECC6A17EFD8B5FB1DF1E1BAB
```

Add the keygrip of your authentication key `[A]` to your `sshcontrol` file:

```
# List of allowed ssh keys. Only keys present in this file are used
# in the SSH protocol. The ssh-add tool may add new entries to this.

2E149DA9C5E46E0DECC6A17EFD8B5FB1DF1E1BAB
```

Move the `sshcontrol` file to `~/.gnupg/`:

```shell
mv sshcontrol ~/.gnupg/
```

Reload the gpg-agent to apply the changes:

```shell
gpgconf --reload gpg-agent
```

## Git (Coming Soon)

## GitHub / GitLab (Coming Soon)

## Thunderbird

[OpenPGP Email Encryption with Thunderbird | nitrokey.com](https://www.nitrokey.com/documentation/openpgp-thunderbird)

## Git-Tower and Transmit

Git-Tower and Transmit don't have access to the environment variables that are defined in your shell
(`.zshrc`, `.profile`, ` ~/.bashrc`). Therefore, the environment variable `SSH_AUTH_SOCK` will still
point at the default macOS ssh-agent instead of the gpg-agent.

In Git-Tower you can create a `environment.plist` file and set the `SSH_AUTH_SOCK` variable to the 
right ssh-agent:

**Create environment.plist**

```xml
<!-- https://www.git-tower.com/help/mac/integration/environment -->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>SSH_AUTH_SOCK</key>
    <string>~/.gnupg/S.gpg-agent.ssh</string>
  </dict>
</plist>
```

```shell
mv environment.plist ~/Library/Application Support/com.fournova.Tower3/
```

After you have moved the file, you need to restart Git-Tower to apply the changes.

The downside of this procedure is, that this is only a solution for Git-Tower. Transmit will still 
point at the default macOS ssh-agent. A more generic and easier way is to define the `IdentityAgent`
in your `.ssh/config` like:

```
Host github.com
  IdentityAgent ~/.gnupg/S.gpg-agent.ssh
  User git
  ...
```



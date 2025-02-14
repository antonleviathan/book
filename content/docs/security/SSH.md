---
title: Secure Shell
---

# SSH

There are a number of different methods for using SSH with a yubikey. Most
of them however require either proprietary components, or modifications to
servers. Such methods are broken and should not be promoted.

Here we will cover only methods that use standard tools, standard protocols,
and don't allow secret keys to come in contact with system memoy.

Secondly solutions here do not require any server modifications. This is key
because you will not be able to modify all systems you use that provide ssh
interfaces such as Github ssh push.

## PKCS11

With this interface it is possible to generate an ssh private key in a
particular format that can be stored inside a PKCS11 capable device such
as a Yubikey 5.

While this does not offer nearly as many assurances as the GPG setup detailed
below, it is the simplest to setup.

Note: Due to limitations in the PKCS11 spec, it is not possible to generate
keys stronger than 2048 bit RSA with this method. Consider the security
requirements of your organization before using this method.

### Generation

For a set of manual steps on how to set this up see:

[https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html]

To simplify this process consider the following script:

[https://gist.github.com/lrvick/9e9c4641fab07f0b8dde6419f968559f]

### Usage

Since SSH will by default only scan folders such as ~/.ssh/ for keys
you will need to inform it that you wish it to also check for smartcards via
the OpenSC interface.

Before using ssh commands be sure you have started your ssh agent like so:

```
ssh-add -s $OPENSC_LIBS/opensc-pkcs11.so
```

## GPG

### Configure SSH to use your Security Token

This assumes you already have a Security Token configured with a GPG Authentication subkey.

The end result will be that you distribute the SSH Public Key from your Security Token to all VCS systems and servers you normally connect to via SSH. From there you will need to have your key insert it, and tap it once for every connection. You will also be required to tap the key for all SSH Agent forwarding hops removing many of the security issues with traditional ssh agent forwarding on shared systems.

#### Most Linux Distros

If you are using a recent systemd based distribution then all the heavy lifting
is likely already done for you.

We recommend simply adding the following to "/home/$USER/.pam_environment":

```
SSH_AGENT_PID DEFAULT=
SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/gnupg/S.gpg-agent.ssh"
```

#### Mac OS, WSL, Non-Systemd Linux Distros

```
echo >~/.gnupg/gpg-agent.conf <<HERE
enable-ssh-support
keep-display # Avoid issues when not using graphical login managers
HERE
```

Additionally if you are using OSX you will want to set a custom program for pin entry:

```
echo "pinentry-program /usr/local/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
```

Edit your ~/.bash_profile (or similar) and add the following lines to use gpg-agent (and thus the Security Token) as your SSH key daemon:

```
# If connected locally
if [ -z "$SSH_TTY" ]; then

    # setup local gpg-agent with ssh support and save env to fixed location
    # so it can be easily picked up and re-used for multiple terminals
    envfile="$HOME/.gnupg/gpg-agent.env"
    if [[ ! -e "$envfile" ]] || ( \
           # deal with changed socket path in gnupg 2.1.13
           [[ ! -e "$HOME/.gnupg/S.gpg-agent" ]] && \
           [[ ! -e "/var/run/user/$(id -u)/gnupg/S.gpg-agent" ]]
       );
    then
        killall gpg-agent
	# This isn't strictly required but helps prevents issues when trying to
	# mount a socket in the /run tmpfs into a Docker container, and puts
	# the socket in a consistent location under GNUPGHOME
        rm -r /run/user/`id -u`/gnupg
        touch /run/user/`id -u`/gnupg
        gpg-agent --daemon --enable-ssh-support > $envfile
    fi

    # Get latest gpg-agent socket location and expose for use by SSH
    eval "$(cat "$envfile")" && export SSH_AUTH_SOCK

    # Wake up smartcard to avoid races
    gpg --card-status > /dev/null 2>&1

fi

# If running remote via SSH
if [ ! -z "$SSH_TTY" ]; then
    # Copy gpg-socket forwarded from ssh to default location
    # This allows local gpg to be used on the remote system transparently.
    # Strongly discouraged unless GPG managed with a touch-activated GPG
    # smartcard such as a Yubikey 4.
    # Also assumes local .ssh/config contains host block similar to:
    # Host someserver.com
    #     ForwardAgent yes
    #     StreamLocalBindUnlink yes
    #     RemoteForward /home/user/.gnupg/S.gpg-agent.ssh /home/user/.gnupg/S.gpg-agent
    if [ -e $HOME/.gnupg/S.gpg-agent.ssh ]; then
        mv $HOME/.gnupg/S.gpg-agent{.ssh,}
    elif [ -e "/var/run/user/$(id -u)/gnupg/S.gpg-agent" ]; then
        mv /var/run/user/$(id -u)/gnupg/S.gpg-agent{.ssh,}
    fi

    # Ensure existing sessions like screen/tmux get latest ssh auth socket
    # Use fixed location updated on connect so in-memory location always works
    if [ ! -z "$SSH_AUTH_SOCK" -a \
        "$SSH_AUTH_SOCK" != "$HOME/.ssh/agent_sock" ];
    then
        unlink "$HOME/.ssh/agent_sock" 2>/dev/null
        ln -s "$SSH_AUTH_SOCK" "$HOME/.ssh/agent_sock"
    fi
    export SSH_AUTH_SOCK="$HOME/.ssh/agent_sock"
fi
```

Now switch to using this new setup immediately:

```
$ killall gpg-agent
$ source ~/.bash_profile
```

### Get SSH Public Key

You (or anyone that has your GPG public key) can get your SSH key as follows:

```
$ gpg --export-ssh-key you@email.com

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCcjW34jewR+Sgsp1Kn4IVnkb7kxwZCi2gnqMTzSnCg5vABTbG
jFvcjvj1hgD3CbEiuaDkDqCuXnYPJ9MDojwZre/ae0UW/Apy2vAG8gxo8kkXn9fgzJezlW8xjV49sx6AgS6BvOD
iuBjT0rYN3EPQoNvkuqQmukgu+R1naNj4wejHzBRKOSIBqrYHfP6fkYmfpmnPyvgxbyrwmss2KhwwAvYgryx7eH
HRkZtBi0Zb9/KRshvQ89nuXik50sGW6wWZm+6m3v6HNctbOI4TAE4xlJW7alGnNugt3ArQR4ic3BeIuOH2c/il4
4BGwdRIp7XlSkkf8cZX59l9kYLaDzw4Z cardno:000603703024
```

You can paste that into your list of authorized keys on all VCS and systems you have access to.

You should disable any other keys that are not backed by a Security Token.

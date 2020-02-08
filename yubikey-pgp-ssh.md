# Using Yubikeys for SSH and git signing

Written January 2020 by Viktor Ahlqvist. All instructions might be incorrect if
you are using a different version of software. Mac OS is tested on Catalina,
Linux on Debian Buster. See _sources_ for more instructions.

Sources:

- [Esev for Linux][esev]
- [Florin for Mac][florin]
- [Ixdy gist, adding to Florins][ixdy]
- [drduhs YubiKey-Guide][drduh]

## Master keys, subkeys and user ids

Open PGP keys have three parts

- Master key  
  Used to manage subkeys, sign/certify other peoples keys and prove the key
  owner. The master key is normally not needed, should be kept offline and only
  used when creating new subkeys.
- Subkeys  
  Can be used for signing, encrypting and/or authentication. The lifetime and
  purpose of the key is controlled by the master key. Subkeys can exist without
  the master key and if a subkey is stolen it can be revoked by the master key.
  Debian has [more information about subkeys](https://wiki.debian.org/Subkeys).
- User id  
  Used to verify the owner of the key, normally contains the name and e-mail
  address.

Open PGP denotes keys as 

    sec = secret key
    ssb = secret subkey
    pub = public key
    sub = public subkey

Keys can be listed with `gpg --list-secret-keys`. Keys that exists only on the
Yubikey will be noted with a `>` afterwards, i.e. `sec>`.


## Configure the Yubikey itself

The Yubikey hardware itself can be configured with Yubikey manager (ykman). The
cli can be used to disable the OTP codes on touch (recommended if the key is
used on a laptop, otherwise it is very easy to accidentally touch it and spam
codes everywhere).

It can also be used to enable/disable touch for signing/authentication et c.

Installing instructions can be found on the Yubico homepage, but the easiest
way on Debian is installing through apt on Debian/Linux and brew on Mac OS.

### Install on Debian/Linux
```shell
sudo apt install yubikey-manager
```

### Install on Mac OS
```shell
brew install ykman
```

### Disable OTP on touch

To disable the OTP on touch
```shell
ykman config usb --disable OTP
```

## Generate keys on Yubikey or on computer

_Advanced section if the keys are used for encryption/decryption and not just
signing._

Keys can be generated on the Yubikey or on the computer. Advantage of using the
computer is that the master key can be saved on an USB drive and be accessable
to generate new subkeys, while keys can never be retrieved from the Yubikey. If
the key is generated on the Yubikey and used to encrypt something, that data
cannot be decrypted if the Yubikey is stolen or lost.

When using multiple Yubikeys, if the master key is generated on the computer,
the same master key can be used for multiple Yubikey subkeys meaning that they can
all encrypt/decrypt/sign based on the same master.

## Instructions for Debian/Linux

1. Install GnuPG and other relevant packages as described by [drduh]  
```shell
sudo apt install -y wget gnupg2 gnupg-agent dirmngr cryptsetup scdaemon \
pcscd secure-delete hopenpgp-tools yubikey-personalization
```

2. Generate new keys on the Yubikey [(see section below)](#generate-the-keys-on-yubikey)
3. Download [drduhs configuration for gpg-agent][drduh-cc]
```shell
cd ~/.gnupg
wget https://raw.githubusercontent.com/drduh/config/master/gpg-agent.conf
```
4. Replace the SSH key agent, add these lines to the shell rc file (`~/.bashrc`
   or similar) (for older systems, see [drduhs instructions][drduh-ra])
```shell
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

5. (Optional) Enable touch with _ykman_
```shell
ykman openpgp set-touch aut on      # for authentication
ykman openpgp set-touch sig on      # for signing
ykman openpgp set-touch enc on      # for encrypting
```

## Instructions for Mac OS

1. Install GnuPG cli
```shell
brew install gpg
```
2. Generate new keys on the Yubikey [(see section below)](#generate-the-keys-on-yubikey)
3. Install pinentry-mac
```shell
brew install pinentry-mac
```
4. Add a `~/.gnupg/gpg-agent.conf` file to get pin entry to work  
```shell
echo "pinentry-program $(which pinentry)
enable-ssh-support
default-cache-ttl 60
max-cache-ttl 120
" > ~/.gnupg/gpg-agent.conf
```
5. Make sure that GPG agent is started properly, edit `~/.bash_profile` (bash) or `~/.zlogin` (zsh)
```no-highlight
# Yubikey
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```
6. To enable the need to touch the Yubikey as well, install ykman `brew install
   ykman`. This will enforce someone is at the computer when keys are loaded  
```shell
ykman openpgp set-touch aut on      # for authentication
ykman openpgp set-touch sig on      # for signing
ykman openpgp set-touch enc on      # for encrypting
```
7. View the loaded keys with `ssh-add -L` and hopefully there will be a `cardno:` key there.

## Generate the keys on Yubikey

Prerequisite is to have GnuPG command line tool installed (gpg)

Change the pins for the Yubikey by running `gpg --change-pin`.
The default pins are 123456 and 12345678 for user and admin respectively.

Change the key length and type with `gpg --card-edit` and then choose `key-attr`

Generate new keys with `gpg --card-edit` and then `generate` after first
writing `admin` to allow admin commands.

## Manage keys

- List keys for which you have both public and private keys
```shell
gpg --list-secret-keys --keyid-format LONG
```
- Remove a key
```shell
gpg --delete-secret-and-public-key NAME
```

## Sign commits in Git

Export the public part of a key
```shell
gpg --armor --export KEYID
```

The output will be starting as below and can be added to Github.

    -----BEGIN PGP PUBLIC KEY BLOCK-----

    mQINBF4plKkBEAChPe10Oz0UufWBphonktTsgQ90WPR+quhTb1Ctf2xF5r3gyNgC
    zuPv1DHTAzZbHgTCXHoxCyhVn/XYM3kvixTzyRqwl+zP4H0zYhGVeVCVxihGP/oR

Configure git to use a specific key for signing with
```shell
git config --global user.signingkey KEYID
```
Omit the `--global` flag to make it local to a specific repository.

Sign commits with `-S` and tags with `-s`

```shell
git commit -S -a -m "Add all files and sign the commit"
git tag -a '1.0.0' -m 'First release' -s
```

[esev]: https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/
[florin]: https://florin.myip.org/blog/easy-multifactor-authentication-ssh-using-yubikey-neo-tokens
[ixdy]: https://gist.github.com/ixdy/6fdd1ecea5d17479a6b4dab4fe1c17eb
[drduh]: https://github.com/drduh/YubiKey-Guide/
[drduh-ra]: https://github.com/drduh/YubiKey-Guide/#replace-agents
[drduh-cc]: https://github.com/drduh/YubiKey-Guide/#create-configuration

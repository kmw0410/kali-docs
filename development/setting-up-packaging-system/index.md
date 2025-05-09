---
title: Setting up a system for packaging
description:
icon:
weight: 10
author: ["gamb1t",]
---

## VM or install?

In this walkthrough we will be explaining certain things that are only on a VM. It is your choice if you want to install a full Kali system (or if you already have one, if you want to use it) or if you want to use a VM, however keep in mind what commands you're entering if it is an install.

## Setting up the VM

It's important to set up a development environment. The easiest way to go about this is to set up a VM with the [latest Kali image](https://cdimage.kali.org/kali-weekly/) and give it a large filesystem. 80GB+ is good for a few packages at a time, however 200GB+ is recommended if [you are using `mr`](https://gitlab.com/kalilinux/tools/packaging) to download all packaging repositories. Likely, you will not need all of the packages to be downloaded.

## Installing packages

We will install tools that we will use later for packaging. [`packaging-dev`](https://packages.debian.org/sid/packaging-dev) is a metapackage, and will install many of the proper packages that are needed:

```console
kali@kali:~$ sudo apt update
[...]
kali@kali:~$
kali@kali:~$ sudo apt install -y packaging-dev sbuild apt-file gitk git-lfs myrepos
[...]
kali@kali:~$
```

## User accounts and keys

Packaging needs to be done on a non-root user with sudo privileges. _The default Kali user is suitable for this_.

You **must** log out of your account and switch to the new user _(rather than using `su`)_. This is done as some pieces (such as variables that are set) of the following setup require you to be on that account, `su` will not work.

Next, we should generate SSH and GPG keys. These are important for packaging as they will allow us to access our files on GitLab easily and ensure the work is ours. This step is not always necessary, however it is helpful in certain cases. You will know if you need to set up a GPG key, however we recommend setting up an SSH key as it will make the packaging process quicker:

```console
kali@kali:~$ ssh-keygen -t rsa
[...]
kali@kali:~$
kali@kali:~$ gpg --gen-key
gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: directory `/home/packaging/.gnupg' created
gpg: new configuration file `/home/packaging/.gnupg/gpg.conf' created
gpg: WARNING: options in `/home/packaging/.gnupg/gpg.conf' are not yet active during this run
gpg: keyring `/home/packaging/.gnupg/secring.gpg' created
gpg: keyring `/home/packaging/.gnupg/pubring.gpg' created
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
 Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y

You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: First Last
Email address: email@domain.com
Comment:
You selected this USER-ID:
     "First Last <email@domain.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
You need a Passphrase to protect your secret key.

We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

Not enough random bytes available. Please do some other work to give
the OS a chance to collect more entropy! (Need 284 more bytes)

gpg: /home/packaging/.gnupg/trustdb.gpg: trustdb created
gpg: key A123BC4D marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
pub   2048R/1234AB5C 2000-00-00
      Key fingerprint = 12AB 34C4 67DE F890 12G3  H45I 6789 J90K L123 MN4O
uid                  First Last <email@domain.com>
sub   2048R/12345A6B 2000-00-00
kali@kali:~$
```

**Please remember to change "First Last <email@domain.com>" to be your name and email**.

The next step is to add the SSH key to your GitLab account. This can be done in the [keys section](https://gitlab.com/-/profile/keys). Run the commands below to put the key in the copy-paste buffer and paste it on GitLab's web page:

```console
kali@kali:~$ sudo apt install -y xclip
[...]
kali@kali:~$
kali@kali:~$ cat ~/.ssh/id_rsa.pub | xclip
kali@kali:~$
```

## Setting up files

We now need to set up git-buildpackage/[`gbp buildpackage`](https://manpages.debian.org/testing/git-buildpackage/gbp-buildpackage.1.en.html):

```console
kali@kali:~$ cat <<EOF > ~/.gbp.conf
[DEFAULT]
pristine-tar = True
cleaner = /bin/true

[buildpackage]
sign-tags = True
export-dir = $HOME/kali/build-area/
ignore-branch = True
ignore-new = True

[import-orig]
filter-pristine-tar = True

[pq]
patch-numbers = False

[dch]
multimaint-merge = True
ignore-branch = True
EOF
kali@kali:~$
kali@kali:~$ grep -q DEBFULLNAME ~/.profile \
  || echo "export DEBFULLNAME='First Last'" >> ~/.aliases
kali@kali:~$
kali@kali:~$ grep -q DEBEMAIL ~/.profile \
  || echo export DEBEMAIL=email@domain.com >> ~/.aliases
kali@kali:~$
```

**Be sure to replace `email@domain.com` with your email, and ensure it is the same one used with your GPG key, if that was setup**.

We enable `pristine-tar` by default as we will use this tool to (efficiently) store a copy of the upstream tarball in the Git repository. We also set `export-dir` so that package builds happen outside of the git checkout directory.

Below, we're customizing some useful tools provided by the `devscripts` package:

```console
kali@kali:~$ gpg -k

pub   rsa2048 2019-01-01 [SC] [expires: 2021-12-21]
      ABC123DE45678F90123G4567HIJK890LM12345N6
uid           [ultimate] First Last <email@domain.com>
sub   rsa2048 2019-01-01 [E] [expires: 2021-12-21]
kali@kali:~$
kali@kali:~$ cat <<EOF > ~/.devscripts
DEBRELEASE_UPLOADER=dput
DEBRELEASE_DEBS_DIR=$HOME/kali/build-area/
DEBCHANGE_RELEASE_HEURISTIC=changelog
DEBCHANGE_MULTIMAINT_MERGE=yes
DEBCHANGE_PRESERVE=yes
DEBUILD_LINTIAN_OPTS="--color always -I"
DEBCHANGE_AUTO_NMU=no
DEBSIGN_KEYID=ABC123DE45678F90123G4567HIJK890LM12345N6
USCAN_DESTDIR=$HOME/kali/upstream/
EOF
kali@kali:~$
kali@kali:~$ mkdir -pv $HOME/kali/{build-area,upstream}
```

**Be sure to put your own key id in `DEBSIGN_KEYID`. In this example we can see from `gpg -k` that our key is `ABC123DE45678F90123G4567HIJK890LM12345N6`**

You may also want to add the following to your git config:

```console
kali@kali:~$ gpg -k

pub   2048R/A123BC4D 2012-12-07
uid                  First Last <email@domain.com>
sub   2048R/12345A6B 2012-12-07
kali@kali:~$
kali@kali:~$ git config --global user.name "First Last"
kali@kali:~$
kali@kali:~$ git config --global user.email email@domain.com
kali@kali:~$
kali@kali:~$ git config --global user.signingkey ABC123DE45678F90123G4567HIJK890LM12345N6
kali@kali:~$
kali@kali:~$ git config --global commit.gpgsign true
kali@kali:~$
```

**The `user.name` and `user.email` must match your gpg key details (`gpg -k`) or you will get a "Secret Key Not Available" error later on**.
**Be sure to put your own key id in `user.signingkey`. In this example we can see from `gpg -k` that our key is `ABC123DE45678F90123G4567HIJK890LM12345N6`**

We also want to enable a dedicated git merge driver for the `debian/changelog` files:

```console
kali@kali:~$ cat <<EOF >> ~/.gitconfig
[merge "dpkg-mergechangelogs"]
         name = debian/changelog merge driver
         driver = dpkg-mergechangelogs -m %O %A %B %A
EOF
kali@kali:~$
kali@kali:~$ mkdir -pv ~/.config/git/
kali@kali:~$
kali@kali:~$ grep mergechangelogs ~/.config/git/attributes \
  || echo "debian/changelog merge=dpkg-mergechangelogs" >> ~/.config/git/attributes
kali@kali:~$
```

## sbuild

We also will need to set up sbuild. Although this isn't too difficult, it does require some extra setup:

```console
kali@kali:~$ sudo mkdir -pv /srv/chroots/
kali@kali:~$
kali@kali:~$ cd /srv/chroots/
kali@kali:/srv/chroots$ sudo sbuild-createchroot \
  --merged-usr \
  --keyring=/usr/share/keyrings/kali-archive-keyring.gpg \
  --arch=amd64 \
  --components=main,contrib,non-free,non-free-firmware \
  --include=kali-archive-keyring \
  kali-dev \
  kali-dev-amd64-sbuild \
  http://http.kali.org/kali
kali@kali:/srv/chroots$
kali@kali:/srv/chroots$ cd ~/
kali@kali:~$
```

Once that is done, we need to edit `/etc/schroot/chroot.d/kali-dev-amd64-sbuild*`, note that "\*" is used as it will generate the last bit randomly.
_Alternatively, use TAB auto-completion_:

```console
kali@kali:~$ echo "source-root-groups=root,sbuild" | sudo tee -a /etc/schroot/chroot.d/kali-dev-amd64-sbuild*
kali@kali:~$
kali@kali:~$ cat /etc/schroot/chroot.d/kali-dev-amd64-sbuild*
[kali-dev-amd64-sbuild]
description=Debian kali-dev/amd64 autobuilder
groups=root,sbuild
root-groups=root,sbuild
profile=sbuild
type=directory
directory=/srv/chroots/kali-dev-amd64-sbuild
union-type=overlay
source-root-groups=root,sbuild
kali@kali:~$
```

Finally, we just need to add our user to the group and do one last change:

```console
kali@kali:~$ sudo sbuild-adduser $USER
kali@kali:~$
kali@kali:~$ cat <<'EOF' > ~/.sbuildrc
$build_arch_all = 1;
$build_source = 1;
$run_lintian = 1;
$lintian_opts = ['-I'];
EOF
kali@kali:~$
```

_Reboot_

## Approx

When building a package with a sbuild, a lot of time (and bandwidth) is spent downloading the build dependencies. To speed up this step, it's possible to use a caching proxy, such as `approx`:

```console
kali@kali:~$ sudo apt install -y approx
```

After the package is installed, we just need a one-line edit of the configuration file `/etc/approx/approx.conf`, in order to define the remote repository to use for Kali. Add it just below the mappings already defined for Debian, so that your config file `/etc/approx/approx.conf` looks like that:

```
debian          http://ftp.debian.org/debian
debian-security	http://security.debian.org/debian-security
kali            http://kali.download/kali
```

Finally, we just need to add a line of configuration inside our chroot, so that apt is configured to use the proxy:

```console
kali@kali:~$ sudo sbuild-shell source:kali-dev-amd64-sbuild
I: /bin/sh
# echo 'Acquire::HTTP::Proxy "http://localhost:9999";' > /etc/apt/apt.conf.d/01proxy
# exit
kali@kali:~$
```

## Quilt (Managing Patches)

```console
kali@kali:~$ cat << EOF > ~/.quiltrc
export QUILT_PATCHES=debian/patches
QUILT_PUSH_ARGS="--color=auto"
QUILT_DIFF_ARGS="--no-timestamps --no-index -p ab --color=auto"
QUILT_REFRESH_ARGS="--no-timestamps --no-index -p ab"
QUILT_DIFF_OPTS='-p'
EOF
kali@kali:~$
```

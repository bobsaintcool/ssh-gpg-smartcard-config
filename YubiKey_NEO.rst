ssh-gpg-smartcard-config for yubikey-neo
========================================

This document covers the procedure for configurating a yubikey as a gpg smartcard for ssh authentication, it also covers setting the correct serial number on the card. The benefit is a good model for `two-factor authentication <http://en.wikipedia.org/wiki/Two-factor_authentication>`_, something you have and something you know. In this example, there is a token and a passphrase. 

The `YubiKey Neo <https://www.yubico.com/products/yubikey-hardware/yubikey-neo>`_ is used here, though other yubikeys make work

Examples below are using a Fedora 20 x86_64 fresh install, there are other tutorials for other operating systems and keys available online. See the CREDITS section below for alternate tutorials, examples, etc.

Configuring Authentication with GNOME-Shell
-------------------------------------------
To configure authentication using the previously generated GnuPG key, the GNOME-Shell needs some adjustements. With help from several resources, configure the system to allow ``gpg-agent`` to take over ssh authentication.

Certain software must be installed, including utilities for the yubikey ``libyubikey-devel``, ``gnupg2`` (which is probably already installed), ``gnupg2-smime`` and ``pcsc-lite``::

  # sudo yum install ykpers-devel libyubikey-devel libusb-devel autoconf gnugpg gnupg2-smime pcsc-lite
  .. snip ..
  Complete!


Configure GNOME-Shell to use gpg-agent
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Turn off ssh-agent inside gnome-keyring-daemon::

  if [[ $(gconftool-2 --get /apps/gnome-keyring/daemon-components/ssh) != "false" ]]; then
    gconftool-2 --type bool --set /apps/gnome-keyring/daemon-components/ssh false
  fi

Configure gpg to use agent (only for smartcard)::

  $ echo "use-agent" >> ~/.gnupg/gpg.conf

Enable ssh-agent drop in replacement support for gpg-agent::

  $ echo "enable-ssh-support" >> ~/.gnupg/gpg-agent.conf

Allow admin actions on your yubikey::

  $ echo "allow-admin" >>  ~/.gnupg/scdaemon.conf 


Intercept gnome-keyring-daemon and put gpg-agent in place for ssh authentication
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

A rather tricky part of this configuration is to have a simple wrapper script, called `gpg-agent-wrapper <http://blog.flameeyes.eu/2010/08/smart-cards-and-secret-agents>`_. This script is used with thanks from Diego E. Pettenò::

  wget -O ~/.gnupg/gpg-agent-wrapper https://github.com/lfit/ssh-gpg-smartcard-config/raw/master/gpg-agent-wrapper && chmod +x ~/.gnupg/gpg-agent-wrapper 

**NOTE:** The above code has been altered to allow the ``.gpg-agent-info`` to run after SSH_AUTH_SOCK. Please see the CREDITS section below for details.

The above **gpg-agent-wrapper** script is invoked using X and bash (or favorite shell). Please ensure the following files exist as below.

The X session::

  $ cat /etc/X11/xinit/xinitrc.d/01-xsession
  [ -f ${HOME}/.xsession ] && source ${HOME}/.xsession

  $ ls -l /etc/X11/xinit/xinitrc.d/01-xsession
  -rwxr-xr-x. 1 root root 53 Nov 23 10:54 /etc/X11/xinit/xinitrc.d/01-xsession

  $ cat ~/.xsession
  source ${HOME}/.gnupg/gpg-agent-wrapper

The shell rc file::

  $ cat ~/.bashrc
  # .bashrc

  # Source global definitions
  if [ -f /etc/bashrc ]; then
    . /etc/bashrc
  fi

  ..snip..

  # ssh authentication component
  source ${HOME}/.gnupg/gpg-agent-wrapper

  ..snip..


Reload GNOME-Shell So that the gpg-agent stuff above takes effect. 
------------------

Reboot the machine works the best.


Get gpshell etc to fix serial number* 
--------------------------------
#\* not relevant to a consumer edition NEO, it can still be relevant to a developer edition NEO 

Install gpshell binary and libs from tykeal's repo::

  $ sudo yum install http://copr-be.cloud.fedoraproject.org/results/tykeal/GlobalPlatform/fedora-19-x86_64/tykeal-GlobalPlatform-release-0.0.1-1.fc19/tykeal-GlobalPlatform-release-0.0.1-1.fc19.x86_64.rpm

  sudo yum install gpshell gppcscconnectionplugin


Create a gpinstall file::

  cat <<EOF >> gpinstall.txt
  mode_211
  enable_trace
  establish_context
  card_connect
  select -AID a000000003000000
  open_sc -security 1 -keyind 0 -keyver 0 -mac_key 404142434445464748494a4b4c4d4e4f -enc_key 404142434445464748494a4b4c4d4e4f
  delete -AID D2760001240102000000000000010000
  delete -AID D27600012401
  install -file openpgpcard.cap -instParam 00 -priv 00
  card_disconnect
  release_context
  EOF


Get the cap file and place it where gpinstall expects to find it::

  wget -O openpgpcard.cap https://github.com/Yubico/yubico.github.com/raw/master/ykneo-openpgp/releases/ykneo-openpgp-1.0.5.cap


Enable your YubiKey NEO’s Smartcard interface (CCID)::

  ykpersonalize -m82


Reboot your yubikey (remove and reinsert) so that ykneomgr works.


put the correct serial number into gpinstall.txt:: 

  if ykneomgr -s; then
    sed -i "s/^install.*/& -instAID D276000124010200006"$(printf %08d "$(ykneomgr -s)")"0000/" gpinstall.txt
  fi


Flash the card\*::

  gpshell gpinstall.txt

  
#\* WARNING This erases existing keys on the smartcard don't do this if you already have keys you want to keep on the card. 


Setting PINs
------------

Included with the gemalto token and GnuPG Smartcard version 2 should be a document describing the default PIN values. There is a regular PIN, which is used to unlock the token for Signing, Encryption or Authentication. Additionally, there is an Admin PIN, which is used to reset the PIN and/or the Reset Code for the key itself.



Complete these steps for PIN and then Admin Pin
~~~~~~~~~~~~~~~~~
default pins are 123456 and 12345678 respectivly 

::

  $ gpg2 --card-edit
  ..snip..

  gpg/card> admin
  Admin commands are allowed

  gpg/card> passwd
  gpg: OpenPGP card no. D27600012401020000050000158A0000 detected

  1 - change PIN
  2 - unblock PIN
  3 - change Admin PIN
  4 - set the Reset Code
  Q - quit

  Your selection? 3

Enter the Current Admin PIN

.. image:: http://sexysexypenguins.com/misc/gpg-admin.png

Then enter the New Admin PIN twice

.. image:: http://sexysexypenguins.com/misc/gpg-new-admin.png

PIN
~~~

::

  1 - change PIN
  2 - unblock PIN
  3 - change Admin PIN
  4 - set the Reset Code
  Q - quit

  Your selection? 1

Enter the Current PIN

.. image:: http://sexysexypenguins.com/misc/gpg-pin.png

Then enter the New PIN twice

.. image:: http://sexysexypenguins.com/misc/gpg-new-pin.png

**NOTE:** If the Admin PIN has not been entered, it may be required before changes are applied.

Generating an SSH Key using GnuPG
---------------------------------

There are several ways to generate an SSH Key using GnuPG. A common way is to link the new Authentication key to an already existing key::

  $ gpg2 --edit-key 8A8F1D53
  gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
  This is free software: you are free to change and redistribute it.
  There is NO WARRANTY, to the extent permitted by law.

  Secret key is available.

  pub  3072R/8A8F1D53  created: 2012-10-06  expires: never       usage: SC
                     trust: ultimate      validity: ultimate
  sub  3072R/2F15E06B  created: 2012-11-23  expires: 2022-11-21  usage: S
  sub  3072R/EB8B4EBD  created: 2012-11-24  expires: 2022-11-22  usage: E
  sub  3072R/6BB325E9  created: 2012-11-24  expires: 2022-11-22  usage: A
  [ultimate] (1). Clint Savage <herlo1@gmail.com>
  [ultimate] (2)  Clint Savage <herlo@fedoraproject.org>
  [ultimate] (3)  Clint Savage <csavage@linuxfoundation.org>

  gpg>

Once in the ``edit-key`` dialog, create a key on the card::

  gpg> addcardkey
  Signature key ....: 91BC 60CC B9EC 8E73 923A  FC6D 58CD 88A6 2F15 E06B
  Encryption key....: 0CC3 DC3E 0D17 6111 A62B  F656 63C6 4DA9 EB8B 4EBD
  Authentication key: 9EBF A9FE 8AE1 0FEB 1699  CE9A 779F 43D5 EC6F CC13

  Please select the type of key to generate:
     (1) Signature key
     (2) Encryption key
     (3) Authentication key
  Your selection? 3

  gpg: WARNING: such a key has already been stored on the card!

  Replace existing key? (y/N) y
  What keysize do you want for the Authentication key? (3072)
  Key is protected.

  You need a passphrase to unlock the secret key for
  user: "Clint Savage <herlo1@gmail.com>"
  3072-bit RSA key, ID 8A8F1D53, created 2012-10-06

  Please specify how long the key should be valid.
           0 = key does not expire
        <n>  = key expires in n days
        <n>w = key expires in n weeks
        <n>m = key expires in n months
        <n>y = key expires in n years
  Key is valid for? (0) 10y
  Key expires at Mon 21 Nov 2022 05:29:00 PM MST
  Is this correct? (y/N) y
  Really create? (y/N) y
  gpg: Note that the key does not use the suggested creation date

  pub  3072R/8A8F1D53  created: 2012-10-06  expires: never       usage: SC
                       trust: ultimate      validity: ultimate
  sub  3072R/2F15E06B  created: 2012-11-23  expires: 2022-11-21  usage: S
  sub  3072R/EB8B4EBD  created: 2012-11-24  expires: 2022-11-22  usage: E
  sub  3072R/6BB325E9  created: 2012-11-24  expires: 2022-11-22  usage: A

  [ultimate] (1). Clint Savage <herlo1@gmail.com>
  [ultimate] (2)  Clint Savage <herlo@fedoraproject.org>
  [ultimate] (3)  Clint Savage <csavage@linuxfoundation.org>

Upon completion of the key, be sure to save the record to the card and gpg key::

  gpg> save
  $

Verify SSH key is managed via gpg-agent
---------------------------------------

Assuming everything above is configured correctly, a simple test is performed with the SmartCard inserted::

  $ ssh-add -L
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL/XmU......BL0luE= cardno:00050000158A

FILES
-----

`The github repository <https://github.com/herlo/ssh-gpg-smartcard-config/>`_ contains all the files to make the changes above. Please feel free to read through them.

CREDITS
-------

A special thanks to the following people and/or links.

  * `How to use gpg with ssh (with smartcard section) <http://www.programmierecke.net/howto/gpg-ssh.html>`_
  * `The GnuPG Smartcard HOWTO (Advanced Features) <http://www.gnupg.org/howtos/card-howto/en/smartcard-howto-single.html#id2507402>`_
  * `Smart Cards and Secret Agents <http://blog.flameeyes.eu/2010/08/smart-cards-and-secret-agents>`_

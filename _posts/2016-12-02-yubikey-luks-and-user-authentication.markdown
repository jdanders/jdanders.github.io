---
layout: post
title:  "Yubikey for LUKS and user authentication"
date:   2016-12-02 13:15:35 -0600
categories: security
---
I would like my laptop to only decrypt the partition and let me log on if my [yubikey](https://www.yubico.com/products/yubikey-hardware/) is inserted in the USB drive (one factor) and I type in a passphrase (2nd factor).

Once I have typed the password once, that should be good for the entire session. If I really only want one passphrase, it would make most sense to require it at boot, and then allow token-only user login. Optional feature: if the yubikey is removed, lock or shutdown the laptop.

# LUKS key slot and hook

When messing with full disk encryption, one should not limit oneself to a single point of failure. When I encrypted my partition, I used a strong not-memorizeable password that is written down. As long as I have that paper, I can decrypt (or put it into a password vault program like [KeePass](http://keepass.info/)). *Additionally* I will create another key entry in LUKS to decrypt using the yubikey and passphrase. If I removed the original password used in LUKS encryption, I would be locked out if the yubikey were damaged or lost.

I am using Arch Linux and followed [this guide](https://github.com/matildah/yubikey-luks-initcpio) to add my LUKS key. One thing I tried and then decided I didn't like was the requirement to touch the yubikey to log in. With the yubikey nano that I'm using it was too annoying to figure out where to touch it.

Package installation:

```
pacman -S yubikey-personalization

pacman -S yubico-pam
```

### Configure yubikey and passphrase

Configure yubikey for challenge-response mode in slot 2 (leave yubico OTP default in slot 1).
{% highlight shell %}
ykpersonalize -v -2 -ochal-resp -ochal-hmac -ohmac-lt64 -ochal-btn-trig -oserial-api-visible #add -ochal-btn-trig to require button press
{% endhighlight %}

Now add the new key to LUKS. My device is /dev/sdb2, be sure to update the device to whichever is the encrypted on. Check what keys slots are currently used with `cryptsetup luksDump /dev/sdb2`. If you need to clear a slot use `cryptsetup luksKillSlot /dev/sdb2 [slotnum]`. You will need the original LUKS key handy in order to add a new key. The `-S7` selects slot 7.

{% highlight shell %}
read -s PW
# the -2 for slot 2
ykchalresp -2 "$PW" | tr -d '\n' > mykeyfile
cryptsetup luksAddKey /dev/sdb2 mykeyfile -S7
rm mykeyfile
unset PW
{% endhighlight %}

### Add hooks/install to LUKS setup to call yubiluks

Put this file in `/etc/initcpio/hooks/` (with 644 permission)
{% highlight shell %}
#!/usr/bin/ash

run_hook() {
    echo "PLUG IN YOUR YUBIKEY NOW IF YOU WANT TO USE IT"
    read -sp "enter yubikey password followed by enter: " PW
    ykchalresp "$PW" | tr -d '\n' > /crypto_keyfile.bin
    echo
    echo "Password entry complete"
}
{% endhighlight %}

Put this file in `/etc/initcpio/install/` (with 644 permission). Check that the path to `ykchalresp` is correct (the first parameter, don't change second parameter).
{% highlight shell %}
#!/bin/bash

build() {
    add_binary "/usr/bin/ykchalresp" "/bin/ykchalresp"
    add_binary "tr"

    add_runscript
}

help() {
    cat <<HELPEOF
This hook generates a temporary keyfile on the fly for LUKS using a
user-supplied password that is fed to a yubikey in HMAC-SHA1 challenge
response mode. It must be run before the "encrypt" hook.
HELPEOF
}
{% endhighlight %}

Finally, modify the `HOOKS=` line of /etc/mkinitcpio.conf to insert `yubiluks` before `encrypt`. Mine looks like this:

```HOOKS="base udev autodetect modconf block yubiluks encrypt filesystems keyboard fsck"```

When all of those files have been added and updated, you will be ready to test it out. The last step that will commit the changes to your boot image is to run `mkinitcpio`. You should see it mention that build hook yubiluks is added.

{% highlight shell %}
mkinitcpio -p linux
{% endhighlight %}

# User login with Yubico PAM
In Arch Linux, the ultimate PAM authentication control file is `/etc/pam.d/system-auth`. To add challenge response Yubico PAM to system authentication, we will add a line to that file. This is adapted from [Yubico's instructions](https://developers.yubico.com/yubico-pam/Authentication_Using_Challenge-Response.html).

Because this can mess up your ability to log in as any user, you should probably have a root console open somewhere so you can fix any problems.

The first step is to create a file that holds the challenge answer. Unlike the LUKS configuration, this is strictly checking that the yubikey is plugged in and is not authenticating against a password mixed in with the challenge.

{% highlight shell %}
# As root
mkdir /var/yubico
chmod +t /var/yubico/
chmod 777 /var/yubico/
{% endhighlight %}
Now as your user, create the challenge file (using slot 2 again). The created file must be of the form `/var/yubico/[username]-[yubi-serial]`.

{% highlight shell %}
$ ykpamcfg -2 -v -p /var/yubico
...
Stored initial challenge and expected response in '/var/yubico/alice-123456'.
{% endhighlight %}

Now add pam_yubico.so to the pam config. My `/etc/pam.d/system-auth` file had this line:

{% highlight shell %}
auth   required  pam_unix.so   try_first_pass nullok
{% endhighlight %}

I added a line above that so it looked like this:
{% highlight shell %}
auth   sufficient pam_yubico.so mode=challenge-response chalresp_path=/var/yubico
auth   required   pam_unix.so   try_first_pass nullok
{% endhighlight %}

If you want to require the user's password as well as the yubikey, make the pam_yubico module `required` instead of `sufficient`.

As soon as you save the `pam.d` file, the new rules will be applied. Open a new terminal (I did a full terminal by pressing `CTRL-ALT-F2`, you can get back by pressing the FX key that matches your X display, usually F8,F9, or F10-ish) and try to log in with and without the yubikey installed. As with the LUKS encryption, if the yubikey is missing, it will ask for a password.

One additional note: I am using LDE for my desktop and when I log in as my user, it still prompts for a password. I just hit `Enter` and it lets me in if the yubikey is plugged in.


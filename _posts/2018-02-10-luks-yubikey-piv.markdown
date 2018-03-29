---
layout: post
title:  "Yubikey PIV for LUKS"
date:   2018-02-10 16:46:35 -0600
categories: repos
---
This is incomplete, but published just so I don't lose it :)

Get key index:
pkcs15-tool --list-keys

Get public key:
pkcs15-tool --read-public-key 12345 > publickey.pem

Created encrypted keyfile:
dd if=/dev/urandom bs=1 count=245 | openssl rsautl -inkey publickey.pem  -pubin -encrypt -pkcs -out mykey.bin

Test decrypt:
pkcs15-crypt --decipher -R -k 12345 --input mykey.bin | wc -c
should output 256

Add temp file key to LUKS (double check slot selectors before running):
cryptsetup luksAddKey {LUKS_DEVICE} /etc/fstab -S3

Add key to LUKS:
pkcs15-crypt --decipher -R -k 12345 --input mykey.bin | cryptsetup --key-file=/etc/fstab luksAddKey {LUKS_DEVICE} -S7

To be sure it works, delete the current key file:
pkcs15-crypt --decipher -R -k 02 --input mykey.bin | cryptsetup luksKillSlot {LUKS_DEVICE} 3

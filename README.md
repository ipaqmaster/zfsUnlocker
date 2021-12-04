# zfsUnlocker

## About 

A port/rewrite of my local git server's zfsvault project which is a heavy modification of the Archlinux zfs boot hook. Rewritten a bit more modular.

Designed to unlock the root dataset at boot time (but also others if seen) by reaching out to a Hashicorp Vault server. 

## Notes

At the current time this is only designed for mkinitcpio hook use


## How to use it

1. Clone the respository or if available grab the latest stable release
2. run zfsUnlocker/install to copy itself into a new directory /etc/zfsUnlocker with a default zfsUnlocker.conf and to place the two mkinitcpio install and hook files in their correct spots.
3. Edit /etc/mkinitcpio.conf and add `zfsUnlocker` to your hooks.
4. Edit /etc/zfsUnlocker/zfsUnlocker.conf and add your vault server's fqdn, a vault token and change the vaultKvEngineName if your kv engine is not named the default 'kv'. Also change vaultRootSearchPath to match your kv path and if you aren't storing the unlock string under 'passphrase' then change vaultKeyName as well.

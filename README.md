# zfsUnlocker

## About 

A port/rewrite of my local git server's zfsvault project which is a heavy modification of the Archlinux zfs boot hook. Rewritten a bit more modular.

Designed to unlock the root dataset at boot time (but also others if seen) by reaching out to a Hashicorp Vault server. 

## Notes

At the current time this is only designed for mkinitcpio hook use


## How to use it

1. Clone the repository or grab the latest stable release
2. run zfsUnlocker/install.

    This will copy itself into a new directory `/etc/zfsUnlocker` with a config file `/etc/zfsUnlocker/zfsUnlocker.conf`.
    
    The script also places a hook and install file for mkinitcpio to reference.
    
3. Edit `/etc/mkinitcpio.conf` and add `zfsUnlocker` to your hooks. I put it at the end.
4. Optionally edit `/etc/zfsUnlocker/zfsUnlocker.conf` adding your vault server's fqdn and either a vault token or approle role_id/secret_id pair. These can also be passed in kernel arguments if desired. Among other features.
5. Run `mkinitcpio -P` to build an initramfs with this hook, its modules and config inside.

If you're using a kv secret engine which is named something other than `kv` you can modify the vaultKvEngineName variable to match.

If you also aren't storing the secrets directly under kvEngineName/zfs/zpoolNames/datasetNames you can also change `vaultRootSearchPath` to match your secret paths.
    
If you also aren't using the key `passphrase` to store the passphrase value there's `vaultKeyName` too.


## Notes and other features

Configuring the conf file is optional for the most part but has some potentially nifty configuration options among other bits and pieces:

* Network bond creation 
  * Designed for servers which can't just DHCP directly on an interface
* iPhone tethering (Enabled by default)
  * For Internet access via idevicepair, ipheth & usbmuxd
* variable `wirelessNetworks`
  * For defining one or more SSID:PSK combos for wpa_supplicant to try joining during the DHCP section of the script.
* variable `forceInterface`
  * to override the DHCP interface
* ssh passphrase generation and pubkey installation
  * For rare scenarios where a token may have expired for unlocking the machine remotely

The script tries stored vault tokens first and if unsuccessful looks to the kernel arguments for another token or approle role_id/secret_id combo to try.

If kernel arguments are used the script attempts to protect them by changing the permissions on `/proc/cmdline`.

When using kernel arguments to drive the script I also recommend changing your EFI mount's fstab entry to include umask=0077 so regular users can't read your `<EFI>/loader/entries/*.conf` files to obtain the host's vault credentials.

## Hashicorp Vault setup example

Examples assuming a ZFS rootfs system consisting of a zpool named after its shortname and a natively encrypted dataset 'root' (computer-name/root)

### kv configuration

1. `vault secrets enable -version=2 kv`

2. `vault kv put kv/zfs/computer-name/root passphrase='testPa5sphraseH3re'`

### Policy
An example policy which allows reading all of a zpool's dataset keys, if defined.

    echo 'path "kv/data/zfs/computer-name/*" {capabilities = ["read"]}
          path "auth/token/lookup-self"    {capabilities = ["read"]}' | \
    vault policy write zfs/computer-name -


### Token based access

    vault token create -policy=zfs/computer-name -ttl=30d -display-name=computer-name

Specify the resulting string as VAULT_TOKEN=xxx in either `/etc/zfsUnlocker/zfsUnlocker.conf` or in your kernel arguments.

### Approle based access

An example approle granting read access to the above strict single-zpool policy.

Successful auth for this example generates a token valid for one minute or two uses. Exactly enough for the hook to look its own token up (-1) and read exactly one kv secret afterwards (-1) before expiring - or 60 seconds. With the secret_id being reusable infinitely until it expires after 7 days.

In an example where there are more datasets to unlock in the early boot stage with unique passphrases, it may be a better idea to set `token_num_uses=0` for infinite uses until the minute is up and the token expires.

Aptly named after the computer the approle's being made for.

1. `vault auth enable approle`
2. `vault write auth/approle/role/zfs_computer-name secret_id_ttl=7d token_num_uses=2 token_ttl=60s token_max_ttl=60s secret_id_num_uses=0 policies=zfs/computer-name`
3. `vault read     auth/approle/role/zfs_computer-name/role-id # Get the role_id`
4. `vault write -f auth/approle/role/zfs_computer-name/secret-id # Create a secret_id`

Note down the returned role and secret IDs. They can be specified as `role_id=xxx` and `secret_id=xxx` in `/etc/zfsUnlocker/zfsUnlocker.conf` or in your kernel arguments.

### Hook > Vault Testing

Once the script is installed and your VAULT_ADDR,and either a VAULT_TOKEN or role_id&secret_id are added to /etc/zfsUnlocker/zfsUnlocker.conf` the vault hook has a test function we can check with:

    $ sudo /etc/zfsUnlocker/modules.d/20_vault/hook test computer-name/root
    Entering test mode.
    
    		Testing config approle...
    		Got a vault token successfully...
    		Attempting unlock of: computer-name/root
    			Passphrase found for computer-name/root
    			cannot open 'computer-name/root': dataset does not exist

My machine here doesn't actually have a zpool named `computer-name` with a dataset named `root` in it. But the example read was a success and wanted to unlock something!

The script will also consider kernel arguments which can be activated and also tested with a reboot - Otherwise the Vault module of this script can be temporarily modified to include additional arguments on the fly.

Testing 'proposed' kernel args on the fly may be added in a later commit if I find myself testing them frequently enough without wanting to hack the script apart for each try or reboot for each test.
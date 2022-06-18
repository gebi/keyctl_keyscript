# decrypt_keyctl

A passphrase caching script to be used in `/etc/crypttab` on Debian and Ubuntu.
When there are multiple cryptsetup (either plain or LUKS) volumes with the same
passphrase, it is tedious to input the passphrase more than once.

Just add this script as keyscript to your /etc/crypttab and it will cache the
passphrase of all cryptab entries with the same identifier.

Either copy `decrypt_keyctl` into the default search path for keyscripts from
cryptsetup `/lib/cryptdisks/scripts/`. So you can just write `keyscript=decrypt_keyctl`
in `/etc/crypttab`, or use a random path of your choice and give the full path
e.g `keyscript=/sbin/decrypt_keyctl`.

As a countermeasure against you misstyping your password please add `:` to the end of the
first key identifier. That means the script always asks for a passphrase on that
specific key entry.

## Requirements

- Debian cryptsetup package with `/etc/crypttab` handling and keyscript option
    - Tested with Debian Lenny, Squeeze and Sid
- Installed and working keyutils package (keyctl)
    - Needs `CONFIG_KEYS=y` in your kernel configuration


## What For?

The current state for dm-crypt in Linux is that it is single threaded, thus
every dm-crypt mapping only uses a single core for crypto operations.
To use the full power of your many-core processor it is thus necessary to split
the dm-crypt device. For Linux software raid arrays the easiest segmentation is to
just put the dm-crypt layer below the software raid layer.

But with a 5 disk raid5 it is a rather daunting task to input the passphrase five
times.
This is what this keyscripts solve for you.


## Usage

Best shown by example

- 5 disks
- Linux software raid5

```
Layer:
      sda             sdb            sdc ... sde
    +-----------+   +-----------+
    | LUKS      |   | LUKS      |
    | +-------+ |   | +-------+ |
    | | RAID5 | |   | | RAID5 | |
    | | ...   | |   | | ...   | |
```

Crypttab Entries:

```
<target>    <source>    <keyfile>        <options>
sda_crypt   /dev/sda2   main_data_raid:  luks,keyscript=decrypt_keyctl
sdb_crypt   /dev/sdb2   main_data_raid   luks,keyscript=decrypt_keyctl
...
sde_crypt   /dev/sde2   main_data_raid   luks,keyscript=decrypt_keyctl
```

## How does it work

### Crypttab Interface

A keyscript is added to options including a keyfile definition as third
parameter in the crypttab file. The keyscript is called with the keyfile as the
first and only parameter. Additionally there are a few environment variables
set but currently are not used by this keyscript (man 5 crypttab for exact
description).

### Keyscript

Keyctl_keyscript uses the Linux kernel keyring facility to securly cache
passphrases between multiple invocations.
The keyfile parameter from cryptab is used to find the same passphrase between
multiple invocations.

Currently the cache timeout is 60 seconds and not configurable (please report a
bug if it is too low for you).

## Problems

Passphrase is piped between processes and could end up in unsecured memory, thus later swapped to disk!

**Warning**
Use of encrypted swap recommend!

## Hints

To remove all traces of your passphrase from kernel keyring left by this script you may want to cleanup the keyring
afterwards with the following command:

```
sudo keyctl clear @u
```

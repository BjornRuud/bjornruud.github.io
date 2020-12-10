+++
title = "Encrypted backup to cloud using EncFS on OS X"
+++
# Encrypted backup to cloud using EncFS on OS X

Using a file synchronization and backup service in the cloud like [JottaCloud](https://jottacloud.com) or [Dropbox](https://dropbox.com) for backups is convenient, but not as secure as you might think. In order to provide services like file sharing and media preview they have to use an encryption key that they control on their end, which means they can decrypt your data. If you want full privacy you have to be able to supply your own encryption key, and very few cloud storage providers have that option.

You can however work around this problem by encrypting the backup before uploading it to the storage provider. Encrypted archives or disk images can be used and are easy to setup, but another option is to use [EncFS](https://vgough.github.io/encfs/) to create an encrypted view of some part of your filesystem.

Note: This article assumes you know your way around a Terminal window, and it will be easier to follow if you know a thing or two about bash scripting (bonus points for expect/tcl knowledge).


## Install Homebrew

The easiest way to get started is to install EncFS with [Homebrew](http://brew.sh). Many Homebrew formulae require Xcode for compilation, so go to the AppStore and install Xcode. Open Xcode and accept any license agreements that pop up, then quit Xcode.

Next you need to install the Xcode command line tools. They are provided in the Xcode app bundle, and can be quickly installed from the command line. Open Terminal and type the following:

```
$ xcode-select --install
```

Follow the instructions in the popup dialog, and after that you won't have to worry about Xcode again.

Now you must install Homebrew. Paste the following into a terminal (or copy it from the Homebrew website if you are cautious/paranoid).

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Follow the instructions, and then you are ready to brew.


## Install EncFS

Installing EncFS is simple:

```
$ brew install encfs
```

If you are running OS X 10.10 or later you'll get an error about having to manually install `osxfuse`. This is because Yosemite (and later OS X versions) require all kernel extensions to be signed, and Homebrew doesn't do binary signing at the moment. Just follow the instructions provided by Homebrew to manually install `osxfuse`, then do `brew install encfs` again. Type `encfs -v` to verify EncFS is available.

Note: If you're having trouble installing `osxfuse`, check if you already have it installed by looking for a "Fuse for OS X" icon in "System Preferences". If there is such an icon there, open it and press the "Remove" button before trying to install again.


## Configure reverse encryption

EncFS has an option to do "reverse encryption". This creates a folder that acts as an encrypted view of another folder, where the encrypted data is provided on the fly from the unencrypted files. This is the opposite of storing the files in encrypted form and providing an unencrypted view, hence "reverse encryption".

It is simple to configure. Assuming an existing folder called `MyFolder` and you want an encrypted folder called `MyFolderEnc`, try this:

```
$ encfs --reverse MyFolder MyFolderEnc
```

You will be presented with two standard configuration options, `standard` and `paranoid`, and an `expert` option. Standard defaults to AES-192 encryption with mostly default options, while Paranoid defaults to AES-256 with a few security enhancements. Even Standard is overkill for most use cases so select that and provide a password. Take look in the `MyFolderEnc` folder and you'll see encrypted versions of the files in `MyFolder`.

To remove the encrypted view, unmount it:

```
$ umount MyFolderEnc
```


## Encryption options

Using the `expert` setup option is useful to tailor the configuration to certain use cases. Let's take a look at the options that can help with performance and portability for the backup scenario described in this article.

- **Algorithm**. Choose AES.
- **Key size**. You can choose between 128, 196 and 256 bits. All are considered secure. Lower key size gives better performance, although it's not a huge difference on modern hardware. I usually choose 128 for data that needs to be private but isn't considered secret.
- **Filesystem block size**. The default is fine. Set it to the block size of your filesystem if you want to optimize. Use `diskutil info /` and search for "Allocation Block Size".
- **Filename encoding**. Go for "Block32". The block encodings make it impossible to determine or approximate original filename length, and the "Block32" variant is appropriate for case-insensitive or case-aware filesystems like HFS+.
- **Per-file initialization vectors**. You want this, or else duplicate files will be the same after encryption, which is an unnecessary information leak.

Setting up encryption with EncFS creates a `.encfs6.xml` file in the source folder with the configured settings and the encryption key. Make a backup of this file and store it safely. If the config file is lost you won't be able to restore the backup!


## Mount shares and encrypted folders

Let's assume you have a NAS at home with Samba shares `share1` and `share2` that you want to backup to some cloud storage provider, and that you want the shares and encrypted folders mounted on login. The encrypted folders will be mounted to `share1-encfs` and `share2-encfs`.

First you test the setup and prepare the EncFS config file:

```
$ cd
$ mkdir -p Volumes/share1 && cd Volumes
$ mount -t smbfs "//user@server/share1" "$HOME/Volumes/share1"
[... Type password ...]
$ ls share1 # Verify contents are there
$ mkdir share1-encfs
$ encfs --reverse "$HOME/Volumes/share1" "$HOME/Volumes/share1-encfs"
[... Configure EncFS settings ...]
$ ls share1-encfs # Verify encrypted contents are there
```

Repeat this for all shares you want to encrypt.

Both Samba and EncFS will require passwords, and you store those in the login keychain for easy (but secure) access from scripts. Open the "Keychain Access" app, select the "login" keychain and the "Passwords" category.

First create a password item for mounting encrypted folders. In the menu, select "File -> New Password Item". Give the item a name (like "EncFS"), set the "Account Name" to your user, and provide a password. Now test it:

```
$ security find-generic-password -gs EncFS -w
```

If this is the first time you use the `security` command you will get a popup asking for permission to access the keychain. Select "Always allow" to be able to run the command in a non-interactive session.

If you are lucky you already have a password item for the server with the network shares. Search the item list for a network password with the name or IP of the server. If you found one, double-click it to see the details. Check that account, address and protocol are correct, and take note of them. If you can't find a matching network password you can create one manually, or simply connect to the share in the Finder and choose to remember credentials in the connect dialog.


## Mount scripts

Next create a script for mounting the SMB shares.

```
$ touch mount_smb.exp
$ chmod a+x mount_smb.exp
```

Why the `exp` extension? I found that `mount_smbfs` has issues with many passwords when they are supplied as part of the host address, so this script will use `expect` which uses the Tcl scripting language to automatically provide user input. Open the file in your favorite text editor and paste in and modify this example script:

```tcl
#!/usr/bin/expect -f

set smb_server "server"
set smb_user "user"
set smb_password [exec security find-internet-password -a $smb_user -s $smb_server -r "smb " -w]
set smb_path "~/Volumes/$smb_server"

puts "Mounting volumes on $smb_user@$smb_server to $smb_path"

set smb_volumes {"share1" "share2"}
foreach vol $smb_volumes {
    set server_path "//$smb_user@$smb_server/$vol"
    set mount_path "$smb_path/$vol"
    set prompt "Password for $smb_server:"

    spawn mount -t smbfs "$server_path" "$mount_path"
    expect {
        "$prompt" {
            send "$smb_password\n"
        } eof {
            puts "Mounted $server_path on $mount_path"
        } timeout {
            puts "Failed to mount $server_path on $mount_path"
        }
    }
}
```

Next do the same for a `bash` script called `mount_encfs.sh` to mount the encrypted folders:

```
$ touch mount_encfs.sh
$ chmod a+x mount_encfs.sh
```

```bash
#!/bin/bash
# Remember to mount SMB volumes before running this script!

ENCFS="/usr/local/bin/encfs"
ENCFS_PASS=$(security find-generic-password -gs EncFS -w)

# Reverse encrypted dirs
SERVER="server"
ENC_DIRS="share1 share2"
for folder in $ENC_DIRS; do
    src_path="$HOME/Volumes/$SERVER/$folder"
    enc_path="$src_path-encfs"
    echo -n "$ENCFS_PASS" | \
    "$ENCFS" -S -o volname="$folder-encfs" --reverse "$src_path" "$enc_path"
done
```


## Mount on login

You can use Automator to create a workflow that mounts everyting on login. Open Automator, choose "File -> New" in the menu and select "Application". Select the "Utilities" category and drag "Run Shell Script" into the workflow. Replace the command contents with the path to the script that mounts Samba shares. Next drag another "Run Shell Script" into the workflow below the first one, and enter the path to the script that mounts the reverse encrypted folders. To test the workflow, unmount the shares and folders that are already mounted and hit "Run".

Save the workflow, then open "System Preferences -> Users and Groups". Select your user and navigate to "Login Items". Press the plus button to add the workflow app you just saved, and check the "Hide" checkbox (you may have to unlock the panel to make changes).

As a final test reboot the Mac. All shares and encrypted folders should be mounted when you login, and are ready to be securely backed up wherever you want.

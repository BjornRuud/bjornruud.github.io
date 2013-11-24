---
layout: post
title: Encrypt Directories With EncFS on Mac OS X 10.8
tags:
- encfs
- encryption
- mac
- osx
- security
- shell
status: publish
type: post
published: true
excerpt: Mac OS X comes with FileVault, so why use EncFS instead? Well, FileVault has a few drawbacks. Personally I also like that with EncFS the encrypted files are stored in the filesystem as normal, you get the ability to use different encryption on different parts of the filesystem, and backup is straightforward.
---
Note: With OS X 10.8 my [original post](/2011/02/encrypt-directories-with-encfs-on-mac-os-x.html) about using encfs has become slightly dated. This is a revised version of that post.

Mac OS X comes with FileVault, so why use EncFS instead? Well, FileVault has a few drawbacks that are [summed up here](http://techieblurbs.blogspot.com/2010/02/howto-replace-filevault-with-encfs.html). Personally I also like that with EncFS the encrypted files are stored in the filesystem as normal, you get the ability to use different encryption on different parts of the filesystem, and backup is straightforward.


## Requirements

*Update 2013-11-24:*  
The installation method described in this post doesn't work with OS X 10.9 and Xcode 5. Fortunately an easy solution is to install everything with [Homebrew](http://brew.sh/) as described in [this blog post](http://www.mulle-kybernetik.com/weblog/2013/11/osx-109-brew-osxfuse-encfs-mac.html):

```
brew install osxfuse
brew install https://gist.github.com/defunctzombie/7324625/raw/8c791a397767b7f9e7f725e49a45151885e3c1cb/encfs.rb
```

*Previous method:*  
[EncFS](http://www.arg0.net/encfs) requires FUSE so download and install [OSXFuse](http://osxfuse.github.io/) (MacFuse successor). Now we must get EncFS itself, but finding a build compatible with OSXFuse isn't easy. Using [Homebrew](http://mxcl.github.io/homebrew/) would be preferable, but the Homebrew encfs package relies on fuse4x which is the predecessor of OSXFuse. Thankfully Patrick Stein has created a Homebrew formula, [encfsmacosxfuse.rb](https://github.com/jollyjinx/encfs.macosx), that compiles encfs against OSXFuse.

```
brew install https://github.com/jollyjinx/encfs.macosx/raw/master/encfsmacosxfuse.rb
```


## Goal

The goal is to mount encrypted directories on login and unmount them on logout. The whole process should be completely transparent to the user and there are several ways to achieve this. One of them is to use login/logout hooks as documented [here](http://techieblurbs.blogspot.com/2010/02/howto-replace-filevault-with-encfs.html).

That is a nice solution, but in order for the keychain password to be accessible when the hook processes execute it must be placed in a public keychain. The reason is that the user's private keychain isn't unlocked yet at that point. I wanted a solution where the password was administered by the user and stored in his private keychain. I also wanted any scripts to be stored and executed in userspace.


## Solution

First we create the encrypted directory. Make a directory where the encrypted files will be stored, and one which will be the mount point in which the decrypted versions of the files will appear. Then run the `encfs` command and follow the instructions.

```
cd
mkdir -p .encrypted/Vault
mkdir -p Documents/Vault
encfs ~/.encrypted/Vault ~/Documents/Vault
... follow instructions ...
... check that encryption works ...
umount Documents/Vault
```

To solve the login/logout problem I created a script which handles both. It executes upon login and mounts the encrypted directories with a password from the user's keychain. The script then goes to sleep. When the user logs out the sleeping script gets a SIGTERM (15) signal which is intercepted. The script then unmounts the encrypted directories, performs cleanup, and exits.

Here is the script which I put in `~/bin/encfsd.sh`:

```bash
#!/bin/bash

ENCFS="/path/to/encfs"
ENCDIR="$HOME/.encrypted/Vault"
DECDIR="$HOME/Documents/Vault"

function cleanup {
    # Kill sleep command ($! is PID of last command launched in background)
    kill $!
    umount "$DECDIR"
    exit
}
trap cleanup 1 2 3 6 15

security find-generic-password -gs EncFS 2>&1 >/dev/null | cut -d'"' -f2 | "$ENCFS" -S "$ENCDIR" "$DECDIR"

# Wait for exit
while true; do
    # Sleeping ignores normal signals so start it in a subprocess and wait for it
    sleep 3600 &
    wait
done
```

The cleanup function is called when the script gets signalled to terminate. It kills the sleep process running in the background, unmounts the encrypted directory, and exits.

After the function body is the trap command which tells the script to call the cleanup function when it receives signal 1, 2, 3, 6 or 15. The last signal, 15, is the default quit signal SIGTERM (check `man signal` for the rest).

Next comes the `security` command which retrieves the password from the user's keychain. The redirect is necessary since the password is printed to stderr. The output with the password is then sent to the cut command where the password is isolated. It is then sent as input to the `encfs` command.

After that the script goes to sleep.

To automatically start the script on login it is installed as a Launch Agent. Here are the contents of my `~/Library/LaunchAgents/localhost.encfsd.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>localhost.encfsd</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/boru/bin/encfsd.sh</string>
  </array>
  <key>KeepAlive</key>
  <true/>
  <key>RunAtLoad</key>
  <true/>
  </dict>
</plist>
```

It basically says the system should run the `encfsd.sh` script once and run when the agent loads.

With the scripts in place the user should now have automatic mount of encrypted directories on login, and they should unmount on logout.

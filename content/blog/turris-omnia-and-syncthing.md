---
title: "Turris Omnia and syncthing"
date: 2023-02-25T20:12:12+01:00
image: "blog/turris-omnia-and-syncthing/syncthing-logo.png"
draft: false
---

The [Turris Omnia](https://www.turris.com/en/omnia/overview/) router is an
unusual device. Manufacturers tend to abandon their devices after a while, forcing their customers to buy shiny new hardware. The good souls behind the Omnia, the
Czech domain registry [CZ NIC](https://www.nic.cz/), have a different strategy.
They are constantly improving and enhancing their offering to existing customers.

# Syncthing integration

The [https://syncthing.net/](https://syncthing.net/) is an open source file
synchronization tool. Simply speaking. It can transfer files from A to B, or
from B to A, or to mirror changes between A and B. And many more! It comes with a web user interface and a dozens of clients for almost every platform.

Turris OS 6.2.0
[added](https://forum.turris.cz/t/turris-os-6-2-0-is-released/18483) a
Syncthing integration, which can be enabled from reForis, the default admin interface.

Once enabled, a new icon will appear on the router's main page
![Enabled Syncthing admin panel](enabled-syncthing-app.png)

Unfortunately the app expects an another login, which is a bit annoying a Turris OS. Every app needs it's own login credentials.

![Syncthing app asks for username and a password](user-and-password-dialog.png)

The first login screen will then appear. Syncthing politely asks for
telemetry, which I have enabled. This can be changed at any time in the settings.

![Syncthing asks for a telemetry to be enabled](opt-in-telemetry.png)

Surprisingly, there are two big warnings appearing on the main page. These leave a bad taste in the mouth of a first time user like me. One is about the need to create yet another user/password and the other one informs about the inability to create `/Sync` directory. Both problems have been reported to the Turris developers. So maybe they will be fixed soon.

![Syncthing app showing two warnings about username and password and permission denied for /Sync](first-login.png)

# Filesystem on Turris

The Turris is an embedded system. Unlike a typical computer, the main system storage is not suitable for writing, as it is slow, has a tiny capacity and won't survive many writes like your SSD or a spinning rust drive. If you want to use it as a NAS, you MUST connect an external hard drive. This will be formatted to [btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) and mounted as `/srv`.

The text above means that it does not make a sense to have a default folder root directory under `/`. This points to a slow flash storage. I even had a hard time figuring out where to change the settings. _Hint it is behind *Advanced* settings with a nice red warning_. So I decided to fix it from the command line. After logging into the system  using `ssh`, I noticed that there was a `/srv/syncthing` directory. And it had some data for the syncthing itself. I was not sure it if it is wise to synchronise to the _same_ place, so I manually created a parallel directory structure for a synchronised folder.

```sh
# mkdir -p /srv/Sync/default
# chown -R syncthing:syncthing /srv/Sync
```

I was not entirely honest there. _Of course_ I made a `/srv/Sync` and set it as the default folder first. Worse! I made the second one as `/srv/Sync/Kamera` ignoring all the warnings. Fortunately it was trivial to fix the setup and move the default location to subdirectory of `/srv/Sync`.

Don't try this at home, kids!

![Loud warnings of a wrong path setup](path-mistake.png)

# Usage

Using Syncthing is so simple, that it's probably not worth explaining. It has a concept of devices. Turris acts as a _primary device_, so in order to synchronise files from an another device, the client application must be installed. There is an application for Android. Once installed - the new devices will be discovered and displayed in the web user interface. All that was needed was to grant the permissions on Turris and on the Android side.

I wanted to sync photos and videos, so had selected `/storage/emulated/0/DCIM` on Adroid and setup the folder type properly:

 * send only on Android
 * receive only on Turris

So the synchronisation will be one way. Deleting a file from phone won't
have any impact on already synchronised folder on the Turris side.

![Synchronization from Android phone to Turris is in progress](sync-in-progress.png)

And that's all folks! The Turris integration have a few rough edges. However the
end result and a software itself is fantastic. Visit
[https://docs.syncthing.net/](https://docs.syncthing.net/) for more details and
features it offers.

Used [Deepl](https://www.deepl.com/write) as an co-editor for this article. All mistakes and errors are mine.

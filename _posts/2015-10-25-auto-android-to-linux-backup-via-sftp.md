---
layout: post
title: Blogging Like a Hacker
---

# Notes from mounting Android folder over wifi and making backups to Linux machine automatic [2015-10-25]

Author: Grzegorz Wierzowiecki

### Writing style remarks

Hi! As I was not writing such tutorial to general audience, I had trouble in balancing tradeoff about assumptions what's obvious, what's not to potential reader. I hope you will enjoy, I am happy to receive suggestions, might be in form of git patches against this file (as it's hosted on github).

### Intro:

Surprisingly, it took more time to find right Android app to mount photos directory of Android device over wifi (not usb cable) as Linux directory.

As I spend a bit of time trying few solutions, this inspired me to share.

Here, I share what I found most robust and flexible to me.
Additionally I extend with a bit of comments that might make those notes helpful also for Linux+Android beginners.

Hopefully this note will let you enjoy your Android files after few minutes :).


### Final goal:

My initial motivations:

* **automatically** my computers fetch files from my phone via wifi, when I reach home wifi 
* I have possibility to trigger automation when I am with my phone on good network (e.g. travelling) (# *Not yet fully implemented but possible via DDNS*)

### Requirements and preferences:

* **no ssh shell** server on Android
    * only read-only access, only to selected folders (e.g. photos folder)
    * encrypted protocol - this rejects hell lot of Android apps, like unencrypted "FTP" servers etc.
* preferably public key authorization
* automatic syncing (read as: scriptable)

### Means / Implemented with:

Combination that worked for me:

* SFTP protocol
* 8192 bit RSA Public:Private Keys for authorization
* Linux:
    * [sshfs] for mounting (FYI: full shell is not required for it)
    * [rsync] for syncing directories
* Android:
    * [SSH Server Pro] (also possible with [free version][SSH Server]) by [Ice Cold Apps]
* [DDNS] for travelling, i.e. other wifi networks. [DDNS] for automatic Android announcing it's IP and for computers to easily reach it (*Please note: it's "open option", i.e. not yet fully configured and tested, but available in [SSH Server] with plenty of [DDNS] services to choose*)

## How To

### Linux - Cryptographic Key-Pair generation:

```bash
$ ssh-keygen -t rsa -b 8192 -f ~/.ssh/id_rsa_nopass
# * empty password: as I want to use it for automatic passwordless authentication I leave password empty
# * type RSA: as SSH Server on Android supports only RSA and DSA	
# Now, add key to ssh-agent so it can be used by ssh and sshfs
$ eval $(ssh-agent)
$ ssh-add ~/.ssh/id_rsa_nopass
# this generates both:
#  * private key: id_rsa_nopass
#  * public key: id_rsa_nopass.pub
# please keep your private key in secret only on your machine and do not transmit
# public key is the only key for public (other people) announcement
```

### Android - Setting up SFTP server:

If section looks lengthy - You can just try do it or follow [screenshots][ASrvScreenshots]! 

If you are not scared of lengthy description, let's go over all options I set, step by step:

* *sftp server*: I used [SSH Server Pro] (or [free][SSH Server] version) by [Ice Cold Apps](http://www.icecoldapps.com/) (but can be other app)
* *public key for authorisation*:
    * downloaded `id_rsa_nopass.pub` (see previous section) file (via whatever channel, even email attachment, it's *public key*)
    * it landed in Download which was placed in `/storage/emulated/0/Download`
    * I created new folder with [SolidExplorer] `/storage/emulated/0/MyFiles`
    * copied there `id_rsa_nopass.pub`
* Setup with [SSH Server Pro] (or [free][SSH Server] version) - [Please click for **screenshots**][ASrvScreenshots] for configuration steps described below.
    * New server SSH Server (don't worry, we will disable Shell) ([Screenshot][SS Adding Server])
    * server configuration (screenshot [1^st^][SS C1] and [2^nd^][SS C2])
        * created new server on some nice port like 43210
        * Disabled "Enable Shell" -> for higher security. I just need files
        * left "Enable SFTP (FTP over SSH)" (leaving SCP Enabled should not harm)
        * (optional step) Set "Start server when connected to selected wifi network and stop when disconnected" -> "Get current SSID" or "Get current BSSID"
  * Adding Users section (screenshots [1^st^][SS U0], [2^nd^][SS U1], [3^rd^][SS U2])
    * Add new user
    * with password of course - long and almost random as I will use public Key anyway
    * and "Enable public key"
    * [Browse] -> "/storage/emulated/0/MyFiles/id_rsa_nopass"
    * Document root
    * /storage/emulated/legacy/DCIM # Here I found photos on Android 5.x
    * selected option "Force stay in document root"
    * Disabled write access -> I want to copy files to NAS only, not modify or remove
    * disallowed all "forwardings"
* Save all settings when coming back to main menu and start server!
* Get [IP][IP Address] of your Android device!
    * local IP: for searching IPs in local network I use [Fing]
    * public IP: you can play with [DDNS] functionality available in [SSH Server] (see [this screenshot][SS Adding Server] of adding new servers, look for *"Dynamic DNS Updater"*)

### Linux - Finding [IP address] of device

#### Scanning local network with *nmap*

My approach is following. As I know that all my devices will run [SFTP] server on some defined and relatively unique to my environment port (let's use 43210 as example), I decided to scan all internal [IPs][IP address] with [nmap]:

```bash
# Assuming your network is 192.168.0.0/24 (ifconfig is your friend)
nmap 192.168.0.0/24 -p 43210 --max-retries 1 --open 
```

To automate this in my scripts I iterate in a loop over all devices with  defined port open:

```bash
for ip in $(nmap 192.168.0.0/24 -p 43210 --max-retries 1 --open | grep -o '\([0-9]\{1,3\}\.\)\{3\}[0-9]\{1,3\}'); do
  # magic explained below...
done
```

#### General network -> Use Dynamic DNS

I've chosen [SSH Server] partly due to it's ability to register IP address via [DDNS].  However this option I did not yet explored it with this application or written results down, so I encourage, dear readers, to give it a try! (Or maybe even suggest patch to this article as it is github!)

### Linux - mounting ssh server despite deprecated *ssh-dss* key type...

And does not mount:


>read: Connection reset by peer

Just mounting with `-d` debug mode results in :

>nullpath_ok: 0
>nopath: 0
>utime_omit_ok: 0
>Unable to negotiate with 192.168.0.123: no matching host key type found. Their offer: ssh-dss
>read: Connection reset by peer

And now, we know what has happened. As we know from [OpenSSH 7.0 announcement] `ssh-dss` keys are deprecated since then and we need to follow [OpenSSH legacy instructions] to use it, to **sum it up**: to bypass it **we need** to pass **`-oHostKeyAlgorithms=+ssh-dss` flag to `ssh`**. (*Yes, I know it sucks, but I really didn't find better SFTP server for android offering more up-to-date crypto and restriction to just read-only ftp (i.e. "no shell"), if you find sth better, please let me know. For now, I consider this still ok for my use case, so let's continue*).

Therefore, let's run `sshfs` with following assumptions:

* `192.168.0.123` is Android's [IP address] (you can check with [Find] or `nmap`)
*  `AwesomeUsername` is username set in Android's SFTP server
* `/storage/emulated/legacy/DCIM` is path server is serving
* `/path_we_mount` - place where you want to mount your Android device. It's can be any. Usually place is under `/mnt`, but it can be temporary `/tmp/mytemporarymount` or in home `/home/mylinuxusername/androidmount`
* `43210` - chosen port in Android's SFTP server
* `-o uid=$(id -u),gid=$(id -g)` - that you want mounted files to appear owned by your user:group . If you want other, simply specify `-o uid=$(id -u username),gid=$(id -g groupname)`

```bash
$ sshfs AwesomeUsername@192.168.0.123:/storage/emulated/legacy/DCIM /path_we_mount -p 43210 -o uid=$(id -u),gid=$(id -g) -o HostKeyAlgorithms=+ssh-dss
```

<!-- TODO: as files have only read permission, check how does rsync behave in case of their change... -->

Double check contents of directory!

```bash
$ ls -lha /path_we_mount /Camera | tail -n 4
-r-------- 1 gwpl users 153M Oct 20 20:22 VID_20151020_202055.mp4
-r-------- 1 gwpl users 346M Oct 22 23:27 VID_20151022_232428.mp4
-r-------- 1 gwpl users 205M Oct 22 23:29 VID_20151022_232735.mp4
-r-------- 1 gwpl users  52M Oct 23 11:57 VID_20151023_115641.mp4
```

### Making backup with *rsync*

Now you can run rsync in dry-run (it means, it will not make any changes, just write what would do):

```bash
$ rsync --dry-run --recursive --times --progress \
  --exclude '*.thumbnails*' \
  --exclude '*Camera/.aux/.nomedia' \
  --exclude '*Camera/.aux/*' \
  --exclude '*Camera/*.mp4.tmp' \
  --exclude '*Camera/thumbnails/*' \
   /path_we_mount \
   /path_where_to_copy
```

I encourage to learn about more flags in [man rsync] (my usual set if `-aAXv`), here short survival summary for photo backups:

* `--dry-run`(or `-n`) makes *rsync* to not do any operations, just write what it would do
* `--recursive` (or `-r`) let *rsync* travel into directories.
* `--times` (or `-t`) make *rsync* to preserve original file modification time (it's nice to have time when photo was made not only inside *exif* section of *jpg* file :) )
* `--progress` show progress during transfer
* `--exclude` allows to ignore some files. Above set is my favourite when making backups of Android *"Camera"* folder.
* `--bwlimit=4096KiB` - you can see below I throttle/limit bandwidth to give device and wifi a lot of breadth, but feel free to drop limit.

<!--
Instead of explaining, like I started below my favourite set of options : `-aAXv --progress` and rewriting `man rsync` let's propose simpler set of attributes for simplicity of article.
---------------Started explanation of my favourite set of options `-aAXv --progress`-------------------
I used `-aAXv` of options for following reasons:

* `-a`, `--archive` equals to `-rlptgoD` basically all those options you usually want to use when making archival/backup recursive copy. (In case of making backup of photos most important to me are: recursive `-r`=`--recursive` and `-t`=`--times` (preserve modification times) - so I could go with just those two.
* `-A`, `--acls`
* `-X`,`--xattrs`
* `-v`
---------------------end of draft of explanation-----------------------------
-->

If output of *dry run* looks fine and operations such we want to happen, then remove `--dry-run` flag and run command again, to actually perform operations:

```bash
$ rsync --bwlimit=4096KiB  --dry-run --recursive --times --progress \
  --exclude '*.thumbnails*' \
  --exclude '*Camera/.aux/.nomedia' \
  --exclude '*Camera/.aux/*' \
  --exclude '*Camera/*.mp4.tmp' \
  --exclude '*Camera/thumbnails/*' \
  /path_we_mount \
  /path_where_to_copy
```

(As you see I limited bandwidth, so give phone and wifi more breathing, but you are free to drop limits.)

### Cleanup -> let's umount  device

After everything, let's umount (opposite to earlier mount):

```bash
$ fusermount -u /path_we_mount
```


<!-- TODO: add "if you liked article, please flattr or donate. And make flattr and bitcoin / paypal donate buttons. Plus make note that I donate regularily opensource and charities so will be happy to forward money and motivated to make more of such works. -->

[DDNS]: https://en.wikipedia.org/wiki/Dynamic_DNS
[sshfs]: https://en.wikipedia.org/wiki/SSHFS
[rsync]: https://en.wikipedia.org/wiki/Rsync
[nmap]: https://en.wikipedia.org/wiki/Nmap
[man rsync]: http://linux.die.net/man/1/rsync
[IP address]: https://en.wikipedia.org/wiki/IP_address
[SSH Server]: https://play.google.com/store/apps/details?id=com.icecoldapps.sshserver
[SSH Server Pro]: https://play.google.com/store/apps/details?id=com.icecoldapps.sshserverpro
[Ice Cold Apps]: http://www.icecoldapps.com/
[Solid Explorer]: https://play.google.com/store/apps/details?id=pl.solidexplorer2
[Fing]: https://play.google.com/store/apps/details?id=com.overlook.android.fing
[OpenSSH 7.0 announcement]: http://lists.mindrot.org/pipermail/openssh-unix-announce/2015-August/000122.html
[OpenSSH legacy instructions]: http://www.openssh.com/legacy.html
[ASrvScreenshots]: https://goo.gl/photos/CFM5PUtK3tRAVmEP9
[ASrvScreenshotsDrive]: https://drive.google.com/folderview?id=0B-v2k2LMbjB6UDcwWlJSSmJmcGM&usp=sharing
[SS Adding Server]: https://photos.google.com/share/AF1QipPNt0r7RTIZW4qzmYeU-57CQqiGpsHhZUQTtR4tjIH_RzjTZqxeWTfrQ1_LSOXRDQ/photo/AF1QipPtv4DWVMuEAbbemvuJPb9WyH4rs6x-HBuX72l_?key=QXJyTG5zNVQ2ak4zdDd3Q0xYX1pVdkd3eW5zUEdB
[SS C1]: https://photos.google.com/share/AF1QipPNt0r7RTIZW4qzmYeU-57CQqiGpsHhZUQTtR4tjIH_RzjTZqxeWTfrQ1_LSOXRDQ/photo/AF1QipNXuorQNfG00-19gnKI1iiBCfpUZIzxKzrtB02k?key=QXJyTG5zNVQ2ak4zdDd3Q0xYX1pVdkd3eW5zUEdB
[SS C2]: https://photos.google.com/share/AF1QipPNt0r7RTIZW4qzmYeU-57CQqiGpsHhZUQTtR4tjIH_RzjTZqxeWTfrQ1_LSOXRDQ/photo/AF1QipPzDAYJXtlu1Fmaz_nOoJT_RH9UFQVHVEjRE4IV?key=QXJyTG5zNVQ2ak4zdDd3Q0xYX1pVdkd3eW5zUEdB
[SS U0]: https://photos.google.com/share/AF1QipPNt0r7RTIZW4qzmYeU-57CQqiGpsHhZUQTtR4tjIH_RzjTZqxeWTfrQ1_LSOXRDQ/photo/AF1QipNrqHBzr6WQcXNlJ0ld-pgdz591FVbUgV_4usGk?key=QXJyTG5zNVQ2ak4zdDd3Q0xYX1pVdkd3eW5zUEdB
[SS U1]: https://photos.google.com/share/AF1QipPNt0r7RTIZW4qzmYeU-57CQqiGpsHhZUQTtR4tjIH_RzjTZqxeWTfrQ1_LSOXRDQ/photo/AF1QipN28DuFBr_VMakrDqdIFpEWTOMcCQoA3-p0uQSB?key=QXJyTG5zNVQ2ak4zdDd3Q0xYX1pVdkd3eW5zUEdB
[SS U2]: https://photos.google.com/share/AF1QipPNt0r7RTIZW4qzmYeU-57CQqiGpsHhZUQTtR4tjIH_RzjTZqxeWTfrQ1_LSOXRDQ/photo/AF1QipNkmmkI7oThIWXIQf32XRmfn1TbzvWbsNispUHo?key=QXJyTG5zNVQ2ak4zdDd3Q0xYX1pVdkd3eW5zUEdB
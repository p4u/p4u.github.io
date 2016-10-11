---
layout: post
title: Luks disk encryption with SSH decryption and Telegram alerts
published: true
---

If you have a server somewhere in the Internet, IMO, it is important to have the full data encrypted.
Specially if it is hosted on some third party location.
If you do not have access to the HOST machine (for instance you are running a virtual private server), you can never be 100%
sure of the privacy of your data. Even if the virtual disk is encrypted with LUKS, in runtime the decryption key is
placed somewhere in the RAM. An addvanced attacker would (in theory) be able to scan the RAM for key-like strings.


However I always prefer to encrypt the information, so it is not that easy for an attacker to browse into my private files.


Usually the VPS providers does not give you the option to encrypt the root partition, however it can be achieved by hacking a bit.
I use Hetzner, they have a very powerful and flexible administration interface where you can boot in *rescue mode* to 
do whatever you want to do to your virtual server. So I installed a plain Debian Linux to afterwars encrypt the root partition.


1. Mount /dev/sda1 to /mnt
2. Copy all data to your local computer: 
  * `rsync -vagHl root@your.server:/mnt/ localfolder/`
3. Fdisk /dev/sda an create partitions (sda1=BOOT, sda2=LVM)
4. Follow [cryptosetup instructions](https://debian-administration.org/article/469/How_to_set_up_an_encrypted_filesystem_in_several_easy_steps)
5. Copy back the OS data: 
  * `rsync -vagHl localfolder/ root@your.server:/mnt/`
6. Set up [SSH unlock](http://blog.neutrino.es/es/2011/unlocking-a-luks-encrypted-root-partition-remotely-via-ssh/) to decrypt the filesystem using SSH

To this point, when the server boots, the initram (which includes busybox and dropbear) is ready to handle a SSH login.
By writing the passphrase to */lib/cryptsetup/passfifo* we will decrypt the partition and the boot process will follow.

However if the server reboots randomly in some moment, we might not notice and it would supose a not acceptable server downtime.
Sending an e-mail would be a good option, but I like even more to use Telegram because my Android phone is always next to me.
So let's see how to achieve this. We have to modify the initram to send a Telegram when the system is waiting to decrypt the disk.

#### Curl + SSL + Telegram = InitRam

This is a script which uses "curl" to send Telegram messages (we will call this script telegram-alert).
To create an api token you need to open a Telegram chat to Botfather and create a new Bot. Then you must start a conversation with it (search for the name you gave to him).
To get your own chat ID you can use this [Bot](https://telegram.me/get_id_bot).

```bash
#!/bin/sh
#message=$( cat )
message="Server foo.bar needs to decrypt the key"
apiToken=<your_botfather_api_token>
userChatId=<your_user_chat_id>

sendTelegram() {
        curl -s \
        -X POST \
        https://api.telegram.org/bot$apiToken/sendMessage \
        -d text="$message" \
        -d chat_id=$userChatId
}

if  [[ -z "$message" ]]; then
        echo "Please pipe a message to me!"
else
        sendTelegram
fi
```

To use this script we need CURL with SSL support. It might look simple but it is not, the initram is a very tiny system which does not include SSL libraries nor CURL.

So after trying to manual compile a stupid static binary of curl with SSL and ZLIB support, I found [this project](https://github.com/odise/busybox-curl) which I like much more.
It is a busybox installation (root filesystem) which includes everything we need (the only modification I had to make to this root filesystem is to write a valid /etc/resolv.conf file). 

Here you can download the one I'm using: [busycurl.tar.gz](https://github.com/p4u/p4u.github.io/raw/master/files/busycurl.tar.gz).

So the strategy will be as follows:

1. Include the **busybox-curl root filesystem** in tar.gz format to the standard initram image
2. Include the **telegram_alert script** to the initram image
3. Include an **initram script** which will be executed automatically to:
  * Uncompress the busybox-curl filessystem.
  * Chroot to it
  * Execute the telegram script
 
So in our VPS server we will copy four new files (find all of them attached to this post):

* The root filesystem containing busybox, CURL and SSL 
  * */etc/initramfs-tools/root/busycurl.tar.gz*
* The script which sends the telegram alert using CURL
  * */etc/initramfs-tools/root/telegram_alert*
* The script executed by the initram every 60 seconds to send the alarm:
  * */etc/initramfs-tools/scripts/init-top/send_decrypt_alert*
* The hook telegram script is executed by initramfs-tools to include the required files into the resulting initram:
  * */etc/initramfs-tools/hooks/telegram*

Once you copied these four files you must run:

`update-initramfs -u -v`

And finally reboot your server to check that it works. If something wrong you can SSH login to the initram to debug the problem:

`ssh -o "UserKnownHostsFile=~/.ssh/known_hosts.initramfs" your.vps.server.com -i .ssh/id_rsa_nil`

I've included an ssmtp binary copy to the busybox filesystem. Would be nice to, in addition to telegram, send e-mail alert using this tiny MTA agent.
If you manage to do it, send me the info so I'll extend this article.

### /etc/initramfs-tools/scripts/init-top/send_decrypt_alert

```bash
#!/bin/sh

# This part is to be compatible with initramfs-tools

PREREQ=""
prereqs()
{
  echo "$PREREQ"
}
case $1 in
# get pre-requisites
prereqs)
  prereqs
  exit 0
  ;;
esac

# Here starts the execution in initram time

send_alarm()
{
  n=0
  while [ $n -lt 10 ]; do
  chroot $DEST /bin/telegram_alert
  [ $? -eq 0 ] && exit 0
  sleep 30
  n=$(($n+1))
  done
  exit 0
}

DEST="/busycurl"
mkdir $DEST
tar xzf /root/busycurl.tar.gz -C $DEST/
cp -f /root/telegram_alert $DEST/bin/telegram_alert
chmod +x $DEST/bin/telegram_alert

send_alarm &
```

### /etc/initramfs-tools/hooks/telegram

```bash
#!/bin/sh

PREREQ=""

prereqs() {
  echo "$PREREQ"
}

case "$1" in
  prereqs)
    prereqs
    exit 0
  ;;
esac

. "${CONFDIR}/initramfs.conf"
. /usr/share/initramfs-tools/hook-functions

copy_exec /etc/initramfs-tools/root/busycurl.tar.gz /root/busycurl.tar.gz
copy_exec /etc/initramfs-tools/root/telegram_alert /root/telegram_alert
```

### /etc/initramfs-tools/root/telegram_alert

```bash
#!/bin/sh
#message=$( cat )
message="Server foo.bar needs to decrypt the key"
apiToken=<your_botfather_api_token>
userChatId=<your_user_chat_id>

sendTelegram() {
        curl -s \
        -X POST \
        https://api.telegram.org/bot$apiToken/sendMessage \
        -d text="$message" \
        -d chat_id=$userChatId
}

if  [[ -z "$message" ]]; then
        echo "Please pipe a message to me!"
else
        sendTelegram
fi
```


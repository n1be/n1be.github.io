<!-- vim: autoindent expandtab ruler tabstop=4 sw=4 sts=4 laststatus=2
-->

Headless Raspberry Pi setup (Raspbian Stretch) from a Linux PC:
===============================================================

This page describes the initial setup, installation and configuration
of software on a Raspberry Pi for "embedded" use.  Since there is no
monitor or keyboard, the Pi will be managed remotely via the network.
Also, Raspbian Lite is installed to save space on the SD card and to
free up system resources.

These links take you to the various setup activities.

1. [Generate an SSH key pair](#SSHClientSetup)
1. [Download Raspbian to an SDHC card](#SDHCSetup)
1. [Configure Raspbian](#RaspiConfig)

<hr /><a name="SSHClientSetup"></a>
Generate an SSH key pair for your PC
------------------

Generate an ssh key so that remote access to the Raspberry Pi will
be limited to your PC.  The key is only generated if one does
not already exist.  For more info, see
[HPR Episode 2356](http://hackerpublicradio.org/eps.php?id=2356)
.  This command will generate the key:

            $ [ -e ~/.ssh/id_ed25519.pub ] || ssh-keygen -t ed25519 -C "Raspberry Pi key"

<hr /><a name="SDHCSetup"></a>
Create a bootable SDHC (or uSDHC) card that will allow remote access
------------------

These steps initialize an SDHC card with the Raspbian OS so that when inserted
into a Raspberry Pi the Pi will be on a wireless network and remote
administration is possible.

* Download a zipped Raspbian Lite image from the Raspberry Pi website 
[Downloads page](https://www.raspberrypi.org/downloads/) to your PC.
In this example the file was called __2017-08-16-raspbian-stretch-lite.zip__

* Also download the __SHA256SUM__ file and use it to verify the downloaded zip.
Here grep is used so only one file is verified although there are multiple files
listed in the SHA256SUM file:

            $ grep lite SHA256SUM | sha256sum -c -
            2017-08-16-raspbian-stretch-lite.zip: OK

* Insert a spare SDHC card into your PC.  __Beware, the entire card will
be overwritten.  Do not use a card containing important data.__

* Use the __lsblk__ utility to determine the device name of the SDHC card.  In
this example the device is __/dev/mmcblk0__ and the only partition is
__/dev/mmcblk0p1__.  (The SDHC card is an `mmcblk` device because it is inserted
into an SDHC slot on the PC.  It might also appear as a SCSI disk with a name
like `sdc`.)  __Note which device names are being used for the SDHC card on your
PC; in commands below you need to use the device names for your SDHC card.__

            $ lsblk
            NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
            sda           8:0    0 931.5G  0 disk 
            +-sda1        8:1    0 931.5G  0 part /Data
            sdb           8:16   0 111.8G  0 disk 
            +-sdb1        8:17   0     1M  0 part 
            +-sdb2        8:18   0    20G  0 part /Alt
            +-sdb3        8:19   0    20G  0 part /
            +-sdb4        8:20   0     4G  0 part [SWAP]
            +-sdb5        8:21   0     5G  0 part /tmp
            +-sdb6        8:22   0  62.8G  0 part /home
            mmcblk0     179:0    0  14.5G  0 disk 
            +-mmcblk0p1 179:1    0  14.5G  0 part /media/rob/8B12-39CA

* If any file systems on the SDHC card are mounted, they must be unmounted.  The
listing above gives the mountpoint for any mounted file systems on the SDHC
card.  This command unmounts it.

            $ umount /media/rob/8B12-39CA

* Extract the image and (as the superuser) copy it to the SDHC card.
The dd command uses the disk device of the SDHC card that was seen above in the
output from `lsblk`.

            $ zcat 2017-08-16-raspbian-stretch-lite.zip | sudo dd of=/dev/mmcblk0 bs=1M conv=fsync
            [sudo] password for rob:
            0+56591 records in
            0+56591 records out
            1854418944 bytes (1.9 GB, 1.7 GiB) copied, 157.615 s, 11.8 MB/s

* Remove the SDHC card from the PC.  Wait a few seconds and re-insert the the SDHC
card.  Use `lsblk` to discover if the file systems on the SDHC card were
automatically mounted.  (Both partitions automatically mounted on my Ubuntu PC.)

            $ lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT | grep mmc
            mmcblk0             14.5G 
            +-mmcblk0p1 vfat    41.8M /media/rob/boot
            +-mmcblk0p2 ext4     1.7G /media/rob/28590797-4810-4851-b4ec-bf9672c2918c

* The larger partition on the SDHC card contains the ext4 root file system.  The
smaller partition contains the vfat /boot file system with the kernel and
configuration information.

    In case the ext4 file system was not automatically mounted, then as the
superuser manually mount it.  A command like this can be used:

            $ sudo mount /dev/mmcblk0p2 /mnt

* Become the superuser and change directory to to where the root file system
is mounted.

            $ sudo -i
            # cd /media/rob/28590797-4810-4851-b4ec-bf9672c2918c

* In case the vfat file system is not mounted, use a command like the following
to mount it.

            # mount /dev/mmcblk0p2 boot

* Enable the ssh server to start when the Raspberry Pi boots by creating a file called `ssh` in the directory where the /boot file system is mounted.
[Remote-Access SSH](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md)
has more details.  Then unmount the /boot file system.  These are the commands used
with the boot filesystem automatically mounted upon SDHC insertion:

            # touch /media/rob/boot/ssh
            # umount /media/rob/boot

* Configure the ssh server so that access is limited to using key-based
authentication as explained in
[Securing your Raspberry Pi](https://www.raspberrypi.org/documentation/configuration/security.md):

            # sed -i etc/ssh/sshd_config \
            > -e '/^[#[:space:]]*ChallengeResponseAuthentication /s/.*/ChallengeResponseAuthentication no/' \
            > -e '/^[#[:space:]]*PasswordAuthentication /s/.*/PasswordAuthentication no/' \
            > -e '/^[#[:space:]]*PermitRootLogin /s/.*/PermitRootLogin no/' \
            > -e '/^[#[:space:]]*UsePAM /s/.*/UsePAM no/'

* Configure user `pi` to allow ssh login via your public key from the PC.
Substitute your username on the PC for `rob` when executing the second command:

            # mkdir home/pi/.ssh
            # cat ~rob/.ssh/id_ed25519.pub >> home/pi/.ssh/authorized_keys
            # chown 1000:1000 home/pi/.ssh home/pi/.ssh/authorized_keys
            # chmod 700 home/pi/.ssh
            # chmod 600 home/pi/.ssh/authorized_keys

* Configure the WiFi settings so the Raspberry Pi will automatically connect to
your network.  Here the country code is set to __US__.  Then a configuration is added for a network with SSID __testing__ and password __testingPassword__.
For more detail, see:
[Setting WiFi up via the command line](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)

            # pushd etc/wpa_supplicant
            # sed -e '/^country=/s/=.*/=US/' -i wpa_supplicant.conf
            # wpa_passphrase "testing" "testingPassword" >> wpa_supplicant.conf
            # popd

* Optionally, set the hostname used by the Raspberry Pi.  Here it is set to
__AudioPi__ making it easier to find on the network.

            # echo 'AudioPi' > etc/hostname

* Unmount the root file system.  Then give up the superuser privilege.

            # cd
            # umount /media/rob/28590797-4810-4851-b4ec-bf9672c2918c
            # exit
            logout
            $

* Remove the SDHC card from the PC.

<hr /><a name="RaspiConfig"></a>
Perform initial Raspbian configuration
------------------

To avoid damage, two guidelines should be followed.

1. Except for USB devices, do not connect or disconnect anything with the
Raspberry Pi when power is applied.

2. If possible, perform a clean shutdown of Raspbian via a command like
`sudo poweroff` before removing power.

These steps boot the Raspberry Pi from the SDHC card and use remote access
to perform initial configuration.


* Insert the SDHC card into the Raspberry Pi.

* If the Raspberry Pi does not have on-board WiFi, insert a USB WiFi dongle.
(Or connect the Raspberry Pi to wired a ethernet.)

* Connect power to the Raspberry Pi.

* Allow 2 minutes for the Pi to boot and get on the network.  Then verify that it is
reachable on the network by entering this command from the PC.  If the hostname was
not changed during the prior steps, it will be `raspberrypi` instead of `audiopi`:

            $ ping -c 5 audiopi

    If connectivity and remote access is not working, disconnect power, then
connect a monitor and USB keyboard to the Raspberry Pi; power up and manually
diagnose.
Alternately, you can insert the SDHC card into your PC and examine the log
files in `var/log` of the root file system.  For example, look in `daemon.log`
or `syslog` to verify that wpa_supplicant could successfully start.

* Use ssh on the PC to login to the Pi via the network:

            $ ssh pi@audiopi
            The authenticity of host 'audiopi (192.168.77.123)' can't be established.
            ECDSA key fingerprint is SHA256:LGg0xwraVMIY3Q2DMAwwRBxhLMvbCbY9Lvr20EWm0bk.
            Are you sure you want to continue connecting (yes/no)? yes
            Warning: Permanently added 'audiopi' (ECDSA) to the list of known hosts.
            /usr/bin/xauth:  file /home/pi/.Xauthority does not exist

* Become the superuser on the Pi.  Then delete the file that allows this
without requiring a password to be entered:

            pi@AudioPi:~ $ sudo -i
            root@AudioPi:~# rm /etc/sudoers.d/010_pi-nopasswd

* Update all software and install ssh related packages.  This step requires
a working Internet connection:

            root@AudioPi:~# apt update && apt upgrade -y
            ...
            root@AudioPi:~# apt install -y molly-guard openssh-blacklist-extra
            ...

* Use the `raspi-config` command to change these settings.  These
settings apply to my use case.  In particular, the Interfacing Options
will depend upon your intended use of the Pi:

            - 1: Change User Password
            - 3: Boot Options
                 - B1: Text console, requiring user to login
                 - B2: No wait during boot until a network connection is established.
            - 4: Localisation Options (Use what applies to your location...)
                 - I1: Locale - disable: en_GB.UTF-8
                              - enable:  en_US.UTF-8
                              - system default: C.UTF-8
                 - I2: Timezone = US Eastern
            - 5 Interfacing Options
                 - P5 I2C Interface:      enabled
                 - P6 Serial Port Login:  enabled
            - 7 Advanced Options:
                 - A3 Memory Split:       8 MB for the GPU
                 - A1 Expand Filesystem:  selected

    The last step in `raspi-config` is to choose __`<Finish>`__.  At this point,
do not allow raspi-config to reboot the Raspberry Pi.  Instead power-off the Pi
so the hardware can be reconfigured if that is required.  Molly-guard requires
verification of which machine to reboot.  (The correct case must be used for the
hostname.)  It may take a very long time before the PC discovers the broken
connection thus a long delay before the `Broken pipe` message.

            root@AudioPi:~# raspi-config
            ...
            root@AudioPi:~# poweroff
            W: molly-guard: SSH session detected!
            Please type in hostname of the machine to reboot: AudioPi
            packet_write_wait: Connection to 192.168.77.123 port 22: Broken pipe

This completes the initial setup.

---
<hr />


# rockpro64
A log of installation procedures for Armbian, Nextcloudpi and Grocy on the RockPro64

## Download Armbian image and verify
### Download
Link: https://dl.armbian.com/rockpro64/Bionic_current

As of 20200404 this is `Armbian_20.02.1_Rockpro64_bionic_current_5.4.20.7z`

I chose this version because I don't need a desktop and I wanted the latest kernel.

### Extract

`7z x Armbian_20.02.1_Rockpro64_bionic_current_5.4.20.7z`

### Verify the image's signature

```
#download public key from the database
gpg --keyserver ha.pool.sks-keyservers.net --recv-key DF00FAF1C577104B50BF1D0093D6889F9F0E78D5
#
gpg --verify Armbian_20.02.1_Rockpro64_bionic_current_5.4.20.img.asc
gpg: assuming signed data in 'Armbian_20.02.1_Rockpro64_bionic_current_5.4.20.img'
gpg: Signature made Mo 17 Feb 2020 13:39:05 CET
gpg:                using RSA key DF00FAF1C577104B50BF1D0093D6889F9F0E78D5
gpg: Good signature from "Igor Pecovnik <igor@armbian.com>" [unknown]
gpg:                 aka "Igor Pecovnik (Ljubljana, Slovenia) <igor.pecovnik@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: DF00 FAF1 C577 104B 50BF  1D00 93D6 889F 9F0E 78D5
```

Apparently, the warning can be ignored (source: https://docs.armbian.com/User-Guide_Getting-Started/)

### Verify download integrity using sha256sum
```
sha256sum -c Armbian_20.02.1_Rockpro64_bionic_current_5.4.20.img.sha
Armbian_20.02.1_Rockpro64_bionic_current_5.4.20.img: OK
```

### Format SD card
```
sudo mkdosfs -F 32 -I /dev/mmcblk0p1
```
### Burn image using 'etcher'
https://www.balena.io/etcher/

## First boot
Insert SD card into RockPro64 slot.
Connect power and ethernet cable.
Press power button.
Look up IP in FritzBox: device is simply called 'rockpro64'

ssh into device

ssh root@192.168.178.55

PW is 1234

You're prompted to immediately change the root PW.
Then you're prompted to add a sudo-enabled new user.

It's a good idea to then install updates
sudo apt update
sudo apt upgrade

Set a static IP (in FritzBox and/or in RockPro64)
armbian-config > network

Install docker from armbian-config > Software > Softy > Docker

## Nextcloudpi
### USB flash drive prep
#### format as BTRFS
```
sudo mkfs.btrfs -L USBDRIVE -f /dev/sda
btrfs-progs v4.15.1
See http://btrfs.wiki.kernel.org for more information.

Label:              USBDRIVE
UUID:               abfdb7fb-bf06-433e-abad-7fc3437c7c51
Node size:          16384
Sector size:        4096
Filesystem size:    57.30GiB
Block group profiles:
  Data:             single            8.00MiB
    Metadata:         DUP               1.00GiB
      System:           DUP               8.00MiB
      SSD detected:       no
      Incompat features:  extref, skinny-metadata
      Number of devices:  1
      Devices:
         ID        SIZE  PATH
             1    57.30GiB  /dev/sda
```

#### mounting
Create mount point
```
sudo mkdir /media/usb0
```
Add entry to `/etc/fstab`
```
UUID=abfdb7fb-bf06-433e-abad-7fc3437c7c51 /media/usb0 btrfs users,rw,exec 0 0
```
mount as user
```
mount /media/usb0
```
Change the external drive's owner to local user.
```
sudo chown daniel:daniel /media/ncp
```

##### Permission probs
Make sure that the user has execute rights in the folder that will be used for NextCloud, i.e. the drive must be mounted with `exec` and the 
Local user must have write and execution rights, i.e. the drive must be mounted using the `exec,rw` options and the directory must have write and execution rights. Otherwise 'Permission denied' errors will pop up when trying to use the NextCloudPi docker.
```
otherwise permission denied
sudo docker logs -f nextcloudpi
Initializing empty volume..
Making /usr/local/etc/ncp-config.d persistent ...
Making /etc/services-enabled.d persistent ...
Making /etc/letsencrypt persistent ...
Making /etc/shadow persistent ...
Making /etc/cron.d persistent ...
Making /etc/cron.daily persistent ...
Making /etc/cron.hourly persistent ...
Making /etc/cron.weekly persistent ...
Making /usr/local/bin persistent ...
/run-parts.sh: line 47: /etc/services-enabled.d/010lamp: Permission denied
/run-parts.sh: line 47: /etc/services-enabled.d/020nextcloud: Permission denied
Init done
/run-parts.sh: line 6: /etc/services-enabled.d/020nextcloud: Permission denied
/run-parts.sh: line 6: /etc/services-enabled.d/010lamp: Permission denied
/run-parts.sh: line 6: /etc/services-enabled.d/000ncp: Permission denied
/run-parts.sh: line 42: /etc/services-enabled.d/000ncp: Permission denied
/run-parts.sh: line 47: /etc/services-enabled.d/010lamp: Permission denied
/run-parts.sh: line 47: /etc/services-enabled.d/020nextcloud: Permission denied
Init done
```

### NextCloudPi using the docker image
#### Docker
Pull the image
```
sudo docker pull ownyourbits/nextcloudpi
```

Create and run new container
```
sudo docker run -d -p 4443:4443 -p 443:443 -p 80:80 -v /media/usb0/ncp:/data --name nextcloudpi ownyourbits/nextcloudpi 192.168.178.64
```
This creates a detached (`-d`) container called 'nextcloudpi'.
Guest ports 4443, 443 and 80 are mapped to the same ports of the host OS.
Guest directory `/data` is mapped to host directory `/media/usb0/ncp`.
IP 192.168.178.64 is added to the trusted domains of nextcloud.


Wait for 'Init done' in `sudo docker logs -f nextcloudpi`. This will take a few minutes.

#### Setup NextCloudpi
Access NextCloudPi instance by typing the RockPro's IP into the browser.
There will probably be a certificate error.
A website with the NextCloudPi user and password and the NextCloud user and password will appear. Save the passwords and click 'activate'. The page reloads and there is another certificate error. Log in using the NCP user (the first password from the page before).
##### NextCloudPi configuration wizard
Forward ports 80 and 443 in FritzBox: Internet > Freigaben > rockpro64
IPv6 Interface ID: last four groups of first ip6 address given by ip -6 addr

###### DDNS
DuckDNS is actually not available in the wizard.
The implementation in the NCP dashboard only supports duckdns.org's automatic IPv4 detection.
Create a NetworkManager dispatcher script in `/etc/NetworkManager/dispatcher.d/30-upduckdns` that executes `/usr/local/bin/duckdns_update_ip6.sh` on every reconnect.
This script gets the current IP6 and updates the DNS entry using `/etc/duckdns.org/token` and `/etc/duckdns.org/domains`.
Make sure all of these scripts are executable.

When I visited mydomain.duckdns.org from outside my network (mobile internet, VPN), I could access NextCloud.
To be able to access it via the name from within the network I needed to add an exception for the domain in the FritzBox: DNS Rebind exception.

##### Let's Encrypt
Currently NextCloudPi doesn't support wildcard certificates, so this has to be done externally.
Install `certbot` per their instructions and run this command.
```
sudo certbot certonly \
    --manual \
    --manual-public-ip-logging-ok \
    --non-interactive \
    --agree-tos \
    -m <name@email.com> \
    --preferred-challenges dns \
    --domains <single_domain> \
    --manual-auth-hook /usr/local/bin/certbot_authenticator.sh \
    --manual-cleanup-hook /usr/local/bin/certbot_cleanup.sh \
    --server https://acme-v02.api.letsencrypt.org/directory
```
This has to be done once each for `yourdomain.duckdns.org` and `*.yourdomain.duckdns.org` because DuckDNS doesn't appear to support multiple TXT records.
The scripts `certbot_authenticator.sh` and `certbot_cleanup.sh` put the TXT record at `_acme-challenge.yourdomain.duckdns.org` and clears them after authentication.
If authentication was successful, certbot should have created a cronjob for renewal.

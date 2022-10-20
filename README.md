# Raspberry Pi Provisioning Setup


## References

https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html


## Final Setup

The installation uses a folder *secret* containing the credential files. *secret* is not checked in, and needs to be provided manually as shown below.  

For my embedded automation controller I use the following setup:  

- **dhcp client** on wlan0 (with configured wpa_supplicant from *secret*), as uplink
- rootfs expanded to the entire SD card
- Serial console login enabled
- Early output on serial enabled
- Bluetooth disabled to make console print readable (PI issue)
- SSH daemon enabled
- Locale US_en.UTF-8
- screen using CTRL-b (emacs user)
- vimrc, emacsrc, mc, bashrc, etc. environment settings
- Camera (legacy) enabled, setup for motion (useful to remote observe LEDs blinking)
- ~/.local is a symlink to /usr/local i.e. actually a one-user-system
- Login: u: pi / p: xdr5XDR%  or auto-login

login: pi / xdr5XDR%  

## Preparation

On host PC  

```
$ pip3 install --user ansible
```

## Download PI/OS image (32 bit)

Raspi OS image for Zero Pi [32 bit], plug SD card in reader  
```
$ mkdir ./sd/download
$ cd ./sd/download
$ wget https://downloads.raspberrypi.org/raspios_lite_arm/images/raspios_lite_arm-2022-09-26/2022-09-22-raspios-bullseye-arm-lite.img.xz
$ export RASPIMG=2022-09-22-raspios-bullseye-arm-lite.img.xz
$ unxz "$RASPIMG"
```

## SD card: Prepare Secrets and Credentials

Prepare a folder ``secret`` and provide content as follows  
```
$ cd ./sd
$ mkdir ./secret
...
$ tree ./secret/ -a
./secret/
    ├── etc
    │   ├── network
    │   │   └── interfaces
    │   └── wpa_supplicant
    │       └── wpa_supplicant.conf
    └── home
        └── pi
            ├── .gitconfig
            └── .ssh
                ├── id_ed25519
                └── known_hosts
```

Example: interfaces, e.g. could be extended with further network connections to work, and corresponding wpa_supplicant entries.  
```
$ cat ./secret/etc/network/interfaces
    # interfaces(5) file used by ifup(8) and ifdown(8)
    # Include files from /etc/network/interfaces.d:
    source /etc/network/interfaces.d

    auto lo
    iface lo inet loopback

    auto wlan0
    allow-hotplug wlan0
    iface wlan0 inet manual
    wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
    #wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
    wireless-power off

    ## home wifi
    iface home inet dhcp

    ## demo: dynamic and static setup
    #iface demosetup inet dhcp
    #
    #iface demosetup inet static
    #    address 192.168.1.222
    #    netmask 255.255.255.0
```

## Setup SD card

Plug card into card reader.  
```
$ lsblk
   -> /dev/sdi

$ cd ./sd
$ ./setup.sh /dev/sdi
```

## Setup PiZero target

Plug SD card into the PiZero and power the board. When it is up and running, figure out IP (nmap or serial connection). Optionally verify the board is up.  
```
$ cd ./ansible
$ ansible all -m ping
```

Optionally update ssh known_hosts, NB: ``ssh-keyscan`` should return something, when the device is up and running  
```
$ ssh-keygen -f "/home/user/.ssh/known_hosts" -R "10.1.10.203"

$ ssh-keyscan 10.1.10.203 >> ~/.ssh/known_hosts
    # 10.1.10.203:22 SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1
    # 10.1.10.203:22 SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1
    # 10.1.10.203:22 SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1
    # 10.1.10.203:22 SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1
    # 10.1.10.203:22 SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1

```

Execute ansible provisioning  
```
$ cd ./ansible
$ ansible-playbook -K ./rpi-conf.yml
```


## Issues

*issue*: prefer pip installed ansible?  

```
$ pip3 install --upgrade --user ansible
$ pip3 show ansible
```

*issue*: ping fails  
```
$ ansible raspi -m ping
10.1.10.200 | UNREACHABLE! => {
    "changed": false,
	    "msg": "Failed to connect to the host via ssh: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\r\n@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @\r\n@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\r\nIT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!\r\nSomeone could be eavesdropping on you right now (man-in-the-middle attack)!\r\nIt is also possible that a host key has just been changed.\r\nThe fingerprint for the ED25519 key sent by the remote host is\nSHA256:vH4JKH+RXxG85SiYz26U7xX7aCgZ1a/YqF5Ip643vVQ.\r\nPlease contact your system administrator.\r\nAdd correct host key in /home/user/.ssh/known_hosts to get rid of this message.\r\nOffending ECDSA key in /home/user/.ssh/known_hosts:78\r\n  remove with:\r\n  ssh-keygen -f \"/home/user/.ssh/known_hosts\" -R \"10.1.10.200\"\r\nHost key for 10.1.10.203 has changed and you have requested strict checking.\r\nHost key verification failed.",
		    "unreachable": true
			}
```
*fix*: adjust .ssh/known_hosts  
```
ssh-keygen -f "/home/user/.ssh/known_hosts" -R "10.1.10.203"
```


*issue*: login failed, no login possible  

*fix*: provide a /boot/userconf.txt file, e.g. when SD card is mounted  
```
$ echo -n "pi:" > ./boot/userconf.txt
$ echo 'mypassword' | openssl passwd -6 -stdin >> /boot/userconf.txt
```

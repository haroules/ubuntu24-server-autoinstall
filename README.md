# Ubuntu 24 Server autoinstallation from PXE with UEFI and secure boot

<!-- TOC -->

- [Ubuntu 24 Server autoinstallation from PXE with UEFI and secure boot](#ubuntu-24-server-autoinstallation-from-pxe-with-uefi-and-secure-boot)
- [Introduction](#introduction)
- [Setup and Configuration steps](#setup-and-configuration-steps)
    - [Update and install dnsmasq  provide dhcp, bootp and tftp](#update-and-install-dnsmasq--provide-dhcp-bootp-and-tftp)
    - [Get your network adapter name](#get-your-network-adapter-name)
    - [Create config file for dhcp/tftp and create filesystem folder for distribution point](#create-config-file-for-dhcptftp-and-create-filesystem-folder-for-distribution-point)
    - [Prepare pxe and boot images as well as installation media](#prepare-pxe-and-boot-images-as-well-as-installation-media)
    - [Prepare grub configuration](#prepare-grub-configuration)
    - [Install apache web server to serve config files, iso and other boot files](#install-apache-web-server-to-serve-config-files-iso-and-other-boot-files)
    - [Autoinstall configuration file used by subiquity for unattended install](#autoinstall-configuration-file-used-by-subiquity-for-unattended-install)
        - [Editing, validating the user-data file and putting it in place](#editing-validating-the-user-data-file-and-putting-it-in-place)
        - [how to hash password for the yaml file](#how-to-hash-password-for-the-yaml-file)
        - [Validate schema and syntax of autoinstall.yaml  aka user-data](#validate-schema-and-syntax-of-autoinstallyaml--aka-user-data)
        - [Copy to distribution](#copy-to-distribution)
    - [Verification before first run](#verification-before-first-run)
        - [Verify services are listening](#verify-services-are-listening)
        - [Verify the directory structure inside your distribution folder  /srv/tftp](#verify-the-directory-structure-inside-your-distribution-folder--srvtftp)
        - [Verify web distribution point](#verify-web-distribution-point)
- [Ready to test !](#ready-to-test-)
- [Working with Secure Boot and some pitfalls to be wary of](#working-with-secure-boot-and-some-pitfalls-to-be-wary-of)
- [Important links for documentation and debugging](#important-links-for-documentation-and-debugging)

<!-- /TOC -->



# Introduction
- For this example i am using Ubuntu 24.04.1 LTS desktop as the PXE/TFTP/BootP server, and I am installing 24.04.1 server to bare metal.

- The example configs for the autoinstall assume you are using an SSD/NVMe drive with at least 300GB of storage available. It also is configured to use LVM with encryption, more on this in the autoinstall section. You will want to likely change the autoinstall to match your disk size and preferences.

- Through trial and error, I have come to the conclusion that it's easiest to do this on an isolated network segment since multicast, and random ephemeral UDP ports are used making firewall configuration with UFW very complicated. I found that temporarily disabling firewall while testing and making things functional eliminates a lot of networking challenges. tcpdump will be your friend should you walk the lonely road to a functional firewalled host configuration. 

- If you have a dhcp server on the same network segment as your tftp/bootp server, make sure to disable bootp queries and tftp on the device, in my case a firewall/gateway/network appliance. If possible, do not have another dhcp server on that segment, or plan to smash your head on DHCP option sets and other challenges.

- I have included the config files i used, you will want to edit them for your particular network, you will also want to change the default user password, and lvm crypt password (more on this in the autoinstall configuration section)

- Please read the section regarding Secure Boot.  This can be really painful with 3rd party drivers like nvidia for graphics, and also the PXE Boot loader complaining about signatures. Since this is a server install, there's alot less drivers to deal with than desktop. My configuration will work with secure boot, but it's a process that involves entering bios before and after the install.

- File naming conventions, directory structure and configuration options in the user-data section are very specific in regards to name and layout.  I'll try to do my best to point those out where possible, and provide a directory structure.  I'll also provide links to the documentation that got me working.

# Setup and Configuration steps

## 1. Update and install dnsmasq ( provide dhcp, bootp and tftp)
- sudo apt update
- sudo apt install dnsmasq

## 2. Get your network adapter name 
run command: ip -a

in this example en01 is the adapter name. 

```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: en01: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 23:87:fb:3d:11:b5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.5/24 brd 192.168.1.255 scope global noprefixroute en01
       valid_lft forever preferred_lft forever

```

## 3. Create config file for dhcp/tftp and create filesystem folder for distribution point
* This content exists in the repository, either copy from git cloned workspace or copy example below.
- sudo vi /etc/dnsmasq.conf.d/pxe.conf

replace adapter name and dhcp IP range to fit your need. An example:
```
interface=en01
bind-interfaces
dhcp-range=en01,192.168.1.100,192.168.1.120
dhcp-boot=grubx64.efi
enable-tftp
tftp-root=/srv/tftp
```

for the purpose of this example I chose to use /srv/tftp to be the distribution folder, whatever you use in pxe.conf, you should ensure exists, as this is also refrenced in the grub configuration.

- sudo mkdir /srv/tftp
- sudo systemctl restart dnsmasq.service

## 4. Prepare pxe and boot images as well as installation media
- sudo apt install cd-boot-images-amd64
- sudo ln -s /usr/share/cd-boot-images-amd64 /srv/tftp/boot-amd64
- wget https://cdimage.ubuntu.com/ubuntu-server/noble/daily-live/current/noble-live-server-amd64.iso
- sudo mount noble-live-server-amd64.iso /mnt
- sudo cp -v /mnt/casper/{vmlinuz,initrd} /srv/tftp/
- sudo umount /mnt
- sudo cp -v noble-live-server-amd64.iso /srv/tftp/
- apt download shim-signed
- apt download grub-efi-amd64-signed
- apt download grub-common
- dpkg-deb --fsys-tarfile shim-signed*deb | tar x ./usr/lib/shim/shimx64.efi.dualsigned -O > bootx64.efi
- dpkg-deb --fsys-tarfile grub-efi-amd64-signed*deb | tar x ./usr/lib/grub/x86_64-efi-signed/grubnetx64.efi.signed -O > grubx64.efi
- dpkg-deb --fsys-tarfile grub-common*deb | tar x ./usr/share/grub/unicode.pf2 -O > unicode.pf2
- sudo cp -v bootx64.efi /srv/tftp/bootx64.efi
- sudo cp -v grubx64.efi /srv/tftp/grubx64.efi
- sudo cp -v unicode.pf2 /srv/tftp/unicode.pf2

## 5. Prepare grub configuration
- sudo mkdir /srv/tftp/grub
- sudo vi /srv/tftp/grub/grub.cfg

see the included grub.cfg file for content.  You will need to edit the example to reflect:

1. The IP of the DHCP/Bootp server (also serving images and other content)
2. Paths to the content if you chose to rename your files pr distribution directory.

## 6. Install apache web server to serve config files, iso and other boot files
* while not covered here, i would recommend disabling the welcome page and securing apache beyond just the default install. Hardening apache is not covered here.

- sudo apt install apache2
- sudo vi /etc/apache2/conf-available/tftp.conf

```
<Directory /srv/tftp>
        Options +FollowSymLinks +Indexes
        Require all granted
</Directory>
Alias /tftp /srv/tftp
```

- cd /etc/apache2/conf-available
- a2econf tftp.conf
- sudo systemctl restart apache2

## 7. Autoinstall configuration file (used by subiquity for unattended install)
The autoinstall.yaml ( example in the source pack ) gets copied into the distribution point as user-data, and is what drives subiquity to perform unattended (pre-selected configuration) install options. I edit and rename on the copy since the file as needed by the autoinstall process must be called user-data, where as the sample from Ubuntu is named as I have included it. 

### Editing, validating the user-data file and putting it in place
I edit the autoinstall.yaml file, validate, source control, and then copy to the distribtion folder. For the purposes of keeping this simpler to follow, i will just share how to do minimal validation, and the tools needed to update hashed password for the initial user.

relevant mappings to look at and update:
- identity   (hostname, username, password)
- timezone   
- locale
- storage (the lvm crypt key is required during boot to unlock the disk)  
- network  (in this example i set a static address, and static dns and search domain)
- packages  (additional packages beyond "minimal install")
- late commands (post install/config changes like enabling firewall, turning off ipv6, disabling services)

### how to hash password for the yaml file
* for the password field you have to provide a hashed value, not a plaintext one, this may be confusing since the lvm crypt key for setting disk encryption does accept plain text.
sudo apt install whois   (package needed for mkpasswd utility)
mkpasswd --method=sha-512 password   (change "passowrd" to what you intend to set)

### Validate schema and syntax of autoinstall.yaml ( aka user-data )
The validation i believe really only verifies schema and yaml syntax, but at least it might save from subiquity errors.
- sudo cloud-init schema -c autoinstall.yaml --annotate

### Copy to distribution
- sudo cp -v autoinstall.yaml /srv/tftp/user-data

## 8. Verification before first run

### Verify services are listening

run netstat -l

assuming nothing else running you should see something like below.
were' looking for http,bootps,tftp 

```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 _localdnsproxy:domain   0.0.0.0:*               LISTEN     
tcp        0      0 _localdnsstub:domain    0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:http            0.0.0.0:*               LISTEN     
tcp        0      0 host.example:domain     0.0.0.0:*               LISTEN     
tcp        0      0 localhost:domain        0.0.0.0:*               LISTEN     
udp        0      0 localhost:domain        0.0.0.0:*                          
udp        0      0 host.example:domain     0.0.0.0:*                          
udp        0      0 _localdnsproxy:domain   0.0.0.0:*                          
udp        0      0 _localdnsstub:domain    0.0.0.0:*                          
udp        0      0 0.0.0.0:bootps          0.0.0.0:*                          
udp        0      0 localhost:tftp          0.0.0.0:*                          
udp        0      0 host.exampl:domain:tftp 0.0.0.0:*        
```

### Verify the directory structure inside your distribution folder ( /srv/tftp )
user@host1:/srv/tftp$ tree
```
.
├── boot-amd64 -> /usr/share/cd-boot-images-amd64
├── bootx64.efi
├── grub
│   └── grub.cfg
├── grubx64.efi
├── initrd
├── noble-live-server-amd64.iso
├── unicode.pf2
├── user-data
└── vmlinuz
```
### Verify web distribution point

open a browser and go to  http://localhost/tftp/  you should see a structure like above

curl or wget locally from the distribution point a small sample file, and validate it's contents such as:
wget http://localhost/tftp/grub/grub.cfg

# Ready to test !
At this point you are ready to test your PXE boot environment this assumes that:
- Firewall is configured or temporarily disabled and that your distribution host and target host are wired and on a private network segment firewalled from the rest of your network.
- dnsmasq is configured and running
- Apache is configured and running
- All the config files have the appropriate edits and are staged in the distribution area.
- Your target bare metal host bios has secure boot either disabled, or put into "audit" mode
- You have configured the UEFI stack and PXE options to be able to boot on target bare metal host

# Working with Secure Boot and some pitfalls to be wary of
I was unable to get a working pxe boot solution on UEFI stack with secure boot set to enforced prior to install. I was however able to set the bios for secure boot to "audit", perform the unattended installation, then return to bios and enable secure boot to enforce. 

getting nvidia drivers to work was a challenge, perhaps will include in future update.

Secure boot was validated by running the following CLI command (after secure boot was enabled in BIOS):
sudo mokutil --sb-state

The reasons for the issues with secure boot are complicated. Most likely is that my hardware is so old that i could not get Ubuntu's signature to be verified, since no updated firmware for my motherboard exist.  I did experiment with creating personal signing keys and importing them, but that caused other issues, but appeared to work and far too messy to consider.

Other mistakes along the way included specifying the kernel in the user-data section along with the oem metadata flag. Since i didn't RTFM (I included the link below), and it wasn't immediately apparent that the oem metadata package contains drivers and content that are kernel specific and that the flags can cause kernel install failures due to package dependency issues, i had subiquity crashing during kernel install.

Getting the pxe boot sequence down required alot of tcpdump, wireshark, and log traversing. One of the biggest challenges was getting debug output from screens that would refresh too fast, and or non specific error messaging.  I chose to follow examples from Ubuntu's documentation. I know now there are alot of other pxe tooling available that may be better than the minimal config i used, and may be explored in the future.

I wasted alot, ALOT of time trying to create a firewalled (at the dhcp/bootp/tftp host) setup and confg.  Stateless connections, and a whole lot of multicast and udp and ephemeral ports lead me to my conclusions.  Temporarily disable firewall on the distribution point, run the test install, enable firewall is as far as i got for a process.

# Important links for documentation and debugging
the autoinstall.yaml params and what they mean, default values etc.
- https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html  

dealing with storage config
- https://docs.cloud-init.io/en/latest/reference/yaml_examples/index_fs.html  

cloud config and everything you want to know about it related to autoinstall
- https://canonical-subiquity.readthedocs-hosted.com/en/latest/explanation/cloudinit-autoinstall-interaction.html#cloudinit-autoinstall-interaction

explanation on how to provide configuration to autoinstall and the various files and how they are used
- https://canonical-subiquity.readthedocs-hosted.com/en/latest/tutorial/providing-autoinstall.html#providing-autoinstall

docs on ubiquity, pxe boot and initial guides
- https://ubuntu.com/server/docs/how-to-netboot-the-server-installer-on-amd64
- https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html
- https://wiki.ubuntu.com/UEFI/SecureBoot/PXE-IPv6
- https://docs.cloud-init.io/en/latest/reference/network-config.html
- https://docs.cloud-init.io/en/latest/explanation/format.html#user-data-formats-cloud-config

github source and documentation for autoinstaller for desktop (not server but had helpful information)
- https://github.com/canonical/autoinstall-desktop/blob/main/autoinstall.yaml


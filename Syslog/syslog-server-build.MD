# Syslog Server Build

- [Syslog Server Build](#syslog-server-build)
  - [Some conventions for this document](#some-conventions-for-this-document)
    - [Company Domain](#company-domain)
    - [Server Naming](#server-naming)
    - [Zabbix](#zabbix)
    - [Linux Version](#linux-version)
  - [Virtual Machine Information \& Resources](#virtual-machine-information--resources)
  - [Date and Time](#date-and-time)
    - [Fix Timezone](#fix-timezone)
    - [Validate NTP - Chrony](#validate-ntp---chrony)
  - [Mount Log Volume](#mount-log-volume)
    - [Partition the drive](#partition-the-drive)
    - [Format volume](#format-volume)
    - [Temporarily mount device](#temporarily-mount-device)
    - [Update fstab](#update-fstab)
  - [Generate new SSH Key](#generate-new-ssh-key)
  - [Verify Firewall Configuration](#verify-firewall-configuration)
    - [Server Farm Firewall](#server-farm-firewall)
      - [Inbound Requirements](#inbound-requirements)
      - [Outbound Requirements](#outbound-requirements)
    - [Local System Firewall](#local-system-firewall)
    - [Load Balancer Configuration](#load-balancer-configuration)
      - [Netscaler Configuration](#netscaler-configuration)
      - [Fortinet Configuration](#fortinet-configuration)
      - [Other](#other)
  - [Install Duo](#install-duo)
    - [Install software dependencies](#install-software-dependencies)
    - [Download Client](#download-client)
    - [Extract and Compile](#extract-and-compile)
    - [Setup Config Files](#setup-config-files)
    - [Configure PAM](#configure-pam)
    - [Update SSHD](#update-sshd)
  - [Join to AD](#join-to-ad)
    - [Install needed applications](#install-needed-applications)
    - [Configure AD Related Files](#configure-ad-related-files)
      - [resolve.conf](#resolveconf)
      - [krb5.conf](#krb5conf)
    - [Satellite Setup](#satellite-setup)
      - [Realm Clean Up](#realm-clean-up)
      - [Join AD](#join-ad)
      - [Fix Login Rights](#fix-login-rights)
      - [sssd.conf](#sssdconf)
      - [Restart SSHD](#restart-sshd)
  - [Authors](#authors)

---

## Some conventions for this document

### Company Domain

For this document our company domain is ```mydomain.com``` and all examples will reference that. The AD domain will be ```ad.mydomain.com```.

### Server Naming

For the purposes of this document we are going to assume you have to different data centers dc-east (for Datacenter East) and dc-west that will be identified as dce and dcw. So for each data center we would name them ```syslog-dce-0.ad.mydomain.com``` and ```syslog-dcw-0.ad.mydomain.com```. If we wanted to add another pair in each datacenter then we would increment the counter and the names would be ```syslog-dce-1.ad.mydomain.com``` and ```syslog-dcw-1.ad.mydomain.com```

### Zabbix

Zabbix is the monitoring system that I am most used to. I like combining Zabbix with Grafana for easy to read dashboards for the health of Splunk and Syslog. This is by no means a requirement or any endorsement. Use the one you like the most. You will just need to adjust the documentation to support your chosen monitoring solution.

### Linux Version

We make use of Red Hat Enterprise Linux. If you use a different variant then you make need to adjust some of the following actions slightly. 

---

## Virtual Machine Information & Resources

If you have the ability to build more than one syslog server it is strongly recommended that you do so. This allows for you to perform reboots and upgrades without the risk of loosing any syslog data. 


|                     |                                        |
|---------------------|----------------------------------------|
| Type                | VMware ESX Virtual Machine v14         |
| Virtual CPUs        | 4 vCPUs                                |
| Memory              | 16,384 MB                              |
| Resource Pool       | Production VMs (High)                  |
| Storage             | (1) 40 GB VMDK on LSI Logic Parallel   |
|                     | (2) 1 TB VMDK on LSI Logic Parallel    |
| Storage Shares      | (1) Normal                             |
|                     | (2) Normal                             |
| Network Adapter(s)  | (1) VMXNET 3 on Splunk-MSB [ FOR MSB ] |
| Network Adapter(s)  | (1) VMXNET 3 on Splunk-HBL [ FOR HBL ] |

## Date and Time

### Fix Timezone

In many cases a fresh install of the OS will have the box in UTC mode. This is for EDT/EST you would use the region that reflects your time zone.

```sh
timedatectl set-timezone America/New_York
```

If you are going to add any cron jobs you will need to reboot at this time.

### Validate NTP - Chrony

Make sure that the pool for the chronyd is set to your organizations local time server. If not then change it the ```/etc/chrony.conf``` file:

```conf
pool time.mydomain.com
```

The restart the service:

```sh
systemctl restart chronyd.service
```

## Mount Log Volume

When building the virtual machine, make sure that the Log volume is added ***AFTER*** the deployment, otherwise it will part of the LVM and that is not going to be ideal.

### Partition the drive

The Log volume will typically be /dev/sdb

To make sure run

```sh
fdisk /dev/sdb
```

You should see the following output.

```text
Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x1155d67e.

Command (m for help): p
Disk /dev/sdb: 1 TiB, 1099511627776 bytes, 2147483648 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x1155d67e

Command (m for help):
```

Type ```n``` and then press enter. Then you should be able to press enter three more times just excepting the defaults.

Example output:

```text
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-2147483647, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-2147483647, default 2147483647):

Created a new partition 1 of type 'Linux' and of size 1024 GiB.

Command (m for help):
```

Press ```p``` to view and confirm the partition size. Then press ```w``` to write the changes and quit.

Example output:

```text
Command (m for help): p
Disk /dev/sdb: 1 TiB, 1099511627776 bytes, 2147483648 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x1155d67e

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdb1        2048 2147483647 2147481600 1024G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@syslog-msb-0 ~]#
```

Make note of the name of the device. In this example it is ```/dev/sdb1```.

### Format volume

Ideally you will use XFS

```sh
mkfs.xfs /dev/sdb1
```

### Temporarily mount device

The existing log data needs to be moved to the new volume. We are going to mount the device to a temporary folder, copy the existing log data and unmount the volume

```sh
mkdir /mnt/logs
mount /dev/sdb1 /mnt/logs/
cd /var/log
cp -Rp * /mnt/logs/
umount /mnt/logs
```

### Update fstab

Edit the file ```/etc/fstab```

You will need to add the device {in this case ```/dev/sdb1```} to the file. Make sure you use spaces and ***NOT*** tabs

The final result should look similar to below.

***NOTE:*** There will be some differences. For example the LVM name will reflect your host and not the one in this example.

```text
#
# /etc/fstab
# Created by anaconda on Wed Nov 13 12:27:15 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
#
# After editing this file, run 'systemctl daemon-reload' to update systemd
# units generated from this file.
#
/dev/mapper/rhel_syslog--dce--0-root      /                       xfs     defaults        0 0
UUID=e8b8ef48-202f-4a23-a88d-4a389e4675d8 /boot                   xfs     defaults        0 0
/dev/mapper/rhel_syslog--dce--0-swap      swap                    swap    defaults        0 0
/dev/sdb1                                 /var/log                xfs     defaults        0 0
```

Now mount the Log volume by running:

```sh
mount -a
```

## Generate new SSH Key

Run:

```sh
ssh-keygen -b 4096
```

Output should look something like:

```text
[root@syslog-msb-0 opt]# ssh-keygen -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
/root/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:oQcyHyun5ga1wvBQPEvYWr2mpVSLWKvx+p7Yy5smi9Y root@syslog-msb-0.grove.ad.uconn.edu
The key's randomart image is:
+---[RSA 4096]----+
| + .             |
|. O o            |
| B *ooo .        |
|* = *+ = .       |
| X *..= S        |
|. O .+ .         |
| ..oo            |
|+=.E.            |
|=*X+.            |
+----[SHA256]-----+
```

Verify security permissions

```sh
chmod 600 ~/.ssh/*
```

---

## Verify Firewall Configuration

### Server Farm Firewall

#### Inbound Requirements

| From Interface | From Host           | To Interface | To Host             | Service              |
|----------------|---------------------|--------------|---------------------|----------------------|
| external       | satellite.uconn.edu | internal     | Syslog Hosts        | Satellite Services   |
| external       | IT Admin Net        | internal     | Syslog Hosts        | SSH                  |
| external       | IT Privileged VPN   | internal     | Syslog Hosts        | SSH                  |
| external       | Zabbix Servers      | internal     | Syslog Hosts        | Zabbix Agent Service |
| external       | Syslog Sources      | internal     | Syslog Hosts        | Syslog               |

#### Outbound Requirements

| From Interface | From Host           | To Interface | To Host             | Service              |
|----------------|---------------------|--------------|---------------------|----------------------|
| internal       | Syslog Hosts        | external     | ALL                 | HTTPS                |
| internal       | Syslog Hosts        | external     | ALL                 | PING                 |
| internal       | Syslog Hosts        | external     | ALL                 | SSH                  |
| internal       | Syslog Hosts        | external     | ALL                 | STASH-SSH            |
| internal       | Syslog Hosts        | external     | Domain Controllers  | Windows AD           |
| internal       | Syslog Hosts        | external     | satellite.uconn.edu | Satellite Services   |
| internal       | Syslog Hosts        | external     | IPADM Servers       | DHCP / NTP / DNS     |
| internal       | Syslog Hosts        | external     | SPLUNK_DEPLOY       | Splunk Management    |
| internal       | Syslog Hosts        | external     | SPLUNK_INDEXERS     | Splunk Indexing      |
| internal       | Syslog Hosts        | external     | Zabbix Servers      | Zabbix Agent Service |

### Local System Firewall

Add Syslog ports but remove access to cockpit, because why offer an opportunity to pwn the box.

```sh
firewall-cmd --zone=public --permanent --add-port=514/tcp
firewall-cmd --zone=public --permanent --add-port=514/udp
firewall-cmd --zone=public --permanent --remove-service=cockpit
firewall-cmd --reload
```

### Load Balancer Configuration

Coming soon [maybe] ;)

#### Netscaler Configuration

To be created

#### Fortinet Configuration

To be created

#### Other

Anyone want to add Palo, F5 and Cisco?

---

## Install Duo

### Install software dependencies

 ```sh
dnf -y groupinstall "Development Tools"
dnf -y install openssl-devel pam-devel
```

### Download Client

```sh
curl https://dl.duosecurity.com/duo_unix-latest.tar.gz -o duo_unix-latest.tar.gz
```

### Extract and Compile

NOTE: At the time of writing this guide the latest version is 1.11.3. You make need to update the commands to change to the directory with future updates.

Also, you will only need the last three lines if you have SELinux enabled.

```sh
tar -zxvf duo_unix-latest.tar.gz
cd duo_unix-1.11.3
./configure --with-pam --prefix=/usr && make && sudo make install

make -C pam_duo semodule
make -C pam_duo semodule-install
setsebool -P authlogin_yubikey=1
```

### Setup Config Files

This will setup the correct directories and files required to use Duo. If you can do it, I would recommend a custom integration for Splunk that is used on Splunk and Syslog servers.

You will need to get the ```ikey```, ```skey```, and ```host``` from your IAM group.

```sh
mkdir /etc/duo
cd /etc/duo
rm -f pam_duo.conf
ln -s /etc/duo/login_duo.conf pam_duo.conf

rm -rf /etc/login_duo.conf
rm -rf /etc/pam_duo.conf


echo "[duo]
; Duo integration key
ikey = [GET_FROM_IAM_GROUP]
; Duo secret key
skey = [GET_FROM_IAM_GROUP]
; Duo API host
host = [GET_FROM_IAM_GROUP]
; Send command for Duo Push authentication
;pushinfo = yes
" > /etc/duo/login_duo.conf

cd /etc/
ln -s /etc/duo/login_duo.conf
ln -s /etc/duo/pam_duo.conf

chmod 600 /etc/duo/login_duo.conf /etc/duo/pam_duo.conf /etc/duo/*
```

### Configure PAM

Setup PAM to use Duo with SSHD

```sh
echo "#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth  required pam_env.so
auth  sufficient pam_duo.so
auth       required     /usr/lib64/security/pam_duo.so
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
" >  /etc/pam.d/sshd
```

### Update SSHD

In order to use Duo with SSHD we will need to make some changes to its configuration file.

```sh
sed -i s/\#UseDNS\ yes/UseDNS\ no/g /etc/ssh/sshd_config
sed -i s/ChallengeResponseAuthentication\ no/ChallengeResponseAuthentication\ yes/g /etc/ssh/sshd_config
sed -i s/\PermitRootLogin\ yes/#PermitRootLogin\ yes/g /etc/ssh/sshd_config

systemctl restart sshd
```

## Join to AD

### Install needed applications

```sh
dnf -y install oddjob oddjob-mkhomedir sssd adcli
/usr/bin/mv /etc/sssd/sssd.conf /etc/sssd/sssd.conf.pre_ad
```

### Configure AD Related Files

#### resolve.conf

```ini
# Generated by NetworkManager
search ad.mycompany.com
nameserver IP_OF_IPADM_OR_DC
nameserver IP_OF_2ND_IPADM_OR_DC
```

#### krb5.conf

Setup your ```/etc/krb5.conf``` file.

```ini
# To opt out of the system crypto-policies configuration of krb5, remove the
# symlink at /etc/krb5.conf.d/crypto-policies which will not be recreated.
includedir /etc/krb5.conf.d/

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    dns_lookup_realm = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
    spake_preauth_groups = edwards25519
    default_ccache_name = KEYRING:persistent:%{uid}
```

### Satellite Setup

Assuming you deployed the machine via Satellite you are likely going to have a FQDN for the host that does not match AD. We are going to fix that here so that the computer name in Satellite and actual computer name match properly.

You will need to get ```org``` and ```activationkey``` from your Satellite Admins.

```sh
subscription-manager remove --all
subscription-manager unregister
subscription-manager clean
hostnamectl set-hostname syslog-dce-0.ad.mycompany.com
subscription-manager register --name="syslog-dce-0.ad.mycompany.com" --org='Your_Organization' --activationkey='YOUR_RHEL_AKEY'
subscription-manager refresh
subscription-manager attach --auto
```

I have an ansible script that cleans out the old host from Satellite that should work for you.

```yaml
- name: "Fix hostname - Remove old host"
  uri:
    url: https://{{ satellite_server }}/api/hosts/{{ deployment_vars_ServerShortName }}.deployment.fqdn.com
    method: DELETE
    user: "{{ satellite_username }}"
    password: "{{ satellite_password }}"
    return_content: yes
    headers:
      application: json
      version: 2
    force_basic_auth: yes
  register: del_sat_host
  ignore_errors: yes

```


#### Realm Clean Up

You are not going to be part of any realm but I have noticed from time to time that the join process does not work 100% of the time without this:

```sh
realm leave
```

#### Join AD

| Variable | Meaning / Use |
|-------------------------------------|----|
| {{ default_domain_suffix }}         | In this example replace with ad.mycompany.com |
| {{ ad_join_username }}              | This is the account that has domain join permissions and create objects in the OU below |
| {{ deployment_vars_DestinationOU }} | This is the full DN of where the AD object will live example: OU=Servers,OU=Splunk,DC=ad,DC=mycompany,DC=com |

NOTE: leave the single quotes for ```--computer-ou```. Obviously remove all of the { and }.

realm join {{ default_domain_suffix }} --user {{ ad_join_username }} --computer-ou='{{ deployment_vars_DestinationOU }}'

#### Fix Login Rights

Replace ```{{ deployment_ServerAdminGroup }}``` with the name of the AD group that has users that are allowed to login to this server.

Also DOMAIN should be replaced with the NETBIOS name of the AD Domain.

```sh
realm deny --all
realm permit -g "{{ deployment_ServerAdminGroup }}"

echo "%MYCOMPANY\\{{ deployment_ServerAdminGroup }} ALL=(ALL) ALL" > /etc/sudoers.d/mycompany
```

#### sssd.conf

Setup your ```/etc/sssd/sssd.conf``` file for AD join.

| Variable | Meaning / Use |
|----------|---------------|
| default_domain_suffix | ad.mycompany.com |
| deployment_ServerAdminGroup | This is the AD group that contains accounts that can ssh to this box |

You will want to replace ```{{ default_domain_suffix }}``` with whatever your ```ad.mycompany.com``` AD FQDN is.

```ini
[sssd]
domains = {{ default_domain_suffix }}
config_file_version = 2
services = nss, pam

[domain/{{ default_domain_suffix }}]
ad_domain = {{ default_domain_suffix }}
krb5_realm = {{ default_domain_suffix|upper }}
realmd_tags = manages-system joined-with-samba 
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = False
fallback_homedir = /home/%u
access_provider = simple
dyndns_update = true
dyndns_update_ptr = false
ignore_group_members = true
simple_allow_groups = {{ deployment_ServerAdminGroup }}
```

#### Restart SSHD

Restart SSHD and then open a new terminal window and try to login. The reason for the new one is in case you have problems, you still have a way to login to the box. 

```sh
systemctl restart sshd
```


---

## Authors

Original: 

- Christopher J. Tarricone (c) 11-15-2019 

Co-Authors:
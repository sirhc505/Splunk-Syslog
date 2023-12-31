# Syslog Server Configuration

- [Syslog Server Configuration](#syslog-server-configuration)
  - [Assumptions](#assumptions)
  - [Configure GIT Repos](#configure-git-repos)
  - [Configure rsyslog](#configure-rsyslog)
    - [Use GIT to sync between syslog servers](#use-git-to-sync-between-syslog-servers)
      - [Add server key to Stash](#add-server-key-to-stash)
      - [Setup Sync Folders](#setup-sync-folders)
      - [Clone Repos](#clone-repos)
    - [Install rsyslog](#install-rsyslog)
      - [Setup Syslog Config Files](#setup-syslog-config-files)
      - [First time rsyslogd.conf](#first-time-rsyslogdconf)
      - [Make service specific directories](#make-service-specific-directories)
      - [Enable and Start rsyslogd](#enable-and-start-rsyslogd)
    - [Setup Logrotate](#setup-logrotate)
      - [logrodated Configuration](#logrodated-configuration)
      - [logrotated Service files](#logrotated-service-files)
  - [Configure Configuration Updates](#configure-configuration-updates)
    - [Check Scripts](#check-scripts)
    - [Setup Crontab](#setup-crontab)
  - [Configure rsyslog cluster member](#configure-rsyslog-cluster-member)
    - [Use GIT to sync between syslog servers](#use-git-to-sync-between-syslog-servers-1)
      - [Add server key to Stash](#add-server-key-to-stash-1)
      - [Setup Sync Folders](#setup-sync-folders-1)
      - [Clone Repos](#clone-repos-1)
      - [Install rsyslog](#install-rsyslog-1)
      - [Setup Syslog Config Files](#setup-syslog-config-files-1)
      - [Make service specific directories](#make-service-specific-directories-1)
      - [Enable and Start rsyslogd](#enable-and-start-rsyslogd-1)
      - [logrotated Service files](#logrotated-service-files-1)
      - [Setup Crontab](#setup-crontab-1)
  - [Install Splunk Forwarder](#install-splunk-forwarder)
    - [Download Agent](#download-agent)
    - [Confirm MD5](#confirm-md5)
    - [Install Agent - Download](#install-agent---download)
    - [Install Agent - Repo](#install-agent---repo)
    - [Create SystemD Service](#create-systemd-service)
    - [Configure Forwarder](#configure-forwarder)
    - [Enable and Start Splunk Agent](#enable-and-start-splunk-agent)

## Assumptions

- You have a fully functioning Splunk Deployment Server
- You know what inputs will be on the Syslog Server and you have the TAs ready with ```inputs.conf``` already configured
- The server class for the Syslog Servers is already set to go.
- You have the new indexes on the indexing cluster already configured

## Configure GIT Repos

Since we are going to have multiple syslog servers we are going to want an easy way to sync configurations between servers.

There are a few ways to slide it. In theory you could do everything with a single repo but since I like to keep everything separate I am going to have 4 Repos

- logrotate: These are all of the files that go in ```/etc/logrotate.d/``` to rotate syslog files so they don't fill up the hard drive.
- syslog: These are all of the files that go in ```/etc/rsyslog.d```. These are individual service files for each device / appliance that is writing to the syslog server.
- syslog_conf: This contains a copy of ```/etc/rsyslog.conf``` so that it is identical across all nodes.
- scripts: This contains any scripts that we use for backup and sync'ing.


## Configure rsyslog

### Use GIT to sync between syslog servers

#### Add server key to Stash

You will likely need to add your SSH Keys to the Git Repositories that hold:

- Scripts
- Syslog Config
- Syslog Service Config files
- Logrotate service config files

Add contents of ```/root/.ssh/id_rsa.pub```

If you are starting from scratch, create the repos and sync them from empty so the backup scripts can download/upload and changes.

#### Setup Sync Folders

Create the necessary directories

```sh
cd /opt
mkdir syslog
cd /opt/syslog
```

#### Clone Repos

You will obviously need to update the git URLS for your company

```sh
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/logrotate.git
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/syslog.git
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/syslog_conf.git
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/scripts.git
```

---

### Install rsyslog

```sh
dnf -y install rsyslog
```

#### Setup Syslog Config Files

Now that the repos have been downloaded, the server needs to be pointed to them.

Setup system specfic configurations for rsyslog

```sh
rm -rf /etc/rsyslog.d/
cd /etc
ln -s /opt/syslog/syslog
mv syslog rsyslog.d
```

Below is an example of one of the templates. Make note of the ```?JUNIPER```. You will want to match that to the directive in ```rsyslog.conf```.

```ini
:fromhost-ip, isequal, "172.16.1.199"   ?JUNIPER
& stop
:fromhost-ip, isequal, "172.16.1.198"   ?JUNIPER
& stop
:fromhost-ip, isequal, "172.16.1.218"   ?JUNIPER
& stop
:fromhost-ip, isequal, "172.16.1.219"   ?JUNIPER
& stop
```


Setup global server ```rsyslog.conf``` file

#### First time rsyslogd.conf 


```ini
# rsyslog v5 configuration file

#### MODULES ####

module(load="impstats"
       interval="60"
       severity="7"
       log.syslog="off"
       log.file="/var/log/impstats.log")

$ModLoad imuxsock # provides support for local system logging (e.g. via logger command)
$ModLoad imklog   # provides kernel logging support (previously done by rklogd)
#$ModLoad immark  # provides --MARK-- message capability

$SystemLogRateLimitInterval 0

# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514 

#### GLOBAL DIRECTIVES ####

$template JUNIPER,"/var/log/juniper/%HOSTNAME%"
$template SEC_NAS,"/var/log/nas/%HOSTNAME%"
$template FIREWALL,"/var/log/firepower/%HOSTNAME%"
$template UITS_UCS,"/var/log/ucs/%HOSTNAME%"
$template TACACS,"/var/log/tacacs/%HOSTNAME%"
$template VPN,"/var/log/vpn/%HOSTNAME%"
$template WISM,"/var/log/wireless_controllers/%HOSTNAME%"
$template SPAM,"/var/log/spam/%HOSTNAME%"
$template SVC,"/var/log/svc/%HOSTNAME%"
$template ESXI,"/var/log/esxi/%HOSTNAME%"
$template VDI,"/var/log/vdi/%HOSTNAME%"
$template TEST_DOCKER, "/var/log/docker/test/%HOSTNAME%"
$template PROD_DOCKER, "/var/log/docker/prod/%HOSTNAME%"
$template SAN, "/var/log/san/%HOSTNAME%"
$template msgonly,"%msg%\n"

$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat

$IncludeConfig /etc/rsyslog.d/*.conf
```

Run the following commands:

```sh
mv rsyslog.conf rsyslog.install
ln -s /opt/syslog/syslog_conf/rsyslog.conf
```

#### Make service specific directories

The following commands will pull all of the template directories out of the configuration file and create them.

```sh
cat /etc/rsyslog.conf |grep template| cut -f 2 -d \"| cut -f 1 -d \%|uniq >> /tmp/dirstocreate.txt
mkdir -p `cat /tmp/dirstocreate.txt`
```

#### Enable and Start rsyslogd

```sh
systemctl enable rsyslog
systemctl start rsyslog
```

---

### Setup Logrotate

#### logrodated Configuration

Edit file ```/etc/logrotate.conf```

```conf
include /opt/syslog/logrotate
```

After

```conf
include /etc/logrotate.d
```

This should make the file ```/etc/logrotate.conf``` look something like:

```conf
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
#compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d
include /opt/syslog/logrotate
# system-specific logs may be also be configured here.
```

#### logrotated Service files

First you will need to link the ```/etc/logrodate.d``` folder to the git repo that you have created in ```/opt/syslog```

```sh
mv /etc/logrodate.d /etc/logrodate.d.old
cd /etc
ln -s /opt/syslog/logrotate/ logrotate.d
```

Then you can place individual service files in that directory.

For example: ```/etc/logrodate.d/firepower```

```
/var/log/firepower/*.log {
        size 50M
        missingok
        rotate 4
        nodateext
        nocompress
        sharedscripts
        postrotate
                /usr/bin/systemctl restart rsyslog.service
        endscript
}
```

---

## Configure Configuration Updates

### Check Scripts

Make sure that the ```update.sh``` script in ```/opt/syslog/scripts``` is executable.

```sh
chmod 755 /opt/syslog/scripts/update.sh
```

If this is the first time then you will want to create the script the following scripts. Make sure you update the paths/directories to match your repo directories that were sync'ed via git. 

***restart.sh***

You may need to add more lines to reference all of the syslog servers that you are going to have.

```sh
CHANGE_DATE=`date`
echo "Update Log Rotate Scripts..."
cd /opt/syslog/logrotate/; git add -A .; git commit -m "Index Changes on $CHANGE_DATE"; git push

echo "Update Rsyslog Units..."
cd /opt/syslog/syslog; git add -A .; git commit -m "Index Changes on $CHANGE_DATE"; git push

echo "Update Main Rsyslog Config..."
cd /opt/syslog/syslog_conf; git add -A .; git commit -m "Index Changes on $CHANGE_DATE"; git push

echo "Updating Scriptss..."
cd /opt/syslog/scripts; git add -A .; git commit -m "Index Changes on $CHANGE_DATE"; git push

echo "Restarting all nodes
systemctl rsyslog.service

echo "Giving Git a chance to update.. Waiting 60 seconds."
sleep 60

# Make sure you update the line below with the proper host name and
# if you have more than 2 hosts you will need to all of them below

ssh syslog-dcw-0.ad.mycompany.com systemctl restart rsyslog.service
```

***update.sh***

```sh
#!/bin/sh

echo `date`

echo "Update Log Rotate Scripts..."
cd /opt/syslog/logrotate/; git pull

echo "Update Rsyslog Units..."
cd /opt/syslog/syslog; git pull

echo "Update Main Rsyslog Config..."
cd /opt/syslog/syslog_conf; git pull

echo "Updating Scriptss..."
cd /opt/syslog/scripts; git pull
echo `date`

```


### Setup Crontab

Setup the crontab to launch logrotate ever 30 minutes and check to updates to the configuration every minute.

```sh
crontab -e
```

```crontab
*/30 * * * * /usr/sbin/logrotate -v /etc/logrotate.conf &> /var/log/logrotate.log
```

---

## Configure rsyslog cluster member

This is for the second (and beyond) syslog server. You are going to make all updates on the first server and the other servers will download the configuration updates automatically. 

### Use GIT to sync between syslog servers

#### Add server key to Stash

You will likely need to add your SSH Keys to the Git Repositories that hold:

- Scripts
- Syslog Config
- Syslog Service Config files
- Logrotate service config files

Add contents of ```/root/.ssh/id_rsa.pub```

If you are starting from scratch, create the repos and sync them from empty so the backup scripts can download/upload and changes.

#### Setup Sync Folders

Create the necessary directories

```sh
cd /opt
mkdir syslog
cd /opt/syslog
```

#### Clone Repos

You will obviously need to update the git URLS for your company

```sh
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/logrotate.git
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/syslog.git
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/syslog_conf.git
git clone ssh://git@atl-stash.ad.mycompany.com:7999/sr/scripts.git
```

#### Install rsyslog

```sh
dnf -y install rsyslog
```

#### Setup Syslog Config Files

Now that the repos have been downloaded, the server needs to be pointed to them.

Setup system specfic configurations for rsyslog

```sh
rm -rf /etc/rsyslog.d/
cd /etc
ln -s /opt/syslog/syslog
mv syslog rsyslog.d
```

Also  the main configuration file

```sh
mv rsyslog.conf rsyslog.install
ln -s /opt/syslog/syslog_conf/rsyslog.conf
```

#### Make service specific directories

The following commands will pull all of the template directories out of the configuration file and create them.

```sh
cat /etc/rsyslog.conf |grep template| cut -f 2 -d \"| cut -f 1 -d \%|uniq >> /tmp/dirstocreate.txt
mkdir -p `cat /tmp/dirstocreate.txt`
```

#### Enable and Start rsyslogd

```sh
systemctl enable rsyslog
systemctl start rsyslog
```

#### logrotated Service files

First you will need to link the ```/etc/logrodate.d``` folder to the git repo that you have created in ```/opt/syslog```

```sh
mv /etc/logrodate.d /etc/logrodate.d.old
cd /etc
ln -s /opt/syslog/logrotate/ logrotate.d
```

#### Setup Crontab

Setup the crontab to launch logrotate ever 30 minutes and check to updates to the configuration every minute.

```sh
crontab -e
```

```crontab
*/30 * * * * /usr/sbin/logrotate -v /etc/logrotate.conf &> /var/log/logrotate.log
* * * * * /bin/bash /opt/syslog/scripts/update.sh > /var/log/configupdate.log
```

Now the all of the other syslog servers should have the exact same configuration as the master. Just remember to run the update script after every change so the other servers can get the config.

---

## Install Splunk Forwarder

I am going to make the assumption that you know how to configure the deployment server so that the UF can download all the required apps that will watch the configured rsyslog directories above. If you want a specific guide for that LMK.

### Download Agent

Download the installer and its associated MD5 file from [Splunk.com](https://www.splunk.com/en_us/download/universal-forwarder.html).

Ideally you are using something like Satellite so that you can keep the latest version of the UF that an admin can download from the link above. This way all the servers in your environment can get the latest version when they next update. 

### Confirm MD5

Make sure the rpm you have downloaded form Splunk has not been messed with.

```sh
md5sum splunkforwarder-RELEASE_VERSION-linux-2.6-x86_64.rpm | awk ' { print $1 } '; cat splunkforwarder-RELEASE_VERSION-linux-2.6-x86_64.rpm.md5 | awk ' { print $4 } '
```

### Install Agent - Download

Assuming the values are identical. Go ahead and install the agent

```sh
rpm -ivh splunkforwarder-RELEASE_VERSION-linux-2.6-x86_64.rpm
chown splunk.splunk /opt/splunkforwarder
/opt/splunkforwarder/bin/splunk enable boot-start -u splunk --accept-license --answer-yes --auto-ports --no-prompt
```

### Install Agent - Repo

Assuming the values are identical. Go ahead and install the agent

```sh
dnf -y install splunkforwarder-RELEASE_VERSION-linux-2.6-x86_64.rpm
chown splunk.splunk /opt/splunkforwarder
/opt/splunkforwarder/bin/splunk enable boot-start -u splunk --accept-license --answer-yes --auto-ports --no-prompt
```


### Create SystemD Service

Make sure that the Splunk Forwarder service can be started by systemd.

```sh
if [ $SYSTEM_RAM -lt 5 ] ; then
        SYSTEM_RAM=`free -g|grep Mem | awk '{ print $2 "G" }'`
else
        SYSTEM_RAM=`free -g|grep Mem | awk '{ print $2 - 4 "G" }'`
fi

CPU_WEIGHT=1024

echo "#This unit file replaces the traditional start-up script for systemd
#configurations, and is used when enabling boot-start for Splunk on
#systemd-based Linux distributions.

[Unit]
Description=Systemd service file for Splunk, generated by 'splunk enable boot-start'
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Restart=always
ExecStart=/opt/splunkforwarder/bin/splunk _internal_launch_under_systemd
KillMode=mixed
KillSignal=SIGINT
TimeoutStopSec=360
LimitNOFILE=65536
LimitRTPRIO=99
SuccessExitStatus=51 52
RestartPreventExitStatus=51
RestartForceExitStatus=52
User=splunk
Group=splunk
NoNewPrivileges=yes
AmbientCapabilities=CAP_DAC_READ_SEARCH
ExecStartPre=-/bin/bash -c \"chown -R splunk:splunk /opt/splunkforwarder\"
Delegate=true
MemoryMax=$SYSTEM_RAM
CPUWeight=$CPU_WEIGHT
User=splunk

PermissionsStartOnly=true
ExecStartPost=-/bin/bash -c \"chown -R splunk:splunk /sys/fs/cgroup/cpu/system.slice/%n\"
ExecStartPost=-/bin/bash -c \"chown -R splunk:splunk /sys/fs/cgroup/memory/system.slice/%n\"

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/SplunkForwarder.service

chown splunk.splunk /opt/splunkforwarder/* -R
```

### Configure Forwarder

Create the ```deploymentclient.conf``` file so the agent knows where to get its license and deployment files.

```sh
echo "[deployment-client]
[target-broker:deploymentServer]
targetUri = splunkdeploy.ad.mycompany.com:8089" > /opt/splunkforwarder/etc/system/local/deploymentclient.conf
```

### Enable and Start Splunk Agent

Because you have just create the ```/etc/systemd/system/SplunkForwarder.service``` you need to tell systemd to re-scan its directories for services.

Now enable and start the service by running the following commands:

```sh
systemctl daemon-reload
systemctl enable SplunkForwarder.service
systemctl start SplunkForwarder.service
```

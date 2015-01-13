---
layout: post
title: "SFTP Server - Proftpd Setup Part1"
excerpt: "sftp server using proftpd part1"
tags: [sftp, proftpd, server, ftp, linux, centos, xferlog, ssh, virtual users ]
comments: true
share: true
---

##SFTP server setup using Proftpd server with xferlog

Lately I have been tasked with setting up a new sftp server, due to openssh's inability to log all file transactions properly. so we explored all the options available and settled with the proftpd as that seems the most popular one and has a widely accepted sftp user base. Proftpd has a learning curve,like every other piece of software so please bare with me with this long article.

  <br />
NOTE: Special Requirement Setup for an example client named RMP will be discussed on part2 of this article so anything refering to rmp in the config is to address that requirement.

##Our Requirements:
 - Multiple Nested folder Access control for specific users ( jailed home folders / shared folder access/ read only access / hide folders which users don't have permission etc..)
 - Key / Password based authentication
 - xferlog log format for file transactions
 - Simple to setup and Maintain
 - Use only virtual users and not system accounts

##Installation:
We are going to use Centos6.6 Server patched to the latest update & Proftpd (version 1.3.5-4.0) avialable at the time of writing this article. To keep the installation simple and consistent we are going to grab the rpms from [city-fan repository](http://www.city-fan.org/ftp/contrib/yum-repo/rhel6/x86_64/) (proftpd-1.3.5-4.0.cf.rhel6.x86_64.rpm, proftpd-utils-1.3.5-4.0.cf.rhel6.x86_64.rpm) as the rpms are not available in epel repos. 

<br />

####Install the packages:
{% highlight bash %}
yum -y  localinstall  proftpd-1.3.5-4.0.cf.rhel6.x86_64.rpm \
		      proftpd-utils-1.3.5-4.0.cf.rhel6.x86_64.rpm
{% endhighlight %}

##Configuration:
we are going to run the proftpd server with user 'proftpd' and group 'ftpgroup'. so please add the relevant user/group.
{% highlight bash %}
#add group - choose a high number for groupid/userid
groupadd -g 2001 ftpgroup

#add user
useradd -g ftpgroup -u 2001 -C "Proftpd System Account" -r proftpd
{% endhighlight %}

####Config Files/Directory Structure: 
(Note: We have a special requirement of a shared folder setup for example client RMP which will be discussed in the part2 section of this article)

{% highlight bash %}
#Main config file:
[root@server02 ~]# ls -lh /etc/proftpd.conf
-rw-r-----. 1 root root 11K Jan  8 15:41 /etc/proftpd.conf

#Backup original config file before editing for reference
[root@server02 ~]# cp -v /etc/proftpd.conf{,.original}

#Folder Layout and corresponding permissions
#please create the folders/files with the corresponding permissions.
[root@server02 ~]# ls -lh /etc/proftpd
total 16K
drwx------. 2 proftpd ftpgroup 4.0K Dec 18 16:31 authorized_keys
drwxr-x---. 2 proftpd ftpgroup 4.0K Jan  8 11:50 clients
-r--------. 1 proftpd ftpgroup  151 Dec 22 16:26 sftpd.group
-r--------. 1 proftpd ftpgroup 1.1K Dec 24 12:33 sftpd.passwd

#Note: sftpd.passwd, sftpd.group will be created when virtual users/groups are added below.

#Key Based Authentication keys folder
[root@server02 ~]# ls -lh /etc/proftpd/authorized_keys/
total 8.0K
-rw-r--r--. 1 root root 468 Dec 18 15:31 user2
-rw-r--r--. 1 root root 468 Dec 18 16:31 user3

#Modular clients ACL Requirements config folder
[root@server02 ~]# ls -lh /etc/proftpd/clients/
total 4.0K
-rwxr-x---. 1 proftpd ftpgroup 993 Jan  8 11:50 rmp.conf

{% endhighlight %}

Update the config file as per the following link. [proftpd.conf](https://gist.githubusercontent.com/tuxfight3r/b62dc3351732615f9e86/raw/proftpd.conf). please go through the config file as most of the bits are self explanatory. <br />
{% highlight bash %}
#Jailed users/Chroot home is achieved by the below line in proftpd.conf

#Everyone Else mapped to thier home directory
DefaultRoot        ~ !adm
 
{% endhighlight %}

####Enable the mod_ban for blocking repeated failed logins.

{% highlight bash %}
#NOTE: To optimize mod_ban check the mod_ban section in proftpd.conf

[root@server02 ~]# cat /etc/sysconfig/proftpd
# Set PROFTPD_OPTIONS to add command-line options for proftpd.
# See proftpd(8) for a comprehensive list of what can be used.
#
# The following "Defines" can be used with the default configuration file:
# -DANONYMOUS_FTP       : Enable anonymous FTP
# -DDYNAMIC_BAN_LISTS   : Enable dynamic ban lists (mod_ban)
# -DQOS                 : Enable QoS bits on server traffic (mod_qos)
# -DTLS                 : Enable TLS (mod_tls)
#
# For example, for anonymous FTP and dynamic ban list support:
# PROFTPD_OPTIONS="-DANONYMOUS_FTP -DDYNAMIC_BAN_LISTS"
PROFTPD_OPTIONS="-DDYNAMIC_BAN_LISTS"
{% endhighlight %}

####Update sftpasswd script to reflect your setup to manage sftp virtual users

{% highlight bash %}
#copy the existing ftpasswd script as sftpasswd
cp /usr/bin/ftpasswd /usr/bin/sftpasswd

#Edit the file and update the code as below.
vi /usr/bin/sftpasswd

#replace from line 37 from the below:
my $default_passwd_file = "./ftpd.passwd";
my $default_group_file = "./ftpd.group";

#replace to
my $base_folder = "/etc/proftpd";
my $default_passwd_file = $base_folder."/sftpd.passwd";
my $default_group_file = $base_folder."/sftpd.group";

#Verify Syntax of that script
[root@server02 bin]# perl -c /usr/bin/sftpasswd
/usr/bin/sftpasswd syntax OK

{% endhighlight %}

####Verify Configs and start Service

{% highlight bash %}
#Check Syntax
[root@server02 bin]# proftpd -t
Checking syntax of configuration file
2015-01-12 23:39:14,269 server02.test.local proftpd[9182]: processing configuration directory '/etc/proftpd/clients'
Syntax check complete.

#start the service
[root@server02 bin]# service proftpd start
Starting proftpd:                                          [  OK  ]

#Enable auto start
[root@server02 bin]# chkconfig proftpd on

{% endhighlight %}

####Create Virtual SFTP Users/Group

{% highlight bash %}
#Add Virtual Group - will create sftpd.group file if it didnt exist
sftpasswd --group --gid=2001 --name=sftpgroup

#Add Virtual Users - will prompt for password for each user separately - will create sftpd.passwd file if it didnt exist
sftpasswd --passwd --name=user1 --home=/sftp/home/user1 --shell=/bin/false --uid=2001 --gid=2001

sftpasswd --passwd --name=user2 --home=/sftp/home/user2 --shell=/bin/false --uid=2001 --gid=2001

sftpasswd --passwd --name=user3 --home=/sftp/home/user3 --shell=/bin/false --uid=2001 --gid=2001

#check if /etc/proftpd/sftpd.{group,passwd} are updated properly. Note: system user/group id doesnt have to match virtual user/group id.

[root@server02 proftpd]# cat /etc/proftpd/sftpd.group
ftpgroup:x:2001:user1,user2,user3

[root@server02 proftpd]# cat /etc/proftpd/sftpd.passwd
user1:$1$sFbAFURA$W6ySumztKRZXQ1Cn7Xg5y.:2001:2001::/sftp/home/user1:/bin/bash
user2:$1$Vhafg9lR$EHP.S79aeXRRlvwMb5MVM1:2001:2001::/sftp/home/user2:/bin/false
user3:$1$oNjFuN8M$TOgga.Gl.2o2tqrxMNr2g1:2001:2001::/sftp/home/user3:/bin/false

#Clients Home Folder Layouts (create the folders as below and set it to be owned by system proftpd.ftpgroup)
[root@server02 ~]# ls -lh /sftp/home/
drwxr-xr-x. 2 proftpd ftpgroup 4.0K Dec 17 18:10 user1
drwxr-xr-x. 2 proftpd ftpgroup 4.0K Dec 17 18:11 user2
drwxr-xr-x. 2 proftpd ftpgroup 4.0K Dec 18 16:59 user3

#Reload the sftp server
[root@server02 bin]# service proftpd restart
Shutting down proftpd:                                     [  OK  ]
Starting proftpd:                                          [  OK  ]

{% endhighlight %}

The above setup will provide you a sftp server running on port 2022 with password based authentication, for which ever client if you prefer to enable key based authentication please drop the users key in /etc/proftpd/authorized_keys/ folder with <client_username> as file name

####Key Based Authentication:

{% highlight bash %}
#Generate ssh keys 
[root@server02 .ssh]# ssh-keygen -b 2048
# it should create a file under .ssh as id_rsa.pub

#convert the key from openssh format to SSH2 format.
[root@server02 .ssh]# ssh-keygen -e -f id_rsa.pub > user2

#please makesure that the comment is not too longer than the key as it may cause issues sometimes
[root@server02 .ssh]# cat user2
---- BEGIN SSH2 PUBLIC KEY ----
Comment: "2048-bit RSA, root@server02.test.local"
AAAAB3NzaC1yc2EAAAABIwAAAQEAqgt6I0nsIqEEXQ9GKRvUp9AuNHuWXjCWM7pUeXfq0+
axidUqkSjiKKjJ1zh5i0oIo208AYcbLieWzRsSL4hOB8a1kYCq2foTokTuAVbowP3r9FVh
WZnQGvsu16DztuWP9CAF04DxCNp2Tzko6PVx8f6qt4HNS6lKRAn31oakROiBKM0807mGIj
rB/25LHikYHr7ifSazd0zEc+t7Tx1k1G3+7v1Stwch8W3oOsTzc6u1IfWo1zML2q3LUxIm
odGrwsjEMV5jS5Aq8ouwqKLbflwwpk2aliXj3SrG3LH4yyaRvogGGjIjWKsR/qOKcOqNPy
4exQ8iDos2/3zIX35xoQ==
---- END SSH2 PUBLIC KEY ----
{% endhighlight %}

Drop the above key in `/etc/proftpd/authorized_keys/` folder as user2 and user2 should now be able to login using  key based authentication. As this article is already too long special requirements are to be discussed in the part2 of this article.

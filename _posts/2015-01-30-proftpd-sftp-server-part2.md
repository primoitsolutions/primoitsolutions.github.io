---
layout: post
title: "SFTP Server - Proftpd Setup Part2"
excerpt: "proftpd sftp server shared folder/read only folder ACL setup"
tags: [sftp, proftpd, server, ftp, linux, centos, xferlog, ssh, virtual users ]
comments: true
share: true
---

##SFTP server - shared folder access with restricted users

If you have followed [part1 of this
article](http://www.primoitsolutions.uk/proftpd-sftp-server-part1/) you would have seen me mentioning about a virtual client named RMP. we are going to discuss that requirement here and look into how we can set it up.

##Our Requirements:
 - 4 Folders [ Data, Documents, Test, Logs ] exist in the top folder /sftp/home/rmp_inbound 
 - 4 internal users grouped as rmp_intgrp and they will have full access on the above mentioned folders
 - 3 External users grouped as rmp_extgrp and they will have read access to the folder Data and Documents only and Test/Logs folder should be invisible
 - xferlog log format for file transactions and everyday log is kept under Logs folder for internal users access
 - Key based authentication

##Setup

####Create the folders
{% highlight bash %}
mkdir -pv /sftp/home/rmp_inbound/{Data,Documents,Test,Logs}
chown proftpd.ftpgroup -R /sftp/home/rmp_inbound
chmod 770 -R /sftp/home/rmp_inbound

[root@server02 rmp_inbound]# pwd
/sftp/home/rmp_inbound

[root@server02 rmp_inbound]# ls -1
Data
Documents
Logs
Test

{% endhighlight %}

####Create Groups/Users:
We are going to create 2 groups and add the 7 users to thier respective groups as mentioned in our requirement

{% highlight bash %}
#Add Virtual Group rmp_intgrp for internal users
sftpasswd --group --gid=2002 --name=rmp_intgrp

#Add Virtual Group rmp_extgrp for external users
sftpasswd --group --gid=2003 --name=rmp_extgrp

#Add Internal users and add them to Internal group - will prompt for password.
sftpasswd --passwd --name=rmp_intuser1 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2002 --gecos "RMP Internal - User1"
sftpasswd --passwd --name=rmp_intuser2 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2002 --gecos "RMP Internal - User2"
sftpasswd --passwd --name=rmp_intuser3 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2002 --gecos "RMP Internal - User3"
sftpasswd --passwd --name=rmp_intuser4 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2002 --gecos "RMP Internal - User4"

NOTE: we have reused the userid of proftpd daemon mentioned in part1 as they are virtual users and proftpd can have full access to them when it comes to unix permissions

#Add External users and add them to External group - will prompt for password.
sftpasswd --passwd --name=rmp_extuser1 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2003 --gecos "RMP External - User1"
sftpasswd --passwd --name=rmp_extuser2 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2003 --gecos "RMP External - User2"
sftpasswd --passwd --name=rmp_extuser3 --home=/sftp/home/rmp_inbound --shell=/bin/false --uid=2001 --gid=2003 --gecos "RMP External - User3"

NOTE: All the above 7 users share the same home folder.

#we should see something similar in /etc/proftpd/sftp.group

[root@server02 proftpd]# cat /etc/proftpd/sftpd.group
ftpgroup:x:2001:user1,user2,user3
rmp_intgrp:x:2002:rmp_intuser1,rmp_intuser2,rmp_intuser3,rmp_intuser4
rmp_extgrp:x:2003:rmp_extuser1,rmp_extuser2,rmp_extuser3

#we should see something similar in /etc/proftpd/sftp.passwd
[root@server02 proftpd]# cat /etc/proftpd/sftpd.passwd
user1:$1$sFbAFURA$W6ySumztKRZXQ1Cn7Xg5y.:2001:2001::/sftp/home/user1:/bin/bash
user2:$1$Vhafg9lR$EHP.S79aeXRRlvwMb5MVM1:2001:2001::/sftp/home/user2:/bin/false
user3:$1$oNjFuN8M$TOgga.Gl.2o2tqrxMNr2g1:2001:2001::/sftp/home/user3:/bin/false
rmp_intuser1:$1$oNjFuN8M$TOgga.Gl.2o2tqrxMNr2g1:2001:2002:RMP Internal - User1:/sftp/home/rmp_inbound:/bin/false
rmp_intuser2:$1$oNjFuN8M$TOgga.Gl.3o2tqrxMNr2g2:2001:2002:RMP Internal - User2:/sftp/home/rmp_inbound:/bin/false
rmp_intuser3:$1$oNjFuN8M$TOgga.Gl.4o2tqrxMNr2g3:2001:2002:RMP Internal - User3:/sftp/home/rmp_inbound:/bin/false
rmp_intuser4:$1$oNjFuN8M$TOgga.Gl.2o5tqrxMNr2g4:2001:2002:RMP Internal - User4:/sftp/home/rmp_inbound:/bin/false
rmp_extuser1:$1$oNjFuN8M$TOgga.Gl.2o2tqrxMNr4g1:2001:2003:RMP External - User1:/sftp/home/rmp_inbound:/bin/false
rmp_extuser2:$1$oNjFuN8M$TOgga.Gl.2o2tqrxMNr6g2:2001:2003:RMP External - User2:/sftp/home/rmp_inbound:/bin/false
rmp_extuser3:$1$oNjFuN8M$TOgga.Gl.2o2tqrxMNr8g3:2001:2003:RMP External - User3:/sftp/home/rmp_inbound:/bin/false

{% endhighlight %}

##Configuration:
If you are following the article in part1 [proftpd.conf](https://gist.githubusercontent.com/tuxfight3r/b62dc3351732615f9e86/raw/proftpd.conf) you would see that there is a reference to home folder configuration which jails the users to thier home folder.

####Jailed Home Folders
please take a look at the above link to look at the full config file, but the below bits are the important ones for this shared folder setup.
{% highlight bash %}
#Main config file:
[root@server02 ~]# ls -lh /etc/proftpd.conf
-rw-r-----. 1 root root 11K Jan  8 15:41 /etc/proftpd.conf

#NOTE: The below config should be placed above the default configuration as shown below
#Jailed users/Chroot home is achieved by the below line in proftpd.conf

#RMP HOME folder configuration
DefaultRoot        /sftp/home/rmp_inbound rmp_intgrp,rmp_extgrp

#Everyone Else mapped to thier home directory
DefaultRoot        ~ !adm


#LOAD CLIENT FOLDER PERMISSION FROM EXTERNAL CONF FILES
Include /etc/proftpd/clients/\*.conf

{% endhighlight %}


####Shared Folders Special ACL Enforcement.
If this little snippet is not included, both the internal/external users will have admin access
{% highlight bash %}
[root@server02 clients]# pwd
/etc/proftpd/clients

[root@server02 clients]# ls
rmp.conf

[root@server02 clients]# cat rmp.conf
{% endhighlight %}
{% gist tuxfight3r/f4945395605b25177789 %}

####Verify Configs and restart Service

{% highlight bash %}

#Check Syntax
[root@server02 bin]# proftpd -t
Checking syntax of configuration file
2015-01-12 23:39:14,269 server02.test.local proftpd[9182]: processing configuration directory '/etc/proftpd/clients'
Syntax check complete.

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
[root@server02 .ssh]# ssh-keygen -e -f id_rsa.pub > rmp_intuser1

#please makesure that the comment is not too longer than the key as it may cause issues sometimes
[root@server02 .ssh]# cat rmp_intuser1
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

Drop the above key in `/etc/proftpd/authorized_keys/` folder as rmp_intuser1 and rmp_intuser1 should now be able to login using  key based authentication. Drop the keys for other users to enable login via key based authentication

NOTE: The above key generation is shown for example only and the clients should provide thier public key themselves

##Logging

####xferlog daily log requirement.
Since the logrotate for the whole sftp server logs is rotated weekly, we are going to use a script to give us daily logs just for the RMP client. The script can be placed under /opt/scripts and run daily at 01:00 hours
{% gist tuxfight3r/0f11e936b4cfe1e7e45b %}

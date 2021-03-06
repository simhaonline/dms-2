.. _Debian-Install:

**************
Debian Install
**************

DNS Management System
=====================

The DNS Management System (DMS) is designed to have a master/replica master
setup. It is a complex setup, requiring the hand configuration of database, DNS
server, and other components. 

For the whole DMS DR shebang, do::

 apt-get install --install-recommends --install-suggested dms

Single Master Server
====================

If your setup does not require one of the components such as ``quagga``,
``dms-wsgi``, or ``etckeeper``, just skip that section.

Just install the ``dms-core`` package::

 apt-get install --install-recommends --install-suggested dms-core

At least do ``--install-recomends`` above, as this will pull in IPSEC to
protect the connections between the master and slave servers.  This is far more
secure than relying on TSIG MD5 HMAC for tamper proofing.

DNS Topology
============

The description below will set up a Master DR setup with hidden master DNS.
Other topologies are possible, which are useful for smaller environments.  You
can run without DR, and the Master server can also be 'unhidden' on the
Internet itself.  Just take what you need from below.

Documentation
=============

The Documentation for DMS is available from the `DMS Web Site <http://dms.anathoth.net>`

Supporting Software Setup Details
=================================

Vim
---

Create the file ``/etc/vim/vimrc.local`` and add the following to it to enable DNS
syntax highlighting etc::

 "Turn on syntax highlighting for DNS etc
 filetype plugin indent on
 syntax on

Repeat on DR master server as well.

Less
----

Set the following environment variable in ``/etc/profile``, or ``/etc/bashrc``::

 LESS=-MMRi

This will display the colorized output produced by ``colordiff``.

Repeat on DR master server as well.

Etckeeper and Ssh
-----------------

``Etckeeper`` is a tool to keep the contents of ``/etc`` in a git VCS.  When
combined with ssh and the appropriate git remote setup with ``cron``, it allows
the ``/etc`` of the other machine in the master/replica DR pair to be kept on
its alternate, and vice-versa.  

The following is a safeguard against Murphy's law (ie Back hoe fade, backups
required at disconnected data center or off-line storage 2 days away). My own
experience is my witness.  This protects against the ``/etc`` on the master
being updated, the replica being missed, and then finding that things aren't
working on the replica when the master dies, with no record of the updates
needed to machine configuration. 

For information on ``etckeeper`` usage, see ``/usr/share/doc/etckeeper/README.gz``

Example for diffing/checking out ``/etc/racoon/racoon-tool.conf`` from other
machine::

 dms-master1:/etc# git diff dms-master2/master racoon/racoon-tool.conf
 dms-master1:/etc# git checkout dms-master2/master racoon/racoon-tool.conf
 dms-master1:/etc# git checkout HEAD racoon/racoon-tool.conf

Etckeeper installation
^^^^^^^^^^^^^^^^^^^^^^

Before installing etckeeper, you need to add a
``.gitignore`` to ``/etc`` to prevent ``/etc/shadow`` and other secrets files from ending
up in ``etckeeper`` for security reasons. The contents of the seed ``/etc/.gitignore``
file is::

 # Ignore these files for security reasons
 krb5.keytab
 shadow
 shadow-
 racoon/psk.txt
 ipsec.secrets
 ssl/
 ssh/moduli
 ssh/ssh_host_*

You probably have to purge ``etckeeper`` removing the initial ``/etc`` git archive if
you are reading this, create the ``.gitignore`` file, and reinstall ``etckeeper``::

 # dpkg --purge etckeeper
 # vi /etc/.gitignore
 # aptitude install etckeeper

Now would be a good time to install DMS on both Master and Replica::

 # apt-get install --install-recommends --install-suggested dms

as this will install scripts needed for ssh configuration following.

Set up ssh
^^^^^^^^^^

As ``root`` on both boxes, turn off the following settings in
``sshd_config``::

 RSAAuthentication no
 PubkeyAuthentication no
 RhostsRSAAuthentication no
 HostbasedAuthentication no
 ChallengeResponseAuthentication no
 PasswordAuthentication no
 GSSAPIAuthentication no
 X11Forwarding no
 UsePAM no

Then add the following to ``/etc/ssh/sshd_config``, and adjust your network
and administrative ``sshd`` authentication settings::

 UsePAM no
 AllowTcpForwarding no
 AllowAgentForwarding no
 X11Forwarding no
 PermitTunnel no
 AllowGroups sudo root
 # Section for DMS master/replica servers
 Match Address 2001:db8:f012:2::3/128,2001:db8:ba69::3/128
         PubkeyAuthentication yes
         # PermitRootLogin forced-commands-only
         # The above only works with commands given in authorized_keys
         PermitRootLogin without-password
         ForceCommand /usr/sbin/etckeeper_git_shell
 # Section for administrative access
 Match Address 2001:db8:ba69::/48,192.0.2.0/24,201.0.113.2/32
         PermitRootLogin yes
         GSSAPIAuthentication yes
         PubkeyAuthentication yes
         MaxAuthTries 10
         X11Forwarding yes
         AllowTcpForwarding yes
         AllowAgentForwarding yes

Reload ``sshd`` on both servers::

 # service ssh reload

Create a password-less  ``ssh`` key on both servers as ``root``, and copy the public
part of the key to ``/root/.ssh/authorized_keys``::

 # mkdir /root/.ssh
 # ssh-keygen -f /root/.ssh/id_gitserve_rsa -t rsa -q -N ''
 # vi /root/.ssh/config

and set contents of ``ssh`` ``config`` as follows, changing Host as appropriate::

 Host dms3-dr*
 IdentityFile ~/.ssh/id_gitserve_rsa

It is also a good idea to set up a ``/etc/hosts`` file entries on each server. 

Set up ``/root/.ssh/authorized_keys``::

 # mkdir /root/.ssh
 # cat - > /root/.ssh/authorized_keys

Cut and paste ``/root/.ssh/id_gitserve_rsa.pub`` from other machine into above,
finishing with ^D.  Then do vice-versa, to make the other direction
functional.

Check that things work on both hosts::

  # ssh -l root dms-master2
  Rejected
  Connection to dms-master2 closed.

etc.

Note: Stopping ssh and running sshd from the command-line ``/usr/sbin/sshd -d`` on
one, and then using ``ssh -vl root`` on the other (and vice versa) is very useful
for connection debugging. 

Git remote set up
^^^^^^^^^^^^^^^^^

To pair up ``/etc`` archives, as root do::

 dms-master1# git remote add dms-master2 ssh://dms-master2.someorg.net/etc

and vice versa

Check that both work by executing::

 dms-master1:/etc# git fetch --quiet dms-master2

and vice versa

Set up crond
^^^^^^^^^^^^

Edit the file ``/etc/cron.d/dms-core``, uncomment the line for git fetch, and set
the remote name::

 # Replicate etckeeper archive every 4 hours
 7 */4 * * * root  cd /etc && /usr/bin/git fetch --quiet dms-master2

Do test each cron command by running it from the root command line.

.. _IPSEC-set-up:

IPSEC set up
------------

The DMS system uses IPSEC to authenticate server access to the master servers,
encrypting and/or integrity protecting the outgoing zone transfers, ``rndc`` and
configuration ``rsync`` traffic.

Each server has IPSEC configured and active to both the replica servers (master
and DR).  The master and replica have IPSEC configured as well.  Both replica
servers and 2 slaves should be PSK keyed with each other if DNSSEC
authentication is to be used for the majority of slaves.  This ensures that the
DNSSEC CERT records can be propagated for use.

Make SURE each individual IPSEC connection has a unique PSK key for security.
They can be generated easily, and cut/paste over terminal root session, so no
big loss if they are lost. Just make sure you have 'out-of-band' access via ssh.

Read through the Strongswan section as it has some useful tips on PSK
generation and other matters.

Sysctl IPSEC settings
^^^^^^^^^^^^^^^^^^^^^

To prevent network problems with running out of buffers, create the file
``/etc/sysctl.d/30-dms-core-net.conf`` with the following contents::


 # Tune kernel for heavy IPSEC DNS work.
 # Up the max connection buffers
 net.core.somaxconn=8192
 net.core.netdev_max_backlog=8192
 # Reduce TCP final timeout
 net.ipv4.tcp_fin_timeout=10
 # Increase size of xfrm tables
 net.ipv6.xfrm6_gc_thresh=16384
 net.ipv4.xfrm4_gc_thresh=16384

and then reload sysctls with::

 # service procps start

Strongswan IPSEC set up
^^^^^^^^^^^^^^^^^^^^^^^

This is only covering basic PSK set up.  If X509 needed see the 
`Strongswan wiki <http://wiki.strongswan.org/projects/strongswan/wiki/UserDocumentation>`

The same PSK has to be at each end of the IPSEC 'connection'.

Generate PSK key with ``openssl``::

 # openssl rand -hex 64

and place in ``/etc/ipsec.secrets``::

 2001:db8:345:678:2::beef : PSK 0xe788749d48c0a020bc26b15685ad7ea1630c090072acf3f1eeac14dfec90bd4c1ff86fbf82b219cb5c309c3c6ede2d072784823a69271eccce166421317be006

Note format: ``IPv6/IPv4/DNS-type-id : PSK 0xdaedbefdeadbeef`` ...

``Racoon`` also takes hex strings as PSK, just add the '0x' to the random number.

Sha1 and sha256 only use 64 bytes (512 bits) for the key.  Sha384 and better
128 bytes.  Making the strings longer does not make sense, and can result in
some wacky behaviour with Strongswan!

Set up /etc/ipsec.conf at each end::

 conn %default
        ikelifetime=60m
        keylife=20m
        rekeymargin=3m
        keyingtries=1
        keyexchange=ikev2
        mobike=no
        installpolicy=yes


 conn dms-master2
        authby=secret
        right=2001:db8:f012:2::2
        rightid=dms-master2.someorg.net
        left=2001:db8:f012:2::3
        leftid=dms-master1.someorg.net
        type=transport
	#ah=sha256-modp2048,sha1-modp1024
        auto=route

and vice versa.  Note use of id statements.  It saves having to bury IP numbers 
in more than one place.  

``auto=route`` sets up SPD (use ip xfrm policy to inspect), and when dynamically
bring up the connection when needed.

AH (authentication header) can be turned on by defining AH protocol at each
end.  This is useful inside DMZ or back end networks, and allows the traffic to
be inspected by a decent filtering firewall.

Reload ``ipsec`` by::

 dms-master1 # ipsec reload
 dms-master1 # ipsec rereadsecrets
 dms4-master # ipsec reload
 dms4-master # ipsec rereadsecrets

Enter a separate PSK in ``/etc/ipsec.secrets`` for each IPSEC connection.

Useful ``ipsec`` commands are::

 # ipsec status
 # ipsec statusall
 # ip xfrm policy
 # ip xfrm state
 # ipsec up <connection name>
 # ipsec down <connection name>
 # ipsec reload
 # ipsec rereadsecrets
 # ipsec restart.

Test the connection by pinging the far end - tests unencrypted reachability, 
and then ``telnet``/``netcat`` the different TCP ports used across the link.  This
will involve ports 873 (rsync),  953 (rndc/named), 53 (named) to each slave,
and port 53 on the masters (from slave).  Between both the replica servers
(master and DR), port 5432 (postgresql) has to be reachable, as well as port 22
(ssh).  Port 80 (http) for apt-get updates may also be involved.

Racoon IPSEC set up
^^^^^^^^^^^^^^^^^^^

An alternative to strongswan is to use ``racoon``.  This might be a better solution
if you are working with a lot of NetBSD or FreeBSD based systems.

This is only covering basic PSK set up.  For X509 etc, see
``/usr/share/doc/racoon/README.Debian``

On each machine, ``dpkg-reconfigure racoon``, and choose the "racoon-tool"
configuration method.  Edit ``/etc/racoon/racoon-tool.conf``, and add the machines
source IP address::

 connection(%default):
        src_ip: 192.168.102.2
        admin_status: disabled

Add the other replica server and each DNS as a separate configuration fragment
in ``/etc/racoon/racoon-tool.conf.d``, named after the machine's short hostname::

 peer(192.168.102.2):

 connection(dms-master2-eth1):
         dst_ip: 192.168.102.2
  	# defaults to esp
         # encap: ah
         admin_status: enabled

For the replica servers, if you want to inspect/control traffic select AH IPSEC
encapsulation.  Note, ``racoon-tool`` sets up a transport mode IPSEC connection if
no ``src_range``/``dst_range`` parameters are given. 

For ``racoon-tool only``, transport mode used to not encrypt ICMP traffic, as that
can complicate UDP/TCP connection issues extensively.  This will be changed
very shortly to conventionally encrypting IPSEC to be compatible with other
IPSEC solutions.

Also enter a separate PSK in ``/etc/racoon/psk.txt`` for each IPSEC connection.

Useful racoon-tool commands are::

 # racoon-tool vlist
 # racoon-tool spddump
 # racoon-tool saddump
 # racoon-tool vup <connection name>
 # racoon-tool vdown <connection name>
 # racoon-tool reload
 # racoon-tool restart.

Test the connection by pinging the far end - tests unencrypted reachability, 
and then telnet/netcat the different TCP ports used across the link.  This
will involve ports 873 (rsync),  953 (rndc/named), 53 (named) to each slave,
and port 53 on the masters (from slave).  Between both the replica servers
(master and DR), port 5432 (postgresql) has to be reachable, as well as port 22
(ssh).  Port 80 (http) for apt-get updates may also be involved.

Firewalling on IPSEC links to Master Servers
--------------------------------------------

The Master servers need protection on the IPSEC connections from the slave
servers, and each other as the SPD does not have any sense of connection
direction, and it is possible to connect to all the services on the Master
Servers.

The ``netscript-ipfilter`` package can save the iptables/ip6tables filters that
you create.

Use the policy match module to match decrypted traffic coming from the IPSEC
connection

An example ``ip6tables`` output::

 shalom-ext: -root- [/tmp/zones] 
 # ip6tables -vnL INPUT
 Chain INPUT (policy ACCEPT 472K packets, 134M bytes)
  pkts bytes target     prot opt in     out     source               destination         
     0     0 REJECT     all      *      *       fd14:828:ba69:2::3   ::/0                 reject-with icmp6-port-unreachable
  157K   20M ipsec-in   all      *      *       ::/0                 ::/0                 policy match dir in pol ipsec
 
 shalom-ext: -root- [/tmp/zones] 
 # ip6tables -vnL ipsec-in
 Chain ipsec-in (1 references)
  pkts bytes target     prot opt in     out     source               destination         
  138K   16M ACCEPT     all      *      *       ::/0                 ::/0                 ctstate RELATED,ESTABLISHED
     0     0 ACCEPT     udp      *      *       ::/0                 ::/0                 udp spt:500 dpt:500
     0     0 icmphost   icmpv6    *      *       ::/0                 ::/0                
   198 18580 ACCEPT     udp      *      *       ::/0                 ::/0                 ctstate NEW udp dpt:53
 17474 3629K ACCEPT     udp      *      *       ::/0                 ::/0                 ctstate NEW udp dpt:514
   118  9440 ACCEPT     tcp      *      *       ::/0                 ::/0                 ctstate NEW tcp dpt:53
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 ctstate NEW tcp dpt:953
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 ctstate NEW tcp dpt:5432
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 ctstate NEW tcp dpt:5433
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 ctstate NEW tcp dpt:873
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 ctstate NEW tcp dpt:22
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 ctstate NEW tcp dpt:113
     0     0 ACCEPT     tcp      *      *       2001:470:f012:2::3   ::/0                 tcp dpt:80 ctstate NEW
     0     0            tcp      *      *       2001:470:f012:2::3   ::/0                 tcp dpt:80 ctstate NEW
     0     0 ACCEPT     tcp      *      *       fd14:828:ba69:1:21c:f0ff:fefa:f3c0  ::/0                 ctstate NEW tcp dpt:80
   128 10240 ACCEPT     tcp      *      *       fd14:828:ba69:1:21c:f0ff:fefa:f3c0  ::/0                 ctstate NEW tcp dpt:22
     0     0 ACCEPT     tcp      *      *       2001:470:c:110e::2   ::/0                 ctstate NEW tcp dpt:80
     0     0 ACCEPT     tcp      *      *       2001:470:66:23::2    ::/0                 ctstate NEW tcp dpt:80
   607 43704 log        all      *      *       ::/0                 ::/0                
 
 shalom-ext: -root- [/tmp/zones] 

The ``icmphost`` and ``log`` chains are created by using ``netscript ip6filter exec log``
and ``netscript ip6filter exec icmphost``.  IPv6 helper chains created from RFC
4890 - 'Recommendations for Filtering ICMPv6 Messages in Firewalls'

Read ``/etc/netscript/network.conf``, and the manpage ``netscript``

The useful commands are::
 
 netscript ipfilter/ip6filter reload 
 netscript ipfilter/ip6filter save
 netscript ipfilter/ip6filter exec icmphost (create an incoming ICMP filter for host traffic) 
 netscript ipfilter/ip6filter usebackup <number>

PostgresQL DB Setup and Master/Replica Configuration
----------------------------------------------------

DB user and DB creation only has to happen on the initial master server, as it
will be 'mirrored' to the replica once DB replication is established.  The
replica server will configured to run in 'hot-standby' mode so that we can 
verify mirroring by read-only means using zone_tool.

Though the master and replica can run the PGSQL ``dms`` cluster on port 5433 or
other port, it is recommended to swap the ports with the main cluster, and
revert the main cluster to manual start up.

Edit ``postgresql.conf`` in ``/etc/postgresql/9.3/main`` and ``/etc/postgresql/9.3/dms``, 
and swap the settings for ``port =``, making ``dms`` port 5432.

Edit ``/etc/postgresql/9.3/main/start.conf``, and set it to manual.

Stop ``postgresql``, and start it, (restart will probably result in failure due to a
port clash...)::

 # pg_ctlcluster 9.3 main stop
 # service postgresql stop
 # service postgresql start

Use ``etckeeper`` to migrate the configuration to the replica::

 dms-master1:/etc# etckeeper commit

 dms-master2:/etc#  git fetch dms-master1
 dms-master2:/etc/# git checkout dms-master1/master postgresql/9.3/main postgresql/9.3/dms
 dms-master2:/etc# pg_ctlcluster 9.3 main stop
 dms-master2:/etc# service postgresql stop
 dms-master2:/etc# service postgresql start

On the master, set the DB passwords for the ``dms user and the ``ruser`` (they will
be copied to the replica when mirroring is started)::

 root@dms-master1:/home/grantma# pwgen -acn 16 10 (to pick your password)
 root@dms-master1:/home/grantma# psql -U pgsql dms
 psql (9.3.3)
 Type "help" for help.

 dms=# \password ruser
 Enter new password: 
 Enter it again: 
 dms=# \password dms
 Enter new password: 
 Enter it again: 
 dms=# \q

Note: The ``pgsql`` database super user exists for cross OS/distro compatibility 
reasons.

Record the 2 passwords you have just set for reference.  Put the ``ruser`` password
in ``/etc/dms/pgpassfile`` on both machines, updating the hostnames part of the
entry as well, which is in the standard PGSQL format (see section 31.14 in
"PostgreSQL 9.3.3 Documentation").

NB: You will have to alter the machine name and password. Use ``vi`` or ``vim``
as ``root`` to prevent permissions and ownership alteration.

Also edit ``/etc/dms/dms.conf``, and set the dms ``db_password`` for
``zone_tool`` on both machines as ``zone_tool`` uses password access unless the
user is in ``pg_ident.conf``

Connecting Replica and Starting Replication
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

On the master, and replica, set the replication address in ``pg_hba.conf``::

 dms-master1:/root# dms_admindb -r dms-master2.someorg.net
 dms-master2:/root# dms_admindb -r dms-master1.someorg.net

Set up PGSQL recovery.conf, and start replica DB::

 dms-master2:/root# service postgresql stop
 dms-master2:/root# dms_pg_basebackup dms-master1.someorg.net
 dms-master2:/root# dms_write_recovery_conf dms-master1.someorg.net
 dms-master2:/root# service postgresql start

Note:  The above is seeing DB replica functionality from the default DB as
master

Edit ``/etc/dms/dr-settings.sh``, and update DR_PARTNER to the name of the opposite
server in the DR pair.

Check that replication is running by seeing if zone_tool can access default
configuration settings::

 dms-master2:/root# zone_tool show_config
 root@dms-master2:/home/grantma# zone_tool show_config
         auto_dnssec:       false
         default_ref:       someorg
         default_sg:        someorg-one
         default_stype:     bind9
         edit_lock:         false
         event_max_age:     120.0
         inc_updates:       false
         nsec3:             false
         soa_expire:        7d
         soa_minimum:       24h
         soa_mname:         ns1.someorg.net. (someorg-one)
         soa_refresh:       7200
         soa_retry:         7200
         soa_rname:         soa.someorg.net.
         syslog_max_age:    120.0
         use_apex_ns:       true
         zi_max_age:        90.0
         zi_max_num:        25
         zone_del_age:      0.0
         zone_del_pare_age: 90.0
         zone_ttl:          24h

Master/Replica rsyncd setup
---------------------------

Both the machines will have to ``rsync`` from one another, depending on which is
running as the DR replica.  So we are setting up ``rsync`` client passwords, and 
``rsyncd`` configuration on one, and using the same settings on the other
machine.

Add the following to ``/etc/rsyncd.conf``::

 hosts allow = 2001:db8:f012:2::2/128 2001:db8:f012:2::3/128
 secrets file = /etc/rsyncd.secrets

 [dnsconf]
         path = /var/lib/dms/rsync-config
         uid=bind
         gid=bind
         comment = Slave server config area
         auth users = dnsconf
         use chroot = yes
         read only = no

 [dnssec]
         path = /var/lib/bind/keys
         uid=bind
         gid=dmsdmd
         comment = DNSSEC key data area
         auth users = dnssec
         use chroot = yes
         read only = no

adjusting IP addresses as needed. And also set up the ``/etc/rsyncd.secrets`` file::


 dnsconf:SuperSecret
 dnssec:PlainlyNotSecret

making it only readable by root::

 # chown root:root /etc/rsyncd.secrets
 # chmod 600 /etc/rsyncd.secrets

and set the passwords in ``/etc/dms/rsync-dnssec-password`` and
``/etc/dms/rsync-dnsconf-password`` using ``vi`` to preserve permissions.

and enable the ``rsyncd`` daemon in ``/etc/default/rsync``, and start the service::

 # service rsync start

Use ``etckeeper`` to mirror the configuration to the replica::

 dms-master1:/etc# etckeeper commit

 dms-master2:/etc#  git fetch dms-master1
 dms-master2:/etc/# git checkout dms-master1/master rsyncd.secrets rsyncd.conf /etc/default/rsync dms/rsync-dnsconf-password dms/rsync-dnssec-password
 dms-master2:/etc/# chmod 600 /etc/rsyncd.secrets

And start ``rsyncd`` on the replica as well.

Check that you can connect to the rsync port on one from the other machine,
and vice-versa::

 root@dms-master2:/home/grantma# telnet dms-master1 rsync
 Trying 192.168.101.2...
 Connected to dms-master1.someorg.net.
 Escape character is '^]'.
 @RSYNCD: 30.0
 ^]c

 telnet> c
 Connection closed.
 root@dms-master2:/home/grantma# 

Lets create the master SG, and disabled replica servers (DMS master and DR),
and check that the DR slave named configuration can be rsynced::

 dms-master1:/etc/# zone_tool
 zone_tool > create_sg -p someorg-master /etc/dms/server-config-templates 2001:db8:f012:2::2 2001:db8:f012:2::3
 zone_tool > create_server -g someorg-master dms-master2 2001:db8:f012:2::2
 zone_tool > create_server -g someorg-master dms-master1 2001:db8:f012:2::3
 zone_tool > rsync_server_admin_config dms-master2 no_rndc
 zone_tool >

 dms-master2:/etc/# zone_tool
 zone_tool > rsync_server_admin_config dms-master1 no_rndc
 zone_tool >

Look in ``/var/log/syslog`` on the ``rsyncd`` server to debug issues.

Setting up rsyslog on Master and Replica
----------------------------------------

On the master, create the file ``/etc/rsyslog.d/00network.conf`` with the
contents::

 # provides UDP syslog reception
 $ModLoad imudp
 $UDPServerRun 514

 # provides TCP syslog reception
 $ModLoad imtcp
 $InputTCPServerRun 514

 #$AllowedSender UDP, [2001:db8:c:110e::2]
 #$AllowedSender TCP, [2001:db8:c:110e::2]
 #$AllowedSender UDP, [2001:db8:66:23::2]
 #$AllowedSender TCP, [2001:db8:66:23::2]
 #$AllowedSender UDP, [2001:db8:ba69:1:21c:f0ff:fefa:f3c0]
 #$AllowedSender TCP, [2001:db8:ba69:1:21c:f0ff:fefa:f3c0]

All replica and slave DNS servers will have to be entered into this file.

Also alter the file ``/etc/rsyslog.d/pgsql`` and change the contents to::

 ### Configuration file for rsyslog-pgsql
 ### Changes are preserved

 $ModLoad ompgsql
 local7.* /var/log/local7.log
 local7.* :ompgsql:/var/run/postgresql,dms,rsyslog,

Do the same for the replica, apart from the following.

IMPORTANT: On the Replica, comment out the last local7.* line.  Don't change
the contents of that line, as the administration scripts go searching for
exactly that line. Replica file is as follows::

 ### Configuration file for rsyslog-pgsql
 ### Changes are preserved

 $ModLoad ompgsql
 local7.* /var/log/local7.log
 #local7.* :ompgsql:/var/run/postgresql,dms,rsyslog,

The default configuration propagated to the DMS servers uses ``local7`` as the
``named`` logging facility.

DMS Configuration
=================

Setting initial DR settings on both machines
--------------------------------------------

On both machines, edit ``/etc/dms/dr-settings.sh``, and set DR_PARTNER to the name
of the opposite machine::

 dms-master1# vi /etc/dms/dr-settings.sh

 # Settings file for dr-scripts and Net24 PG database scripts

 # DR Partner server host name
 # ie default dms_start_as_replica master
 # This is the exact DNS/host name which you have to replicate from
 DR_PARTNER="dms4-d4-dr.someorg.net
 .
 .
 .

The rest of the file is for type of DR fail over - by IP on a loop back
interface and/or routing, or by a fail over domain in the DNS.  We will set this
up later.

Setting up Bind9 master DNS server
----------------------------------

Create all the required TSIG rndc and dynamic DNS update keys, and generate
required ``/etc/bind/rndc.conf``:

(If any of these commands stall, VM/machine does not have enough entropy.  Make
sure haveged is installed and running.)

::

 root@dms-master1:/etc/dms/bind# zone_tool generate_tsig_key -f update-ddns hmac-sha256 update-session.key
 root@dms-master1:/etc/dms/bind# zone_tool generate_tsig_key -f rndc-key hmac-md5 rndc-local.key
 root@dms-master1:/etc/dms/bind# zone_tool generate_tsig_key -f remote-key hmac-md5 rndc-remote.key
 root@dms-master1:/etc/dms/bind# zone_tool write_rndc_conf -f
 root@dms-master1:/etc/dms/bind# cp -a rndc-remote.key /etc/dms/server-admin-config/bind9
 root@dms-master1:/etc/dms/bind# cp rndc-remote.key /var/lib/dms/rsync-config

Add the ``/etc/dms/bind/named.conf`` to ``/etc/default/bind9``, and add a line to get
rid of the default ``rndc.key`` to stop ``rndc`` complaining::

 # Get rid of default bind9 rndc.key, that debian install scripts always
 # generate  Stops rndc complaining:
 rm -f /etc/bind/rndc.key

 # run resolvconf?
 RESOLVCONF=no

 # startup options for the server
 OPTIONS="-u bind -c /etc/dms/bind/named.conf"

Create ``/etc/bind/rndc.conf``, to include the following::

 # include rndc configuration generated by DMS zone_tool
 include "/var/lib/dms/rndc/rndc.conf";

Restart ``named`` to make sure all is good::

 root@dms-master1:/etc/bind# service bind9 stop
 root@dms-master1:/etc/bind# service bind9 start
 root@dms-master1:/etc/bind# rndc status
 version: 9.9.5-2-Debian <id:f9b8a50e>
 CPUs found: 1
 worker threads: 1
 number of zones: 5
 debug level: 0
 xfers running: 0
 xfers deferred: 0
 soa queries in progress: 0
 query logging is OFF
 recursive clients: 0/0/1000
 tcp clients: 0/100
 server is up and running

Enable ``dmsdmd``, the dynamic DNS update and DMS event daemon by editing
``/etc/default/dmsdmd``, setting ``DMSDMD_ENABLE=true``, and start it::

 root@dms-master1:/etc# vi /etc/default/dmsdmd
 root@dms-master1:/etc# service dmsdmd start
 [ ok ] Starting dmsdmd: dmsdmd.
 root@dms-master1:/etc# service dmsdmd status
 [ ok ] dmsdmd is running.

Enable the master server so that the server SM can monitor named on the
machine (briefly, this server twitters to itself)::

 root@dms-master1:/etc# zone_tool enable_server dms-master1

This means that when ``dmsdmd`` is started, it will set up an index in the
Master SM in the DB to the Master server in the ServerSM table (important for
keeping track of where the master is for human output and ServerSM
functionality - uses machines actual network addresses cf.  ``master_address``
and ``master_alt_address`` in replica SG)

And make sure you can create a domain::

 root@dms-master1:/etc/dms/bind# zone_tool create_zone foo.bar.org
 root@dms-master1:/etc/dms/bind# zone_tool show_zone foo.bar.org
 $TTL 24h
 $ORIGIN foo.bar.org.

 ;
 ; Zone:      foo.bar.org.
 ; Reference: someorg
 ; zi_id:     1
 ; ctime:     Mon Jul  2 11:30:26 2012
 ; mtime:     Mon Jul  2 11:31:03 2012
 ; ptime:     Mon Jul  2 11:31:03 2012
 ;


 ;| Apex resource records for foo.bar.org.
 ;!REF:someorg
 @                       IN      SOA             ( ns1.someorg.net. ;Master NS
                                                 soa.someorg.net. ;RP email
                                                 2012070200   ;Serial yyyymmddnn
                                                 7200         ;Refresh
                                                 7200         ;Retry
                                                 604800       ;Expire
                                                 86400        ;Minimum/Ncache
                                                 )           
                         IN      NS              ns2.someorg.net.
                         IN      NS              ns1.someorg.net.


 root@dms-master1:/etc/dms/bind# zone_tool show_zonesm foo.bar.org
         name:            foo.bar.org.
         alt_sg_name:     None
         auto_dnssec:     False
         ctime:           Mon Jul  2 11:30:26 2012
         deleted_start:   None
         edit_lock:       False
         edit_lock_token: None
         inc_updates:     False
         lock_state:      EDIT_UNLOCK
         mtime:           Mon Jul  2 11:30:26 2012
         nsec3:           False
         reference:       someorg
         soa_serial:      2012070200
         sg_name:         someorg-one
         state:           PUBLISHED
         use_apex_ns:     True
         zi_candidate_id: 1
         zi_id:           1
         zone_id:         1
         zone_type:       DynDNSZoneSM
         zi_id:           1
         ctime:           Mon Jul  2 11:30:26 2012
         mtime:           Mon Jul  2 11:31:03 2012
         ptime:           Mon Jul  2 11:31:03 2012
         soa_expire:      7d
         soa_minimum:     24h
         soa_mname:       ns1.someorg.net.
         soa_refresh:     7200
         soa_retry:       7200
         soa_rname:       soa.someorg.net.
         soa_serial:      2012070200
         soa_ttl:         None
         zone_id:         1
         zone_ttl:        24h
 
 root@dms-master1:/etc/dms/bind# dig -t AXFR +noall +answer foo.bar.org @localhost
 foo.bar.org.		86400	IN	SOA	ns1.someorg.net. soa.someorg.net. 2012070200 7200 7200 604800 86400
 foo.bar.org.		86400	IN	NS	ns1.someorg.net.
 foo.bar.org.		86400	IN	NS	ns2.someorg.net.
 foo.bar.org.		86400	IN	SOA	ns1.someorg.net. soa.someorg.net. 2012070200 7200 7200 604800 86400
 root@dms-master1:/etc/dms/bind# zone_tool delete_zone foo.bar.org
 
Reflect the ``bind`` and ``dms`` directories to the DR via ``etckeeper``::

 root@dms-master1:/etc# etckeeper commit

 root@dms-master2:/etc# git fetch dms-master1
 root@dms-master2:/etc# git checkout dms-master1/master dms/bind
 root@dms-master2:/etc# git checkout dms-master1/master bind
 root@dms-master2:/etc# git checkout dms-master1/master default/bind9

Setting UP DR bind9 slave server on Replica
-------------------------------------------

Edit ``/etc/dms/server-admin-config/bind9/controls.conf`` and add each masters IP
address to the uncommented inet  allow line.  IPv4 address will have to be
prefixed with ``::ffff:`` as by default Linux binds  v6 sockets to IPv4.

``Rsync`` the admin config from the master to the DR replica, not doing any ``rndc``
reconfig::

 root@dms-master1:/etc# zone_tool rsync_server_admin_config dms-master2 no_rndc

Copy the ``/etc/dms/server-admin-config/bind9`` directory to
``/var/lib/dms/rsync-config``::

 root@dms-master1:/etc# cp -a /etc/dms/server-admin-config/bind9/* /var/lib/dms/rsync-config
 root@dms-master1:/etc# chown bind:bind /var/lib/dms/rsync-config/*

Reflect the bind directory to the DR via ``etckeeper``:

 root@dms-master1:/etc# etckeeper commit

 root@dms-master2:/etc# git fetch dms-master1
 root@dms-master2:/etc# git checkout dms-master1/master dms/server-admin-config

To apply permissions on master to replica::

 root@dms-master2:/etc# git checkout dms-master1/master .etckeeper
 root@dms-master2:/etc# etckeeper init
 root@dms-master2:/etc# etckeeper commit

Create ``rndc.conf`` include needed to start ``bind``::

 root@dms-master2:/etc# zone_tool write_rndc_conf

On the replica, edit ``/etc/default/bind9``, adding ``-c
/etc/bind/named-dr-replica.conf`` to OPTIONS, and restart ``named``::

 root@dms-master2:/etc# service bind9 restart

On the master, enable the DR replica server in the replica SG::

 root@dms-master1:/etc# zone_tool enable_server dms-master2

Check by switching between master and replica::

 root@dms-master1:/etc# dms_master_down

 root@dms-master2:/etc/# dms_promote_replica

 root@dms-master1:/etc# dms_start_as_replica dms-master2.someorg.net

Wait for synchronization to be shown 15 - 20 minutes::

 root@dms-master2:/etc# zone_tool show_replica_sg -v
         sg_name:             someorg-master
         config_dir:          /etc/dms/server-config-templates
         master_address:      192.168.101.2
         master_alt_address:  192.168.102.2
         replica_sg:          True
         sg_id:               2
         zone_count:          0

         Slave Servers:
         dms-master2                      192.168.102.2                           
                 OK
         dms-master1                  192.168.101.2                           
                 OK

and switch back as above.

Importing Zones to DMS system
-----------------------------

Set the default settings shown in zone_tool show_config on the DMS master::

 root@dms-master1:/etc# zone_tool show_config
 root@dms-master1:/etc# zone_tool set_config soa_mname ns1.foo.bar.net
 root@dms-master1:/etc# zone_tool set_config soa_rname soa.foo.bar.net
 root@dms-master1:/etc# zone_tool set_config default_sg foo-bar-net
 root@dms-master1:/etc# zone_tool set_config default_ref FOO-BAR-NET
 root@dms-master1:/etc# zone_tool show_apex_ns
 root@dms-master1:/etc# zone_tool edit_apex_ns

Create the default SG::

 root@dms-master1:/etc# zone_tool create_sg someorg-one

Aside: Apex NS records can be created and edited for each server group.  By
default, the ``apex_ns`` records for the default SG are used.  Use::

 zone_tool> show_apex_ns some-sg
 zone_tool> edit_apex_ns some-sg

to create and edit the apex NS server names.

Create all required reverse zone on the master, setting the ``zone_tool
create_zone`` ``inc_updates`` flag argument so that auto reverse zone records can be
created and managed::

 root@dms-master1:/etc# zone_tool create_zone 2001:2e8:2012::/32 inc_updates

Import all the zones. First of all, load the apex zone which contains the
ns1/ns2 records with ``no_use_apex_ns``, then load all the rest. Its an idea to
have a look at the ``edit_lock`` flag at the same time for those top zone(s).  Note
that ``zone_tool load_zones`` requires all files to be named by full domain name::

 root@dms-master1:/some/dir/with/zone/files# zone_tool load_zone foo.bar.net foo.bar.net no_use_apex_ns edit_lock
 root@dms-master1:/some/dir/with/zone/files# zone_tool load_zones *

Setting up failover domain
--------------------------

This is the easiest way to re-point the Web UIs at the correct master server.
Another alternative is to use a loop back interface with a floating IP address,
and propagation via routing or simply by being on the ethernet segment. At the
moment the interface method requires the installation of ``netscript-2.4`` instead
of ``ifupdown``.

1. Create a fail-over domain

   This needs to be updated by incremental updates::

     zone_tool > create_zone failover.someorg.net inc_updates


2. Edit ``/etc/dms/dr-settings.sh``, enable ``DMSDRDNS``, set ``DMS_FAILOVER_DOMAIN``, 
   the ``DMS_WSGI_LABEL`` (DNS host that 'floats' to where master is), and the TTL::

     # zone_tool update_rrs settings, for WSGI DNS name  
     # Uses a CNAME based template.
     # Following is a flag to turn it on or off
     DMSDRDNS=true

     # If not defined or empty, the following is set to the hostname
     DMS_MASTER=""
 
     DMS_FAILOVER_DOMAIN="failover.someorg.net."
 
     DMS_WSGI_LABEL="dms-server"
 
     DMS_WSGI_TTL="30s"
 
     DMS_UPDATE_RRS_TEMPLATE='
     $ORIGIN @@DMS_FAILOVER_DOMAIN@@
     $UPDATE_TYPE wsgi-failover
     ;!RROP:UPDATE_RRTYPE
     @@DMS_WSGI_LABEL@@  @@DMS_WSGI_TTL@@  IN      CNAME      @@DMS_MASTER@@
     '

3. Repeat 2. on DR server

4. Fail over system back and forth to establish DNS records and test::

     dms-master1 # dms_master_down
     dms-master2 # dms_promote_replica
     dms-master1 # dms_start_as_replica
     .
     .
     .

and reverse::

 .
 .
 .
 dms-master2 # dms_master_down
 dms-master1 # dms_promote_replica
 dms-master2 # dms_start_as_replica

And you should be good to go, with a DMS WSGI server name of
``dms-server.failover.someorg.net.`` 

Setting up WSGI on apache
-------------------------

Enable WSGI in ``/etc/dms/dr-settings.sh`` on both machines by editing file.

Include the ``/etc/dms/dms-wsgi-apache.conf`` fragment into the file
``/etc/apache2/sites-available/default-ssl``.

Set the apache log level to info, delete the cgi-bin section, and set up the 
SSL certificates.

Create the ``htpasswd`` file ``/etc/dms/htpasswd-dms``, and set the passwords for
``admin-dms``, ``helpdesk-dms``, ``value-reseller-dms``, ``hosted-dms`` WSGI users.

Also don't forget to::

 dms-master1 # cd /etc/dms
 dms-master1 # chown root:www-data htpasswd-dms
 dms-master1 # chmod 640 htpasswd-dms

Use ``a2ensite`` and ``a2dissite`` to enable the SSL default site::

 # a2dissite default
 # a2ensite default-ssl

Reload apache2::

 # service apache2 reload

Reflect configuration as above to DR partner server.

Check that it functions by using curl on the master server::

 # cd /tmp
 # cp -a /usr/share/doc/dms-core/examples/wsgi-json-testing .
 # cd wsgi-json-testing

Edit ``json-test.sh`` so that it works for you, re URLs and user/password.
``Test4.jsonrpc`` uses ``list_zone``, so try that first to check WSGI is live:

 # ./json-test.sh test4

It is helpful to edit curl command to include ``--insecure`` if you are using a
self-signed SSL certificate.

It may take a while before anything shows up if you have imported tens of
thousands of zones.  Full error information will be shown in the configured
apache error log  ``/var/log/apache2/error.log``.  You can also try some of the
other example tests as well after editing them for the current setup.

Edit the WSGI configuration in ``/etc/dms`` to your liking. See documentation for
more details.

Mirror apache2 config to other DR partner server::

 dms-master1 # etckeeper commit

 dms-master2 # cd /etc
 dms-master2 # git fetch dms-master1
 dms-master2 # git checkout dms-master1/master apache2 dms/htpasswd-dms \
               dms/dms-wsgi-apache.conf dms/wsgi-scripts

Fix permissions::

 dms-master2 # git checkout dms-master1/master .etckeeper
 dms-master2 # etckeeper init
 dms-master2 # etckeeper commit

NOTE: Also try some of the read-only tests on the other DR partner server to
make sure WSGI is functional there. You will have to fail over to do this.

.. _Setting-up-a-Slave-Server:

Setting up a Slave DNS Server
=============================

Based on Debian Wheezy.  

Has 2 connections back to DMS DR partner servers.  You can leave one
server out for racoon if only running one single DMS master server.

To prevent installation of recommended packages add the following to
``/etc/apt/apt.conf.d/00local.conf``::

 // No point in installing a lot of fat on VM servers
 APT::Install-Recommends "0";
 APT::Install-Suggests "0";

Install these packages::

 # aptitude install bind9 strongswan rsync cron-apt bind9-host dnsutils \
   screen psmisc procps tree sysstat lsof telnet-ssl apache2-utils ntp

IPSEC
-----

See section above on IPSEC for how to do this.

Rsync
-----

1. Edit ``/etc/default/rsync``, and enable ``rsyncd``

2. Create ``/etc/rsyncd.conf``::

     hosts allow = 2001:db8:f012:2::2/128 2001:db8:f012:2::3/128
     secrets file = /etc/rsyncd.secrets

     [dnsconf]
            path = /srv/dms/rsync-config
 	    uid=bind
 	    gid=bind
 	    comment = Slave server config area
 	    auth users = dnsconf
 	    use chroot = yes
 	    read only = no

3. Create ``/etc/rsyncd.secrets``::

     dnsconf:SuperSecretRsyncSlavePasswoord

   and make it only readable/writable by ``root``::

     root # chmod 600 /etc/rsyncd.secrets
     root # chown root:root /etc/rsyncd.secrets

4. Do this at the shell to create target ``/srv/dms/rsync-config`` directory::

     # mkdir -p /srv/dms/rsync-config
     # chown bind:bind /srv/dms/rsync-config

5. And named slave directory::

     # mkdir /var/cache/bind/slave
     # chown root:bind /var/cache/bind/slave
     # chmod 775 /var/cache/bind/slave

6. Start ``rsyncd``. Edit /etc/default/rsync to enable daemon

 # service rsync start

7. Test connectivity from DMS Masters::

     dms-master1# telnet new-slave domain
     dms-master1# telnet new-slave rsync
     dms-master2# telnet new-slave domain
     dms-master2# telnet new-slave rsync

Test by rsyncing configuration to slave - needed for configuring ``bind9``::

 zone_tool> create_server new-slave-name ip-address
 zone_tool> rsync_server_admin_config new-slave-name no_rndc
 
Bind9
-----

Change ``/etc/bind/named.conf.options`` to the following::

 options {
 directory "/var/cache/bind";
         // If there is a firewall between you and nameservers you want
         // to talk to, you may need to fix the firewall to allow multiple
         // ports to talk. See http://www.kb.cert.org/vuls/id/800113
         // If your ISP provided one or more IP addresses for stable
         // nameservers, you probably want to use them as forwarders.
         // Uncomment the following block, and insert the addressesm replacing
         // the all-0's placeholder.

         // forwarders {
         //       0.0.0.0;
         // };

 //========================================================================
 // If BIND logs error messages about the root key being expired,
 // you will need to update your keys. See https://www.isc.org/bind-keys
 //========================================================================
        // dnssec-validation auto;
        // auth-nxdomain no; # conform to RFC1035
 
        listen-on { localhost; };
        listen-on-v6 { any; };
        include "/srv/dms/rsync-config/options.conf";
 };

Note that the listen directives are given in file, Debian options commented
out, as they are set in the rsync-ed include at the bottom.

Change ``/etc/bind/named.conf.local`` to the following::

 //
 // Do any local configuration here
 //

 // Consider adding the 1918 zones here, if they are not used in your
 // organization
 //include "/etc/bind/zones.rfc1918";

 // rndc config
 include "/etc/bind/rndc.key";
 include "/srv/dms/rsync-config/rndc-remote.key";
 include "/srv/dms/rsync-config/controls.conf";
 // Logging configuration
 include "/srv/dms/rsync-config/logging.conf";
 // Secondary zones
 include "/srv/dms/rsync-config/bind9.conf";

This file is used to include all the required bits from the
``/srv/dms/rsync-config`` directory. All this configuration can now be updated
from the master server, and the slave reconfigured – but watch it when you go
changing the rndc keys.

Restart bind9::

 # touch /srv/dms/rsync-config/bind9.conf
 # chown bind:bind /srv/dms/rsync-config/bind9.conf

 # service bind9 restart
 
and check ``/var/log/syslog`` for any errors.

Check that on the master servers that ``zone_tool rsync_server_admin_config``
works, by default will ``rndc`` the slave::

 dms-master1# zone_tool write_rndc_conf
 dms-master1# zone_tool rsync_server_admin_config new-slave

 dms-master2# zone_tool write_rndc_conf
 dms-master2# zone_tool rsync_server_admin_config new-slave

Enable Server
-------------

On the live DMS master, enable the slave, and watch that it changes state to
OK. This may take 15-20 minutes

NOTE: a ``reconfig_sg`` may be needed to initially seed zone configuration files on the master.
These files are automatically created/updated if a new domain is added to the
server group.

::

 dms-master-live# zone_tool
 zone_tool > enable_server <slave-name>
 zone_tool > reconfig_sg someorg-one    .
 .
 .
 zone_tool > ls_pending_events
 ServerSMCheckServer       dms-master2                  Fri Mar 14 11:28:55 2014
 ServerSMConfigure         dms-slave1                   Fri Mar 14 11:31:35 2014
 ServerSMCheckServer       dms-master1                  Fri Mar 14 11:28:54 2014
 .
 .
 .
 zone_tool > show_sg someorg-one
         sg_name:             someorg-one
         config_dir:          /etc/dms/server-config-templates
         master_address:      None
         master_alt_address:  None
         replica_sg:          False
         zone_count:          4
 
         DNS server status:
         dms-slave1                   fd14:828:ba69:7::18                     
                 OK
 zone_tool >


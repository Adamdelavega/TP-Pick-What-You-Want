# TP-Pick-What-You-Want

#### Conf DHCP

**server1**

```
[adam@server1 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s9
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
NAME=enp0s9
DEVICE=enp0s9
ONBOOT=yes
IPADDR=10.2.20.253
NETMASK=255.255.255.0
GATEWAY=10.2.20.252
```

* serveur CentOS7
* installer le paquet `dhcp`
```
[adam@server1 ~]$ yum list installed
Loaded plugins: fastestmirror
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Loading mirror speeds from cached hostfile
 * base: centos-mirror.usessionbuddy.com
 * extras: centos.crazyfrogs.org
 * updates: centos-mirror.usessionbuddy.com
 [...]
dhclient.x86_64                                                                                     12:4.2.5-83.el7.centos.1                                                                      @updates 
dhcp.x86_64                                                                                         12:4.2.5-83.el7.centos.1                                                                      @updates 
dhcp-common.x86_64                                                                                  12:4.2.5-83.el7.centos.1                                                                      @updates 
dhcp-libs.x86_64  
 [...]
```
* configurer `/etc/dhcp/dhcpd.conf`
```
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page

# create new

# specify domain name
option domain-name     "adam.tp1";

# specify DNS server's hostname or IP address
option domain-name-servers     server1.adam.tp1;

# # default lease time
default-lease-time 600;

# max lease time
max-lease-time 7200;
# this DHCP server to be declared valid

authoritative;
# specify network address and subnetmask

subnet 10.2.20.253 netmask 255.255.255.255 {}

subnet 10.2.30.0 netmask 255.255.255.0 {
     # specify the range of lease IP address
         range dynamic-bootp 10.2.30.1 10.2.30.253;
                     # specify gateway
                         option routers 10.2.30.254;
                         }

```
* démarrer le service `dhcpd`
```
[root@server1 adam]# sudo systemctl status dhcpd
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/etc/systemd/system/dhcpd.service; enabled; vendor preset: disabled)
   Active: active (running) since mer. 2022-01-12 18:05:20 CET; 14min ago
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)
 Main PID: 1022 (dhcpd)
   Status: "Dispatching packets..."
   CGroup: /system.slice/dhcpd.service
           └─1022 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid enp0s9

```
* Le test ne peut pas avoir lieux pour l'instant car le routage n'a pas encore été fait.

#### Conf DNS

* installer les paquets `bind` (serveur DNS) et `bind-utils` (outils DNS)
```
[adam@server1 ~]$ yum list installed
Loaded plugins: fastestmirror
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Loading mirror speeds from cached hostfile
 * base: centos-mirror.usessionbuddy.com
 * extras: centos.crazyfrogs.org
 * updates: centos-mirror.usessionbuddy.com
Installed Packages
[...]
bind.x86_64                                                                                         32:9.11.4-26.P2.el7_9.8                                                                       @updates 
bind-export-libs.x86_64                                                                             32:9.11.4-26.P2.el7_9.8                                                                       @updates 
bind-libs.x86_64                                                                                    32:9.11.4-26.P2.el7_9.8                                                                       @updates 
bind-libs-lite.x86_64                                                                               32:9.11.4-26.P2.el7_9.8                                                                       @updates 
bind-license.noarch                                                                                 32:9.11.4-26.P2.el7_9.8                                                                       @updates 
bind-utils.x86_64                                                                                   32:9.11.4-26.P2.el7_9.8                                                                       @updates 
binutils.x86_64  
[...]
```
* configurer le serveur DNS en suivant les exemples de configuration
  * [`./hints/named.conf`](./hints/named.conf)
    * fichier de configuration de bind
  * [`./hints/dnsdomip.db`](./hints/dnsdomip.db)
    * fichier de zone
    * détermine quels noms de domaine seront traduisibles en IP
  * [`./hints/dnsipdom`](./hints/dnsipdom.db)
    * fichier de zone inverse
    * détermine quelles IPs pouront être traduites en noms de domaine
  * [exemples de conf](./dns)
```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//
// See the BIND Administrator's Reference Manual (ARM) for details about the
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

options {
	listen-on port 53 { any; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	recursing-file  "/var/named/data/named.recursing";
	secroots-file   "/var/named/data/named.secroots";
	allow-query     { any; };

	/* 
	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
	 - If you are building a RECURSIVE (caching) DNS server, you need to enable 
	   recursion. 
	 - If your recursive DNS server has a public IP address, you MUST enable access 
	   control to limit queries to your legitimate users. Failing to do so will
	   cause your server to become part of large scale DNS amplification 
	   attacks. Implementing BCP38 within your network would greatly
	   reduce such attack surface 
	*/
	recursion no;

	dnssec-enable yes;
	dnssec-validation yes;

	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.root.key";

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

zone "adam.tp1" IN {
	type master;
	file "/var/named/adam.tp1.db";
	allow-update { none; };	
};

zone"10.2.0.db" IN {
	type master;
	file "var/named/10.2.0.db";
	allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

**zone forward**
```
$TTL 86400
@   IN  SOA     server1.adam.tp1. root.adam.tp1. (
2020031101 ;Serial
10800   ;Refresh
600     ;Retry
1814400 ;Expire
10800   ;Minimum TTL
)
;Name Server Information
@      IN  NS      server1.adam.tp1.
;IP address of Name Server
server1 IN  A       10.2.20.253
;A - Record HostName To IP Address
server2 IN  A       10.2.20.251
R1      IN  A       10.2.20.254
R7      IN  A       10.2.20.252
```

**zone inverse**
```
$TTL 86400
@   IN  SOA     server1.adam.tp1. root.adam.tp1. (
2011071001 ;Serial
3600    ;Refresh
1800    ;Retry
604800  ;Expire
86400   ;Minimum TTL
)
;Name Server Information
@ IN  NS      server1.adam.tp1.
@ IN  NS      server2.adam.tp1.
;Reverse lookup for Name Server
253        IN  PTR     server1.adam.tp1.

server1    IN  A  10.2.20.253
server2    IN  A  10.2.20.251
R1         IN  A  10.2.20.254
R7         IN  A  10.20.2.252
;PTR Record IP address to HostName
251      IN  PTR     server2.adam.tp1
252      IN  PTR     R7
254      IN  PTR     R1
```

#### Conf serveur web

**server2**

```
[adam@server2 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s9
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
NAME=enp0s9
DEVICE=enp0s9
ONBOOT=yes
IPADDR=10.2.20.251
NETMASK=255.255.255.0
GATEWAY=10.2.20.252
```

```
[adam@server2 ~]$ sudo ss -alntp
[sudo] password for adam: 
State       Recv-Q Send-Q                                                        Local Address:Port                                                                       Peer Address:Port              
LISTEN      0      128                                                                       *:80                                                                                    *:*                   users:(("nginx",pid=1053,fd=6),("nginx",pid=1052,fd=6))
LISTEN      0      128                                                                       *:22                                                                                    *:*                   users:(("sshd",pid=1033,fd=3))
LISTEN      0      100                                                               127.0.0.1:25                                                                                    *:*                   users:(("master",pid=1264,fd=13))
LISTEN      0      128                                                                    [::]:80                                                                                 [::]:*                   users:(("nginx",pid=1053,fd=7),("nginx",pid=1052,fd=7))
LISTEN      0      128                                                                    [::]:22                                                                                 [::]:*                   users:(("sshd",pid=1033,fd=4))
LISTEN      0      100                                                                   [::1]:25                                                                                 [::]:*                   users:(("master",pid=1264,fd=14))
[adam@server2 ~]$ 
```

```
adam@server2 ~]$ curl 127.0.0.1
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css"> 

	html {
	background-image:url(img/html-background.png);
	background-color: white;
	font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
	font-size: 0.85em;
	line-height: 1.25em;
	margin: 0 4% 0 4%;
	}
[...]
```

#### Conf switchs

**sw1**
```
Switch#show running-config
Building configuration...

Current configuration : 3672 bytes
!
! Last configuration change at 20:51:29 UTC Wed Jan 3 2022
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname Switch
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
!
!
!
!
!         
!         
!         
!         
ip cef    
no ipv6 cef
!         
!         
!         
spanning-tree mode pvst
spanning-tree extend system-id
!         
vlan internal allocation policy ascending
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet0/1
 switchport access vlan 30
 switchport mode access
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet0/2
 switchport access vlan 10
 switchport mode access
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet0/3
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/0
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/1
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/2
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/3
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/0
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/1
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/2
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/3
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/0
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/1
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/2
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/3
 media-type rj45
 negotiation auto
!         
ip forward-protocol nd
!         
no ip http server
no ip http secure-server
!         
!         
!         
!         
!         
!         
control-plane
!         
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!         
line con 0
line aux 0
line vty 0 4
 login    
!         
!         
end   
```

**sw2**

```
Switch#show running-config
Building configuration...

Current configuration : 3672 bytes
!
! Last configuration change at 19:32:45 UTC Wed Jan 3 2022
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname Switch
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
!
!
!
!
!         
!         
!         
!         
ip cef    
no ipv6 cef
!         
!         
!         
spanning-tree mode pvst
spanning-tree extend system-id
!         
vlan internal allocation policy ascending
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet0/1
 switchport access vlan 30
 switchport mode access
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet0/2
 switchport access vlan 10
 switchport mode access
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet0/3
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/0
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/1
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/2
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet1/3
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/0
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/1
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/2
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet2/3
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/0
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/1
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/2
 media-type rj45
 negotiation auto
!         
interface GigabitEthernet3/3
 media-type rj45
 negotiation auto
!         
ip forward-protocol nd
!         
no ip http server
no ip http secure-server
!         
!         
!         
!         
!         
!         
control-plane
!         
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!         
line con 0
line aux 0
line vty 0 4
 login    
!         
!         
end   
```

#### Conf routeurs

**R1**

```
R1#show running-config
Building configuration...

Current configuration : 1307 bytes
!
version 12.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R1
!
boot-start-marker
boot-end-marker
!
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
!
!
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
ip tcp synwait-time 5
!         
!         
!         
!         
!         
interface FastEthernet0/0
 no ip address
 duplex auto
 speed auto
!         
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.2.10.254 255.255.255.0
!         
interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.2.20.254 255.255.255.0
!         
interface FastEthernet0/0.30
 encapsulation dot1Q 30
 ip address 10.2.30.254 255.255.255.0
!         
interface FastEthernet0/1
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
interface FastEthernet1/0
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
interface FastEthernet2/0
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
ip forward-protocol nd
!         
!         
no ip http server
no ip http secure-server
!         
no cdp log mismatch duplex
!         
!         
!         
control-plane
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
line con 0
 exec-timeout 0 0
 privilege level 15
 logging synchronous
line aux 0
 exec-timeout 0 0
 privilege level 15
 logging synchronous
line vty 0 4
 login    
!         
!         
end 
```

**R7**

```
R7#show running-config
Building configuration...

Current configuration : 1038 bytes
!
version 12.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R7
!
boot-start-marker
boot-end-marker
!
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
!
!
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
ip tcp synwait-time 5
!         
!         
!         
!         
!         
interface FastEthernet0/0
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
interface FastEthernet0/1
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
interface FastEthernet1/0
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
interface FastEthernet2/0
 no ip address
 shutdown 
 duplex auto
 speed auto
!         
ip forward-protocol nd
!         
!         
no ip http server
no ip http secure-server
!         
no cdp log mismatch duplex
!         
!         
!         
control-plane
!         
!         
!         
!         
!         
!         
!         
!         
!         
!         
line con 0
 exec-timeout 0 0
 privilege level 15
 logging synchronous
line aux 0
 exec-timeout 0 0
 privilege level 15
 logging synchronous
line vty 0 4
 login    
!         
!         
end  
```
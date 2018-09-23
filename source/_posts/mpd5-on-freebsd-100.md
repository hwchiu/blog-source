---
title: mpd5 on FreeBSD 10.0
keywords: 'freebsd, vpnserver, mpd5,vpn'
date: '2014-07-26 14:29'
comments: true
tags:
  - VPN
  - FreeBSD
  - Network
  - System
abbrlink: 44759
description: VPN server is a very useful tool for your network connectivity, although there're many online VPN service around the world, it's slow speed and money cost and you can't sure they won't collect your connection data. That's why sometimes we want to build the VPN server by ourself and this porst introduce a way to setup a VPN server in your FreeBSD server.

---

# Preface
Install a VPN server on FreeBSD 10.0-Release on Amazon EC2.

# Install
## Pkg
- pkg install mpd5  

## Ports
- portmaster net/mpd5

# Config
## VPN Configuraion
- **cp /usr/local/etc/mpd5/mpd.conf.sample /usr/local/etc/mpd5/mpd.conf**
- `set user hwchiu 123456` used to config the admin's accoutn and password of the web page.
- `set web self 0.0.0.0 5006` is the listen ip address and port of the web page.

```bash=
startup:
        # configure mpd users
        set user hwchiu 123456  
        # configure the console
        set console self 127.0.0.1 5005
        set console open
        # configure the web server
        set web self 0.0.0.0 5006  
        set web open
```

- comment the `dialup` and add `pptp_server`, we will config the options of pptp_server later.

``` sh
default:
        #load dialup
        load pptp_server
```

- `set ippool add pool1 ip_start, ip_end` is used to set the private ip range for vpn user. the name `ippool` and `pool1` must be the same as `set ipcp ranges 192.168.1.1/32 ippool pool1`
- `set ipcp ranges 192.168.1.1/32 ippool pool1` is the ip address of the server.
- `set ipcp dns 172.31.0.2` is used to set the dns server. In my case, since my machine is behind the EC2, i used the same configuration in my FreeBSD.
- `set ipcp nbns 172.31.0.2`is used to for windows client.
- `set pptp self 172.31.18.110`. You should set your ip address which is shown on the network interface.
``` sh
# Define dynamic IP address pool.
        set ippool add pool1 192.168.1.50 192.168.1.99
# Create clonable bundle template named B
        create bundle template B
        set iface enable proxy-arp
        set iface idle 1800
        set iface enable tcpmssfix
        set ipcp yes vjcomp
# Specify IP address pool for dynamic assigment.
        set ipcp ranges 192.168.1.1/32 ippool pool1
        set ipcp dns 172.31.0.2
        set ipcp nbns 172.31.0.2
# The five lines below enable Microsoft Point-to-Point encryption
# (MPPE) using the ng_mppc(8) netgraph node type.
        set bundle enable compression
        set ccp yes mppc
        set mppc yes e40
        set mppc yes e128
        set mppc yes stateless
# Create clonable link template named L
        create link template L pptp
# Set bundle template to use
        set link action bundle B
# Multilink adds some overhead, but gives full 1500 MTU.
        set link enable multilink
        set link yes acfcomp protocomp
        set link no pap chap eap
        set link enable chap
# We can use use RADIUS authentication/accounting by including
# another config section with label 'radius'.
#       load radius
        set link keep-alive 10 60
# We reducing link mtu to avoid GRE packet fragmentation.
        set link mtu 1460
# Configure PPTP
        set pptp self 172.31.18.110
# Allow to accept calls
        set link enable incoming
```

## Use configuration
- **cp /usr/local/etc/mpd5/mpd.secret.sample  /usr/local/etc/mpd5/mpd.secret**
The format of mpd.secret is `username password ip_address` per line.  
- **Example**

``` sh
fred            "fred-pw"
joe             "foobar"        192.168.1.1
```

## System configuration
- **sysctl net.inet.ip.forwarding=1**
- **Pf configuraion**
  1. use NAT for internal private network.
  2. skip the lo interface.
  3. block adll traffic adn log all packet by default.
  4. pass in tcp for port 1723 (PPTP)
  5. pass in protocol gre
  6. pass in from any to internal private network and vice versa.
  7. Use the `pfctl -f file` to reload the pf instead of `/etc/rc.d/pf restart`, the latter will disconnect all exist connection.
``` sh
my_int = "xn0"
internal_net = "192.168.0.0/16"
external_addr = "172.31.18.110"
nat on $my_int from $internal_net to any -> $external_addr 
set skip on lo    
block in log all  
pass in on $my_int proto tcp from any to any port 1723 keep state
pass in on $my_int proto tcp from any to any port 443 keep state
pass in quick on $my_int proto icmp all keep state
pass in proto gre all keep state
pass in from any to $internal_net
pass in from $internal_net to any
pass out proto { gre, tcp, udp, icmp } all keep state
```

## Log configuration.
- Edit /etc/syslog.conf
```
!mpd
	*.*                                             /var/log/mpd.log
```
- Touch /var/log/mpd.log
- Restart syslog

# Usage
- **/usr/local/etc/rc.d/mpd5 start**

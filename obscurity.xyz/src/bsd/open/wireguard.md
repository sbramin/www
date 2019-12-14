# wireguard

[wireguard](https://www.wireguard.com/) (wg) is a modern vpn protocol, using the latest class of encryption algorithms while at the same time promising speed and a small code base.

modern crypto and lean code are also tenants of openbsd, thus it was a no brainer to migrate my router from openvpn over to wireguard.

## my setup

a collection of devices, both wired and wireless, that are nat'd through my router out via my vpn provider [azire\*](https://www.azirevpn.com/wireguard) and out to the internet using the wg-quick for starting wg.

```
[wifi-ap] --------
			   	 |-- [switch] -- [router (wg-client)] -- [isp] -- [endpoint wg-server)] -- {the internet}
[wired-clients] --
```

## prerequisits

### install

install wireguard-go (the client/server) and wireguard-tools.

```
~ doas pkg_add -v wireguard_tools wireguard-go
```

### pf.conf

This is the simplest pf filei _/etc/pf.conf_, my wan interface is **re0**, my switch plugs into lan **re2** and wg will create the vpn interface **tun0**. The two important sections are **max-mss 1360** (more below on this) and the **nat_to** line on the vpn interface.

```
lan = "re2"
wan = "re0"
vpn = "tun0"

table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16 \
      172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
	  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 \
	  203.0.113.0/24 }

set block-policy drop
set loginterface egress
set skip on { lo0 }

match in all scrub (no-df random-id max-mss 1360)
match out on $vpn inet from !(egress:network) to any nat-to ($vpn:0)

antispoof quick for { egress $lan $vpn }

block in quick on egress from <martians> to any
block return out quick on egress from any to <martians>
block all

pass out quick inet

pass in on { $lan } inet
```

### wg config

if you are connecting to a wg endpoint you run, you will need to generate your public/private keys and use the relevant address/endpoint/dns settings, this is well documented in [jasper's](https://blog.jasper.la/wireguard-on-openbsd.html) article, oterhwise your provider should provide these eg. [azire's](https://www.azirevpn.com/cfg/wireguard) config generator. this file is stored in _/etc/wireguard/vpn.conf_ (can name it anything you like, needs to end in .conf).

```
[Interface]
PrivateKey = 5L+k............................+QOvXlU=
DNS = 192.211.0.2
Address = 10.0.0.42/19

[Peer]
PublicKey = xuxo8g............................qxTdfQ=
Endpoint = 149.248.160.60:51820
AllowedIPs = 0.0.0.0/0

```

### network settings

this was the missing piece to get wg fully working, by default my outbound interface had an mtu of _1500_ and _wg-quick_ creates a tun device with an mtu of 80 bytes fewer, _1420_. i wasn't able to find particular documentation as to why this is 80 bytes, instead of the usual 40/60 bytes a tcp header takes. at this point the router was able to use the wg connection perfectly, but the nat'd clients had issues connecting to particular sites. it was also necessary to set the _max_mss_ another 80 bytes lower than the tun0 mtu and _net.inet.tcp.mssdflt_ to the same.

```
~ doas sysctl net.inet.tcp.mssdflt=1360
# echo 'net.inet.tcp.mssdflt=1360' >> /etc/sysctl.conf
```

## running

doubtless this could be improved on, but currently i start wg manually when my router boots. this, and the nat'ing on the vpn interface mean its impossible for clients to connect to the internet without the vpn being up. as my router is on a ups and only reboots when a kernel patch requires it, it's a compromise i can live with. run wg-quick (please replace **vpn** with whatever you named your wg _.conf_ file.) and reload pf rules.

```
~ doas wg-quick up vpn
~ doas pfctl -f /etc/pf.conf
```

`*` i am not affiliated with azire in anyway, just a happy customer, and they are supporters of the technology. doubtless other providers will also offer wg if they don't already, or you can run your own as described in jasper's article.

# Komunikacijski protokoli sk12
# Splošno
### Gesla:

ESXi:
USER:sk12
GESLO:M@rtinnino12

VYOS(router12):
USER:sk12
GESLO:martinnino12

user:
username: sk12
password: martinnino12

masina_za_raft_2:
USER:test
GESLO:martinnino12

raft-masina3
USER:test
GESLO:martinnino12

### Splošni ukazi
Spremembe z: 
* configure 
* \<ukazi\>
* commit
* save
* exit

show interfaces
show configuration - izpiše vse kar je nastavljeno
ip route
ipv6 route

# Skica omrežja

![](https://i.imgur.com/HQ8eqnU.png)

> Omrežje vsebuje 3 segmente:
    1. eth1 - dmz (strežniki) 192.168.12.1/24,  2001:1470:fffd:ad::1/64
    2. eth2 - users (uporabniki) 10.12.0.1/24, 2001:1470:fffd:ae::1/64
    3. eth3 - ipv6only 2001:1470:fffd:af::1/64
  Strežniki (4) so winserver, rest, masina_za_raft_2, raft-masina3
  Uporabnika (2) sta user, uporabnik.
  Omrežje ima WireGuard VPN.


--------------------
# Trivialni del
#### Uporabnik:
Ukaz ki nastavi novega admin uporabnika.
```
set system login user sk12 full-name "skupina 12"
set system login user sk12 authentication plaintext-password martinnino12 
set system login user sk12 level admin
delete system login user vyos
```

#### IPv4:
IPv4 povezava z LRK routerjem.
```
set interface ethernet eth0 address 88.200.24.242/24
set protocols static route 0.0.0.0/0 next-hop 88.200.24.1
set interfaces ethernet eth0 description 'LRK router'
```

#### IPv6:
IPv6 povezava z LRK routerjem.
```
set interface ethernet eth0 address 2001:1470:fffd:ac::2/64
set protocols static route ::/0 next-hop 2001:1470:fffd:ac::1
```

#### Nastavljanje segmentov za uporabnike, strežnike in ipv6 segment
```
set interface ethernet eth2 address 10.12.0.1/24
set interface ethernet eth2 address 2001:1470:fffd:ae::1/64
set interface ethernet eth1 address 2001:1470:fffd:ad::1/64
set interface ethernet eth1 address 192.168.12.1/24
set interface ethernet eth3 address 2001:1470:fffd:af::1/64
```
> eth0 - LRK router
eth1 - strežniki (dmz) [winserver, rest, masina_za_raft_2, raft-masina3]
eth2 - uporabniki [user, uporabnik]
eth3 - ipv6 only

--------------------
#### HOST-NAME, DOMAIN:
Nastavitev imena gostitelja in domene.
```
set system host-name router12
set system domain-name skupina12
```
#### DNS
Statična DNS nastavitev.
```
set system name-server 193.2.1.6
set system name-server 8.8.8.8
```
#### NTP
Nastavitev NTP strežnika.
``` set system ntp server ntp1.arnes.si ```


#### SSH povezave
Vklopljen SSH na Vyos usmerjevalniku na portu 1191. SSH možen tudi na ostalih mašinah.
```
set service ssh port 1191
```
> povezava preko programa Putty 88.200.24.242
port: 1191
povezava na rest-api/masina_1_raft server na 192.168.12.18
port : 1192
povezava na masino_3_raft 192.168.12.21
port : 1194
povezava na masina_za_raft_2 192.168.12.20
port: 1195

--------------------
# DHCP, NAT, SLAAC, DHCPv6

### DHCP-server
Nastavitev DHCP-ja za segmenta dmz in uporabniki.
```
// serverji

set service dhcp-server shared-network-name LAN subnet 192.168.12.0/24 
edit service dhcp-server shared-network-name LAN subnet 192.168.12.0/24
set default-router 192.168.12.1
set range 0 start 192.168.12.10
set range 0 stop 192.168.12.250
set domain-name 'skupina12'
set dns-server 9.9.9.9
set lease '86400'
set 

// uporabniki

set service dhcp-server shared-network-name LAN1 subnet 10.12.0.0/24 default-router '10.12.0.1'
set service dhcp-server shared-network-name LAN1 subnet 10.12.0.0/24 dns-server 8.8.8.8
set service dhcp-server shared-network-name LAN1 subnet 10.12.0.0/24 domain-name 'skupina12'
set service dhcp-server shared-network-name LAN1 subnet 10.12.0.0/24 range 0 start 10.12.0.10
set service dhcp-server shared-network-name LAN1 subnet 10.12.0.0/24 range 0 stop '10.12.0.254'
```
> show dhcp server leases

#### NAT
Nastavitev NAT servisa za segmenta dmz in uporabniki.
```
// serverji

set nat source rule 1 outbound-interface 'eth0'
set nat source rule 1 source address '192.168.12.0/24'
set nat source rule 1 translation address masquerade

// uporabniki

set nat source rule 2 outbound-interface 'eth0'
set nat source rule 2 source address '10.12.0.0/24'
set nat source rule 2 translation address masquerade
```

#### SLAAC
SLAAC dodeljevanje implementirano na segmentu uporabniki (eth2).
```
set interfaces ethernet eth2 ipv6 router-advert send-advert true
set interfaces ethernet eth2 ipv6 router-advert 
set interfaces ethernet eth2 ipv6 router-advert prefix 2001:1470:fffd:ae::/64
set interfaces ethernet eth2 ipv6 router-advert max-interval 10
set interfaces ethernet eth2 ipv6 router-advert name-server '2620:0:ccc::2'
```
> dhclient -6 \<interface\>
> show dhcpv6 server leases

#### DHCPv6
DHCPv6 dodeljevanje implementirano na segmentu (eth1).
```
edit interfaces ethernet eth1 
set ipv6 router-advert send-advert true
set ipv6 router-advert max-interval 10
set ipv6 router-advert prefix 2001:1470:fffd:ad::/64 
set ipv6 router-advert prefix 2001:1470:fffd:ad::/64  autonomous-flag true
set ipv6 router-advert other-config-flag true
set ipv6 router-advert default-preference high
set ipv6 router-advert managed-flag true
-----------------------------------
edit service dhcpv6-server shared-network-name lan subnet 2001:1470:fffd::ad/64
set name-server '2620:0:ccc::2'
set name-server '2620:0:ccd::2'
set address-range start 2001:1470:fffd::ad::2
set address-range start 2001:1470:fffd::ad::2 stop 
2001:1470:fffd::ad::100
```
--------------------
# Active Directory, WireGuard

### Active Directory na Windows 2019 serverju

Na virtualki winserver implementiran je AD - Microsoft Active Directory.

domain-name: sk12.local
NETBIOS name: SK12
geslo: M@rtinnino12

1. powershell: ldp
2. LDP: 
2.1 Connection - 192.168.12.16:389
2.2 Bind - uporabnik spodaj

**Nov user** 
    
    User: Janez JN. Novak
    logon name: jnovak@sk12.local
    Pass: M@rtinnino12

--------------------

### VPN WireGuard config
```
generate wireguard keypair
show wireguard privkey 
show wireguard pubkey
configure
set interfaces wireguard wg01 address '172.16.100.1/24'
set interfaces wireguard wg01 port '50100'
set interfaces wireguard wg01 peer CLIENT1 allowed-ips '172.16.100.2/32' 
set interfaces wireguard wg01 peer CLIENT1 persistent-keepalive '15'
set interfaces wireguard wg01 peer CLIENT1 pubkey 'cL25yajtyNu6qUx5pcAjLaw+cFv93JMz0UeeFsBqgxs='
set protocols static interface-route '172.16.100.0/24' next-hop-interface wg01


//nastavi nat
set nat source rule 4 outbound-interface 'eth0'
set nat source rule 4 source address '172.16.100.0/24'
set nat source rule 4 translation address masquerade
```

\
**Ubuntu client /etc/wireguard/wg01.conf**
```
[Interface]
PrivateKey = cAk7giMJscRyP3+6YhUjE7soy54Et1A0TlnloiHrOF0=
Address = 172.16.100.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = Ya7dYf390xXih+Q85qTOpHxVKFLx4BdUzYhfx5auD1w=
AllowedIPs = 0.0.0.0/0
Endpoint = 88.200.24.242:50100
PersistentKeepalive = 15
```

--------------------
# REST, SNMP, Cacti

### REST-API

username:test
geslo: martinnino12

Ssh -p 1192 test@88.200.24.242

Rest-api je implementiran na sk12-rest serverju na ip-naslovu 192.168.12.18, teče pa na portu 8080 in na 8081, kjer je kriptirana povezava z **ssl** in uporabljen je tudi **http/2**, vzpostavljena je je povezava s podatkovno bazo, vendar ni implementiran content-negotiation.

> Postman za testiranje HTTP metod vzpostavljen na "masina_za_raft_2" v direktoriju **/opt/Postman**

Primeri REST zahtevkov:
1. GET http://192.168.12.18:8080/v1/uporabniki
2. POST http://192.168.12.18:8080/v1/uporabniki
2.1 body: {"davcna":1235453}

![](https://i.imgur.com/xv1EhpG.png)



--------------------

### SNMP


sudo apt-get purge maria*
sudo dpkg -l | grep mariadb 

sudo apt-get --purge remove php-common
sudo apt-get install php-common php-mysql php-cli

sudo apt install -y apache2 php-mysql libapache2-mod-php

### Cacti

Cacti vizualizacija SNMP podatkov je narejena z veliko pomočjo naslednje [spletne strani](https://www.itzgeek.com/post/how-to-install-cacti-on-ubuntu-20-04/)
Orodje Cacti je implementirano na "masina_za_raft_2"
Dostopno je na povezavi 192.168.12.18/cacti/
Uporabniško ime: admin
Geslo: M@rtinnino12

Dostopna je vizualizacija Load averagea, Memory usagea, števila procesov, števila prijavljenih uporabnikov...

![](https://i.imgur.com/9zO7k2w.jpg)

--------------------
# RAFT

### RAFT etcd
Kar se tiče instaliranja etcd-ja, najprej je bilo treba instalirati go in sicer iz urande strani golang.org, potem pa sva klonirala etcd-io iz githuba, zadnjo master verzijo. 
```
$ git clone https://github.com/etcd-io/etcd.git
$ cd etcd
$ ./build
```
S tem korakom sva instalirala etcd na vseh 3 mašinah, potem je bilo treba kreirati cluster in mašine med seboj povezati, to je bilo preprosto, le z nekaj ukazi, ki sva jih zapakirala v skripto na vsakem računalniku da je testiranje olajšano.
```
TOKEN=token-01
CLUSTER_STATE=new
NAME_1=machine-1
NAME_2=machine-2
NAME_3=machine-3
HOST_1=192.168.12.18
HOST_2=192.168.12.20
HOST_3=192.168.12.21
CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380

# For machine 1
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For machine 2
THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For machine 3
THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

Potem le še poženemo nekaj ukazov, da se na etcd povezemo s etcdctl:
```
export ETCDCTL_API=3
HOST_1=192.168.12.18
HOST_2=192.168.12.20
HOST_3=192.168.12.21
ENDPOINTS=$HOST_1:2379,$HOST_2:2379,$HOST_3:2379

etcdctl --endpoints=$ENDPOINTS member list
```

--------------------
# Firewall

### Firewall

V omrežju je implementiran preprost požarni zid.
V dmz segmentu, kjer so strežniki, je varnostna politika takšna da strežniki ne morejo dostopati navzven do ostalih interfaceov omrežja (npr. do uporabnikov) in prav tako do spleta. Tako bi pri morebitnem vdoru obvarovali vsaj uporabniški segment. Sicer se lahko serverji med seboj vidijo in tudi pingajo.
V internal segmentu nismo kaj posebej omejevali, uporabniki lahko dostopajo v do ostalih omrezij in tudi do spleta imajo dostop. 
Kar se tiče wg segmenta prav tako nismo omejevali, ker je wireguard že sam po sebi varen.


```
set firewall all-ping 'enable'
set firewall broadcast-ping 'disable'
set firewall config-trap 'disable'
set firewall group address-group REST-v4 address '192.168.12.18'
set firewall group address-group WAN-v4 address '88.200.24.242'
set firewall group ipv6-address-group WAN-v6 address '2001:1470:fffd:ac::2'
set firewall group ipv6-network-group DMZ-v6 network '2001:1470:fffd:ad::/64'
set firewall group ipv6-network-group INTERNAL-v6 network '2001:1470:fffd:ae::/64'
set firewall group ipv6-network-group ONLY-v6 network '2001:1470:fffd:af::/64'
set firewall group network-group DMZ-v4 network '192.168.12.0/24'
set firewall group network-group INTERNAL-v4 network '10.12.0.0/24'
set firewall group network-group VPN-v4 network '10.12.1.0/24'
set firewall group port-group DNS port '53'
set firewall group port-group DNS port '853'
set firewall group port-group NTP port '123'
set firewall group port-group SSH port '22'
set firewall group port-group WEB port '80'
set firewall group port-group WEB port '443'
set firewall name DMZ-IN-v4 default-action 'accept'
set firewall name DMZ-IN-v4 rule 10 action 'accept'
set firewall name DMZ-IN-v4 rule 10 destination group port-group 'SSH'
set firewall name DMZ-IN-v4 rule 11 action 'accept'
set firewall name DMZ-IN-v4 rule 11 destination group port-group 'DNS'
set firewall name DMZ-IN-v4 rule 12 action 'accept'
set firewall name DMZ-IN-v4 rule 12 destination group port-group 'NTP'
set firewall name DMZ-IN-v4 rule 20 action 'accept'
set firewall name DMZ-IN-v4 rule 20 destination group address-group 'REST-v4'
set firewall name DMZ-IN-v4 rule 20 destination group port-group 'WEB'
set firewall name DMZ-LOCAL-v4 default-action 'accept'
set firewall name DMZ-LOCAL-v4 rule 10 action 'accept'
set firewall name DMZ-LOCAL-v4 rule 10 destination group port-group 'DNS'
set firewall name DMZ-LOCAL-v4 rule 11 action 'accept'
set firewall name DMZ-LOCAL-v4 rule 11 destination group port-group 'NTP'
set firewall name DMZ-OUT-v4 default-action 'accept'   !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
set firewall name DMZ-OUT-v4 rule 10 action 'accept'
set firewall name DMZ-OUT-v4 rule 10 destination group port-group 'SSH'
set firewall name DMZ-OUT-v4 rule 11 action 'accept'
set firewall name DMZ-OUT-v4 rule 11 destination group port-group 'NTP'
set firewall name DMZ-OUT-v4 rule 20 action 'accept'
set firewall name DMZ-OUT-v4 rule 20 destination group address-group 'REST-v4'
set firewall name DMZ-OUT-v4 rule 20 destination group port-group 'WEB'
```

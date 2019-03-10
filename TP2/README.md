TP2 Maxim Germain
============


I. Mise en place du lab
------------

###1. Création des VMs et adressage IP###

Création des VM centos7

Création des cartes:

    #vboxnet0 (ip: 10.2.12.1/29)
    #vboxnet1 (ip:10.2.2.1/24)
    #vboxnet2(ip:10.2.1.1/24)

Configuration des adresses ip statique pour chaque VM

Exemple de configuration : net2 de router2

    #TYPE=Ethernet
    #BOOTPROTO=static
    #DEVICE=enp0s8
    #ONBOOT=yes
    #IPADDR=10.2.2.254
    #NETMASK=255.255.255.0

Par la suite, connexion SSH à chacune des VM

Exemple de connexion à client1

    #ssh ccna@10.2.1.10

Définition du nom de domaine

    #echo 'client1' | sudo tee /etc/hostname

Edition du fichier hosts

    #127.0.0.1    localhost
    #::1          localhost
    #10.2.1.1     pc1
    #10.2.2.1     pc2
    #10.2.12.1    pc3
    #10.2.1.10    client1
    #10.2.2.10    server1
    #10.2.1.254   router1.1
    #10.2.12.2    router1.2
    #10.2.2.254   router2.1
    #10.2.12.3    router2.2

###1. Routage statique###

Ajout de l’ip forwarding dans le fichier de config /etc/sysctl.conf sur router1 et router2

    #net.ipv4.conf.all.forwarding=1
    #reboot

Ajout d’une route de router1 vers net2 en utilisant net12 de router2

    #sudo ip route add 10.2.2.0/24 via 10.2.12.3 dev enp0s8


Ajout d’une route de router2 vers net1 en utilisant net12 de router1

    #sudo ip route add 10.2.1.0/24 via 10.2.12.2 dev enp0s8


Ajout d’une route de client1 vers net2

    #sudo ip route add 10.2.2.0/24 via 10.2.1.254 dev enp0s3


Ajout d’une route de server1 vers net1

    #sudo ip route add 10.2.1.0/24 via 10.2.2.254 dev enp0s3

Test pour voir si les routes fonctionnent

    #ping client1 vers server1

    #[ccna@client1 ~]$ ping server1
    #PING server1 (10.2.2.10) 56(84) bytes of data.
    #64 bytes from server1 (10.2.2.10): icmp_seq=319 ttl=62 time=2.33 ms
    #64 bytes from server1 (10.2.2.10): icmp_seq=320 ttl=62 time=2.24 ms
    #64 bytes from server1 (10.2.2.10): icmp_seq=321 ttl=62 time=2.03 ms
    #64 bytes from server1 (10.2.2.10): icmp_seq=322 ttl=62 time=1.99 ms
    #64 bytes from server1 (10.2.2.10): icmp_seq=323 ttl=62 time=2.22 ms
    #64 bytes from server1 (10.2.2.10): icmp_seq=324 ttl=62 time=1.88 msping server1 vers client1
    #[ccna@server1 ~]$ ping client1
    #PING client1 (10.2.1.10) 56(84) bytes of data.
    #64 bytes from client1 (10.2.1.10): icmp_seq=1 ttl=62 time=1.05 ms
    #64 bytes from client1 (10.2.1.10): icmp_seq=2 ttl=62 time=2.27 ms
    #64 bytes from client1 (10.2.1.10): icmp_seq=3 ttl=62 time=2.23 ms
    #64 bytes from client1 (10.2.1.10): icmp_seq=4 ttl=62 time=2.03 ms
    #64 bytes from client1 (10.2.1.10): icmp_seq=5 ttl=62 time=2.10 ms


ping router1 vers server1

    #[ccna@router1 ~]$ ping server1
    #PING server1 (10.2.2.10) 56(84) bytes of data.
    #^C
    #--- server1 ping statistics ---
    #10 packets transmitted, 0 received, 100% packet loss, time 9003ms

Il ne fonctionne pas car il n’a aucune route vers server1


###3. Visualisation du routage avec Wireshark###

Sur client1

    #ping server1

sur router2

capture net12  

    #sudo tcpdump -i enp0s8 -w ping1.pcap


capture net2 

    #sudo tcpdump -i enp0s3 -w ping2.pcap

Copie du fichier ping.pcap

    #scp /home/ping1.pcap ccna@10.2.12.3:/home/maxim/Bureau

La seul choses qui change entre ces deux captures, ce sont les adresses mac des machines qui envoient le ping.



II. NAT et services d'infra
============

###1. Mise en place du NAT###

Vérification que l’on peux se connecter à internet avec le router1


    #[ccna@router1 ~]$ dig cloudflare.com
    #; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> cloudflare.com
    #;; global options: +cmd
    #;; Got answer:
    #;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43962
    #;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
    #;; OPT PSEUDOSECTION:
    #; EDNS: version: 0, flags:; udp: 65494
    #;; QUESTION SECTION:
    #;cloudflare.com.
    #IN A
    #;; ANSWER SECTION:
    #cloudflare.com.   179  IN   A 198.41.214.162
    #cloudflare.com.   179  IN   A 198.41.215.162
    #;; Query time: 35 msec
    #;; SERVER: 10.0.2.3#53(10.0.2.3)
    #;; WHEN: lun. mars 04 11:02:25 EST 2019
    #;; MSG SIZE rcvd: 75


ajout sur le nat et net1 de `ZONE=public`

ajout de `ZONE=internal` sur net12

redémarrage des interfaces

    #sudo ifdown enp0s3
    #sudo ifup enp0s3
    #sudo ifdown enp0s8
    #sudo ifup enp0s8
    #sudo ifdown enp0s9
    #sudo ifup enp0s9

Activation du Nat dans la zone public

    #sudo firewall-cmd --add-masquerade --zone=public --permanent
    #firewall-cmd --reload

ajout de la route par default sur router2 et client1

Dans `/etc/sysconfig/network-script/ifcfg-enp0s3`

    #GATEWAY= 10.2.12.2

Test curl

    #[ccna@client1 ~]$ curl google.com
    #<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
    #<TITLE>301 Moved</TITLE></HEAD><BODY>
    #<H1>301 Moved</H1>
    #The document has moved
    #<A HREF="http://www.google.com/">here</A>.
    #</BODY></HTML>


###2. DHCP server###
Changement du nom de la machine: `/etc/hostname`

    #Installation dhcp : sudo yum install -y dhcp

Copie dans le fichier de config `/etc/dhcpd` du fichier de configuration `dhcpd.conf`


Lancement du service dhcpd

    #sudo systemctl start dhcpd


Création de la nouvelle vm client2 avec carte host-only net1 comme serveur-dhcp


Modification dans `/etc/sysconfig/network-scripts/ifcfg-enp0s8`

    #NAME=enp0s8
    #DEVICE=enp0s8
    #BOOTPROTO=dhcp
    #ONBOOT=yes

Puis la commande `dhclient` sur cette même machine


Vérification sur client2 avec traceroute

    #default via 10.2.1.254 dev enp0s8 proto static metric 100

On remarque que la route par défaut est celle défini dans le fichier de configuration


On utilise la commande  `ip a` pour vérifier que l’on a bien une nouvelle ip

    #enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    #link/ether 08:00:27:e1:8e:46 brd ff:ff:ff:ff:ff:ff
    #inet 10.2.1.51/24 brd 10.2.1.255 scope global dynamic enp0s8
    #valid_lft 446sec preferred_lft 446sec
    #inet 10.2.1.52/24 brd 10.2.1.255 scope global secondary dynamic enp0s8
    #valid_lft 364sec preferred_lft 364sec
    #inet6 fe80::a00:27ff:fee1:8e46/64 scope link
    #valid_lft forever preferred_lft forever


###3. NTP server###

Ajout du pool de serveur français dans le fichier chrony.conf sur router1

    #server 0.fr.pool.ntp.org
    #server 1.fr.pool.ntp.org
    #server 2.fr.pool.ntp.org
    #server 3.fr.pool.ntp.org


Ouverture du port pour NTP

    #firewall-cmd --zone=public --permanent --add-port=123/udp


Démarrage du service chronyd

    #sudo systemctl start chronyd


Configuration pour router1 et server1

    ## Server to syncrhonize with
    #server router1.1 prefer
    #initstepslew 20 router1.1 prefer
    ## Record the rate at which the system clock gains/losses time.
    #driftfile /var/lib/chrony/drift
    # Allow the system clock to be stepped in the first three updates
    ## if its offset is larger than 1 second.
    #makestep 1.0 3
    ## Enable kernel synchronization of the real-time clock (RTC).rtcsync
    ## Serve time even if not synchronized to a time source.
    #local stratum 10
    ## Specify file containing keys for NTP authentication.
    #keyfile /etc/chrony.keys
    ## Specify directory for log files.
    #logdir /var/log/chrony
    ## Select which information is logged.
    #log measurements statistics tracking


Ouverture du port pour NTP

    #firewall-cmd --zone=public --permanent --add-port=123/UDP

chronyc tracking + sources

    #MS Name/IP address
    #Stratum Poll Reach LastRx Last sample
    #^+ ks3370497.kimsufi.com
    #2 10 377 711 -355us[ +227us] +/- 52ms
    #^- 151.80.196.50
    #3 10 377 119
    #-11ms[ -11ms] +/- 184ms
    #^* server.bertold.org
    #2 9 377 172 -639us[ -557us] +/- 42ms
    #^+ eva.aplu.fr
    #2 7 377
    #36 -177us[ -177us] +/- 41ms
    
    #[ccna@router2 ~]$ chronyc tracking
    #Reference ID : 5E170250 (server.bertold.org)
    #Stratum
    #: 3
    #Ref time (UTC) : Sun Mar 10 20:07:34 2019
    #System time : 510760.968750000 seconds slow of NTP time
    #Last offset
    #: +0.000081706 seconds
    #RMS offset
    #: 11011.600585938 seconds
    #Frequency
    #: 1.299 ppm slow
    #Residual freq : +0.013 ppm
    #Skew
    #: 0.240 ppm
    #Root delay
    #: 0.033222210 seconds
    #Root dispersion : 0.018553471 seconds
    #Update interval : 476.7 seconds
    #Leap status : Normal4. 


###4. Web server###

Installation des paquet pour le serveur web

    #sudo yum install -y epel-release
    #sudo yum install -y nginx

Ouverture du port 80 en tcp

    #sudo firewall-cmd --add-port=80/tcp --permanent
    #sucess

Lancement du serveur web

    #sudo systemctl start nginx

Vérification lancement

    #sudo systemctl status nginx

    #● nginx.service - The nginx HTTP and reverse proxy server
    #Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    #Active: active (running) since dim. 2019-03-10 16:23:11 EDT; 1min 48s ago
    #Process: 1087 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
    #Process: 1085 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)


    #sudo ss -altnp4

    #State Recv-Q Send-Q Local Address:Port
    #Peer Address:Port
    #LISTEN
    #0
    #128
    #*:80
    #*:*
    #users:(("nginx",pid=1091,fd=6),("nginx",pid=1089,fd=6))
    #LISTEN
    #0
    #128
    #*:22
    #*:*
    #users:(("sshd",pid=938,fd=3))
    #LISTEN
    #0
    #100
    #127.0.0.1:25
    #*:*
    #users:(("master",pid=1037,fd=13))

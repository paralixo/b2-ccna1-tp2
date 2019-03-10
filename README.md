# TP 2 - Routage statique et services d'infra
## I. Mise en place du lab
### 1. Création des VMs et adressage IP

On clone notre patron de VM 4 fois. On créer 3 réseaux host-only qu'on attribue correctement aux VMs (voir TP).

#### Définition des IPs statiques

On exécute les commandes suivante avec X représentant le numéro de l'interface. 

	$ sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0sX
	
	$ sudo cat /etc/sysconfig/network-scripts/ifcfg-enp0sX
	TYPE=Ethernet
	BOOTPROTO=static
	NAME=enp0s8
	DEVICE=enp0s8
	ONBOOT=yes
	IPADDR=10.2.2.10
	NETMASK=255.255.255.0
	ZONE=public
	
	$ sudo ifdown enp0sX
	$ sudo ifup enp0sX

#### Connexion ssh depuis Powershell

Maintenant que les IPs sont statiques nous pouvons nous connecter en SSH à nos VMs (pour plus de confort). On ouvre donc notre Powershell : 

	PS C:\Users\flori>ssh florian@10.2.1.10

#### Modification du hostname

On va réaliser une modification permanente du hostname (à répéter pour les 4VMs en adaptant le nom) :

	$ echo 'client1.net1.b2' | sudo tee /etc/hostname

#### Modification fichier hosts

On modifie le fichier hosts pour ne plus avoir à taper des adresses IPs (plus simple de retenir un nom qu'une suite de chiffres), à effectuer sur toutes les VMs : 

	$ sudo vi /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	10.2.1.10   client1 client1.net1.b2
	10.2.2.10   server1 server1.net2.b2
	10.2.1.254  router1 router1.net12.b2
	10.2.12.2   router1 router1.net12.b2
	10.2.2.254  router2 router2.net12.b2
	10.2.12.3   router2 router2.net12.b2

### Routage statique
On commence par activer l'IPv4 Forwarding sur `router1`et `router2` (pour leur permettre de traiter des paquets IP qui ne leur sont pas destinés) : 

	$ sudo sysctl -w net.ipv4.conf.all.forwarding=1

Modification permanente dans le fichier `/etc/sysctl.conf`.

Sur `router1`, on ajoute une route vers `net2` : 

	sudo ip route add 10.2.2.0/24 via 10.2.12.3 dev enp0s9

Sur `router2`, on ajoute une route vers `net1` : 

	sudo ip route add 10.2.1.0/24 via 10.2.12.2 dev enp0s9
	
Sur `client1`, on ajoute une route vers `net2` : 

	sudo ip route add 10.2.2.0/24 via 10.2.1.254 dev enp0s8

Sur `server1`, on ajoute une route vers `net1` : 

	sudo ip route add 10.2.1.0/24 via 10.2.2.254 dev enp0s8

Les modifications des routes ci-dessus sont temporaires, si on veut une modification permanente on écrit dans `/etc/sysconfig/network-scripts/route-<interface> `.

On va maintenant tester le bon fonctionnement de notre réseau. On va ping `client1` et `server1` qui ne se connaissent pas : 

ping client1 & serve1

	[florian@client1 ~]$ ping server1
	PING server1 (10.2.2.10) 56(84) bytes of data.
	64 bytes from server1 (10.2.2.10): icmp_seq=1 ttl=62 time=1.32 ms
	64 bytes from server1 (10.2.2.10): icmp_seq=2 ttl=62 time=1.24 ms
	64 bytes from server1 (10.2.2.10): icmp_seq=3 ttl=62 time=1.34 ms
	^C
	--- server1 ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 2011ms
	rtt min/avg/max/mdev = 1.246/1.304/1.346/0.059 ms
	
ping router1 & server1

	[florian@router1 ~]$ ping server1
	PING server1 (10.2.2.10) 56(84) bytes of data.
	^C
	--- server1 ping statistics ---
	3 packets transmitted, 0 received, 100% packet loss, time 2006ms

Pourquoi ? -> Le router1 connais server1 mais server1 ne connais pas router1 ducoup il ne peut pas lui répondre


## 3. Visualisation du routage avec Wireshark

### net12
On remarque 8 paquets : 6 pings/pongs et 2 paquets ARP

### net2
On remarque plus de paquets que précedement : les pings/pongs, les 2 paquets ARP mais aussi un paquet SSH (dû à la connexion SSH) et un paquet TCP (je ne sais pas pourquoi)

Les pings/pongs sont normaux. Et on remarque que la table ARP "demande son chemin" tout du long du processus (dans net12 et net2).

# II. NAT et services d'infra

## 1. Mise en place du NAT

On vérifie que la NAT fonctionne bien sur router1 avec un `curl www.google.com`
Ensuite : 

	$ sudo vi /etc/sysconfig/network-scripts/ifcfg-enp0sx

On mets l'interface NAT en `ZONE=public` et les deux autres en `ZONE=trusted`.
On fait la même chose sur le `router2` (il n'y a pas de NAT).
>On oublie pas de redémarrer les interfaces modifiés.

	$ sudo firewall-cmd --add-masquerade --zone=public --permanent
	$ sudo systemctl restart firewalld

On ajoute une route par défaut sur client1 et router2 : 
client1

	$ sudo ip route add default via 10.2.1.254 dev enp0s8

router2

	$ sudo ip route add default via 10.2.12.2 dev enp0s9

server1

	sudo ip route add default via 10.2.2.254 dev enp0s8

On `curl www.google.com` et `ping 8.8.8.8` sur toutes les VMs pour vérifier que tout fonctionne.


## 4. Web server

Sur `server1`, on installe les dépôts EPEL et NGINX :

	$ sudo yum install -y epel-release
	$ `sudo yum install -y nginx`

On autorise la connexion sur le port 80 en TCP : 

	$ sudo firewall-cmd --add-port=80/tcp --permanent
	$ sudo systemctl restart firewalld

On lance le serveur web et on vérifie qu'il est bien lancé : 

	$ sudo systemctl start nginx
	$ sudo systemctl status nginx

On doit voir écrit en vert `active (running)`. Sinon on peut vérifier avec la commande : 

	$ sudo ss -altnp4
	State       Recv-Q Send-Q         Local Address:Port                        Peer Address:Port
	LISTEN      0      128                        *:80                                     *:*                   users:(("nginx",pid=4330,fd=6),("nginx",pid=4329,fd=6),("nginx",pid=4328,fd=6))
	LISTEN      0      128                        *:22                                     *:*                   users:(("sshd",pid=3253,fd=3))
	LISTEN      0      100                127.0.0.1:25                                     *:*                   users:(("master",pid=3471,fd=13))

A la première ligne, on voit que 'nginx' écoute sur le port 80 de server1. Donc le serveur web est bien lancé !

On peut modifier la page d'accueil avec:

	$ sudo vi /usr/share/nginx/html/index.html

Et pour tester la modification de notre page : 

	[florian@client1 ~]$ curl server1



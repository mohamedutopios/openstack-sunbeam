cellule c'est quoi ?
Un schéma de comment fonctionne la region (même controle plane ou pas) ?
RPC ? RPC vs GRPC
cinder-volume : Verrous distribués (coordination tooz : etcd, Redis)
HAProxy + Keepalived	Non	Actif / passif pour les VIP	VRRP : un seul nœud porte les adresses virtuelles	Bascule de VIP, double maître
libvirt 
OVN
cellule Nova
Ceph MON, MGR
Ceph OSD

Contrôle	Mémoire et latence disque	Workers des API (un par cœur par défaut), MariaDB, RabbitMQ. -> un par cœur par défaut ???

Calcul	Cœurs et mémoire vendables -> mémoire vendables ?

Passerelle	Débit réseau et CPU de commutation -> CPU de commutation ?

Stockage Ceph	Disques, mémoire par OSD, réseau -> OSD ?

Neutron	Placement	Déclarer la bande passante garantie (QoS) et les segments de réseaux routés ??

oslo.messaging
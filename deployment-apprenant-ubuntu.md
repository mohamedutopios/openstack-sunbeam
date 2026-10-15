# Déploiement apprenant : lab OpenStack Sunbeam sous Ubuntu (VirtualBox)

Ce pas à pas installe le lab à partir des deux OVA fournies, `Services-base.ova` et `Sunbeam-base.ova`, sur un **hôte Ubuntu 24.04 avec VirtualBox pour Linux**. Les OVA, les adresses et les contrôles sont les mêmes que sous Windows ; seules les commandes côté hôte changent.

**État :** les OVA ont été validées sur un hôte Windows (OpenStack, instance Nginx jointe en SSH et HTTP). **Ce parcours Ubuntu n’a pas encore été rejoué de bout en bout** : teste-le sur un poste avant de le distribuer.

Variante Windows et préparation des OVA par le formateur : `deployment-apprenant-windows.md` (partie A).

## Vue d’ensemble

| VM | Rôle | Carte 1 | Carte 2 : gestion `192.168.56.0/24` | Carte 3 : instances `192.168.57.0/24` |
|---|---|---|---|---|
| **Lab-Services** | Keycloak, DNS (dnsmasq), passerelle des instances | NAT | `mgmt0` = `192.168.56.20` | `provider0` = `192.168.57.2` (MTU 1442) |
| **Lab-Sunbeam** | OpenStack Sunbeam, Ceph, hyperviseur Nova | NAT | `mgmt0` = `192.168.56.10` + API `.201`, `.202`, `.203` | `provider0` sans IP (pont Open vSwitch) |
| **Hôte Ubuntu** | Navigateur, SSH, CLI | — | `vboxnet0` = `192.168.56.1` | `vboxnet1` = `192.168.57.1` |

| Nom | Adresse | Service |
|---|---|---|
| `sso.lab.test` | `192.168.56.20` | Keycloak `https://sso.lab.test:8443` |
| `cloud.lab.test` | `192.168.56.201` | Horizon et API publiques |
| `internal.lab.test` | `192.168.56.202` | API internes |
| `rgw.lab.test` | `192.168.56.203` | Stockage objet Ceph |

Fichiers fournis par le formateur, à placer par exemple dans `~/Lab-OpenStack/` :

```text
Services-base.ova
Sunbeam-base.ova
Configurer-Lab-Sunbeam.sh
ca.crt
demo/README.md
```

---

## 1. Vérifier l’hôte Ubuntu

| Ressource | Valeur |
|---|---|
| Système | Ubuntu 24.04 LTS, bureau (pour le navigateur) |
| Architecture | **Intel** x86-64 (AMD : prévenir le formateur, voir étape 10.3) |
| RAM | 32 Go minimum |
| Processeurs | 12 threads ou plus conseillés (les VM en utilisent 10) |
| Stockage | SSD, environ 300 Go libres |
| Virtualisation | Intel VT-x activé dans l’UEFI/BIOS |

```bash
nproc
free -g
df -h ~
grep -c vmx /proc/cpuinfo
```

`grep -c vmx` doit renvoyer un nombre supérieur à 0. Sinon, VT-x est désactivé dans l’UEFI/BIOS, ou l’hôte Ubuntu est lui-même une VM qui n’expose pas la virtualisation.

### Hôte Ubuntu qui est lui-même une VM

Si ta machine Ubuntu est une VM (VMware, Proxmox, Hyper-V, cloud…), l’hyperviseur qui l’héberge doit exposer **VT-x imbriqué** (`grep -c vmx` > 0). Les instances Nova tournent alors sur **trois niveaux** de virtualisation : le lab peut démarrer, mais les instances seront lentes et ce montage n’est pas validé. Préfère un Ubuntu installé directement sur le PC.

Vérifie aussi que les réseaux `192.168.56.0/24` et `192.168.57.0/24` ne sont pas déjà utilisés par l’hôte :

```bash
ip -br addr | grep -E "192\.168\.5[67]\."
```

Aucune ligne ne doit s’afficher avant la création des réseaux de l’étape 4.

## 2. Libérer VT-x pour VirtualBox

Sur Linux, KVM et VirtualBox ne peuvent pas utiliser VT-x en même temps. Ubuntu charge KVM automatiquement : selon la version du noyau, VirtualBox refuse alors de démarrer une VM avec l’erreur `VERR_VMX_IN_VMX_ROOT_MODE`.

```bash
lsmod | grep -E "^kvm"
uname -r
```

Si des modules `kvm` sont chargés, décharge-les et empêche leur chargement (aucune VM KVM/libvirt ne doit tourner) :

```bash
sudo modprobe -r kvm_intel kvm
```

```bash
printf 'blacklist kvm_intel\nblacklist kvm\n' | sudo tee /etc/modprobe.d/blacklist-kvm-virtualbox.conf
```

Cela rend KVM, libvirt, GNOME Boxes et Multipass inutilisables sur cet hôte. Pour revenir en arrière après la formation : supprime ce fichier et redémarre.

## 3. Installer VirtualBox

Installe VirtualBox depuis le dépôt Oracle, de préférence **la version validée par le formateur** (7.2.x) :

```bash
sudo apt update
sudo apt install -y wget gpg
```

```bash
wget -qO- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes -o /usr/share/keyrings/oracle-virtualbox-2016.gpg
```

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian $(. /etc/os-release; echo "$VERSION_CODENAME") contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
```

```bash
sudo apt update
sudo apt install -y virtualbox-7.2
sudo usermod -aG vboxusers "$USER"
```

**Secure Boot :** si `mokutil --sb-state` affiche `SecureBoot enabled`, l’installation demande un mot de passe pour signer les modules VirtualBox. Au redémarrage suivant, l’écran bleu **MOK Management** apparaît : choisis **Enroll MOK**, puis saisis ce mot de passe.

Ferme ta session puis rouvre-la (groupe `vboxusers`), ou redémarre. Lance ensuite **Oracle VirtualBox** depuis le menu des applications : **Aide → À propos de VirtualBox** affiche la version installée.

## 4. Créer les deux réseaux Host-only

VirtualBox pour Linux n’accepte que certaines plages pour les réseaux Host-only. Déclare d’abord les deux réseaux du lab, dans un terminal :

```bash
sudo mkdir -p /etc/vbox
echo "* 192.168.56.0/24 192.168.57.0/24" | sudo tee /etc/vbox/networks.conf
```

Si ce fichier existait déjà avec d’autres plages, ajoute-les sur la même ligne au lieu de les écraser. Sans ce fichier, VirtualBox refuse l’adresse `192.168.57.1`.

Dans VirtualBox : **Fichier → Outils → Gestionnaire de réseau**, onglet **Réseaux privés hôte**. Clique deux fois sur **Créer** : VirtualBox crée `vboxnet0` puis `vboxnet1`. Pour chacun, sélectionne-le et configure :

| Paramètre | `vboxnet0` : gestion | `vboxnet1` : instances |
|---|---|---|
| Onglet **Adaptateur** | **Configurer l’adaptateur manuellement** | **Configurer l’adaptateur manuellement** |
| Adresse IPv4 | `192.168.56.1` | `192.168.57.1` |
| Masque réseau IPv4 | `255.255.255.0` | `255.255.255.0` |
| Onglet **Serveur DHCP** | **Activer le serveur** décoché | **Activer le serveur** décoché |

Clique sur **Appliquer** après chaque réseau.

Si des réseaux Host-only existaient déjà, les numéros seront différents : **repère chaque réseau par son adresse IP, pas par son nom**.

## 5. Importer les deux OVA

**Fichier → Importer un appareil virtuel** :

1. Sélectionne `~/Lab-OpenStack/Services-base.ova`.
2. Nomme la VM **`Lab-Services`**.
3. Choisis le **dossier de base** sur le SSD.
4. **Politique d’adresse MAC** : **Inclure toutes les adresses MAC des cartes réseau**. N’utilise pas « uniquement les cartes NAT » : dans les VM, netplan reconnaît les cartes 2 et 3 par leur MAC.
5. Clique sur **Terminer**.
6. Recommence avec `Sunbeam-base.ova`, nommée **`Lab-Sunbeam`**.

**Ne démarre pas encore les VM.**

## 6. Brancher les cartes réseau

Pour **chacune des deux VM** : sélectionne-la, puis **Configuration → Réseau**.

| Carte | Mode d’accès réseau | Nom |
|---|---|---|
| Adaptateur 1 | **NAT** | — |
| Adaptateur 2 | **Réseau privé hôte** | Le réseau `192.168.56.1` (`vboxnet0`) |
| Adaptateur 3 | **Réseau privé hôte** | Le réseau `192.168.57.1` (`vboxnet1`) |

Sur les trois cartes : **Activer l’interface réseau** coché. Dans **Avancé** : **Câble branché** coché, type de carte et adresse MAC inchangés.

Sur les adaptateurs **2 et 3**, dans **Avancé**, règle **Mode promiscuité : Tout autoriser** : les IP des API OpenStack et des instances Nova utilisent d’autres MAC que celles des cartes.

Toujours dans **Avancé**, vérifie l’**adresse MAC** des cartes 2 et 3 :

| VM | Adaptateur 2 | Adaptateur 3 |
|---|---|---|
| Lab-Services | `080027560020` | `080027570002` |
| Lab-Sunbeam | `080027560010` | `080027570010` |

Si les MAC diffèrent, supprime la VM (**Supprimer → Supprimer tous les fichiers**) et réimporte-la en choisissant la bonne politique MAC.

## 7. Activer la virtualisation imbriquée de Lab-Sunbeam

VM **Lab-Sunbeam éteinte** : **Configuration → Système**.

1. Onglet **Carte mère** : vérifie **20480 Mo** de mémoire vive.
2. Onglet **Processeur** : vérifie **8** processeurs et coche **Activer VT-x/AMD-V imbriqué**.

## 8. Démarrer les VM

### 8.1 Lab-Services d’abord

Sélectionne **Lab-Services** et clique sur **Démarrer**. La console de la VM s’ouvre dans une fenêtre ; tu peux t’y connecter avec `vagrant`, ou utiliser SSH depuis un terminal de l’hôte.

Attends une minute, puis :

```bash
ssh vagrant@192.168.56.20
```

```bash
ip -br addr
ip link show provider0 | grep -o "mtu [0-9]*"
systemctl is-active docker dnsmasq
sudo docker ps --format "{{.Names}} {{.Status}}"
exit
```

| Attendu | |
|---|---|
| `mgmt0` | `192.168.56.20/24` |
| `provider0` | `192.168.57.2/24`, `mtu 1442` |
| Services | `active` et `active` |
| Conteneurs | `keycloak-keycloak-1` et `keycloak-postgres-1` démarrés |

### 8.2 Lab-Sunbeam ensuite

Sélectionne **Lab-Sunbeam** et clique sur **Démarrer**.

Si VirtualBox affiche une erreur `VERR_VMX_IN_VMX_ROOT_MODE`, KVM occupe VT-x : reprends l’étape 2.

Vérifie le moteur utilisé par VirtualBox : clic droit sur **Lab-Sunbeam → Afficher le journal**, onglet `VBox.log`, puis recherche `HMR3Init` (champ de recherche en bas de la fenêtre).

| Résultat | Signification |
|---|---|
| `VT-x w/ nested paging and unrestricted guest execution hw support`, et plus bas `VT-x nested-guest init` | ✅ Configuration correcte |
| Pas de `nested-guest init` | ❌ Case **VT-x/AMD-V imbriqué** non cochée : éteins la VM et reprends l’étape 7 |

Puis :

```bash
ssh vagrant@192.168.56.10
```

```bash
ip -br addr show mgmt0
ip -br addr show provider0
ls -l /dev/kvm
```

`mgmt0` porte `192.168.56.10/24` ; `provider0` est `UP` **sans adresse IP** (pont Open vSwitch : ne lui en ajoute pas) ; `/dev/kvm` existe.

**Après chaque démarrage, Sunbeam a besoin de 10 à 15 minutes** pour relancer Kubernetes et les services OpenStack. Pendant ce temps, Horizon peut ne pas répondre ou renvoyer une erreur 503 : attends avant de diagnostiquer.

## 9. Configurer les noms et le certificat sur l’hôte

```bash
cd ~/Lab-OpenStack
sudo bash ./Configurer-Lab-Sunbeam.sh
```

Le script ajoute les quatre noms du lab à `/etc/hosts` (avec sauvegarde) et approuve la CA du lab pour le système (`curl`, `openssl`…).

| Message du script | Signification |
|---|---|
| `OK : <nom> -> <adresse>` pour les quatre noms | ✅ `/etc/hosts` correct |
| `OK : Keycloak, port 8443 accessible.` | ✅ Lab-Services joignable |
| `OK : Horizon HTTP, port 80 accessible.` | ✅ Lab-Sunbeam joignable (après 10 à 15 min) |
| `AVERTISSEMENT : OpenStack HTTPS ... inaccessible` | **Attendu** : Horizon est encore en HTTP dans ces OVA |

Les navigateurs ont leur propre magasin de certificats. Pour Keycloak, importe `ca.crt` :

- **Firefox** : Paramètres → Vie privée et sécurité → Certificats → **Afficher les certificats** → onglet **Autorités** → **Importer**, coche « Confirmer cette AC pour identifier des sites web ».
- **Chrome / Chromium** : `chrome://settings/certificates` → **Autorités** → **Importer**, même case à cocher.

## 10. Vérifier OpenStack

Toutes les commandes se lancent dans **Lab-Sunbeam** (`ssh vagrant@192.168.56.10`), sauf 10.4.

### 10.1 Unités Juju

```bash
juju status -m openstack
juju status -m admin/openstack-machines
```

Le modèle des machines appartient à l’utilisateur Juju `admin` : `juju status -m openstack-machines` renvoie `model ... not found`. Toutes les applications doivent être `active` (quelques `blocked` ou `maintenance` sont normaux les premières minutes).

Au démarrage, des hooks échouent parfois tant que l’API Juju n’est pas joignable, et Juju ne les relance pas : l’unité reste en `error`. Par exemple, `keystone-mysql-router` en erreur bloque toute authentification (`HTTP 500`).

```bash
juju status -m openstack | grep -w error
```

Pour chaque unité listée :

```bash
juju resolved -m openstack keystone-mysql-router/0
```

Attends une minute : l’unité doit repasser `active`.

### 10.2 Ceph

```bash
sudo ceph -s | grep -E "osd:|pgs:"
```

Attendu : `1 osds: 1 up` et `289 active+clean`. Le `HEALTH_WARN` « pools have no replicas configured » est normal sur un nœud unique.

Si l’OSD est `0 up`, vérifie son lien :

```bash
sudo ls -l /var/snap/microceph/common/data/osd/ceph-1/block
```

S’il ne pointe pas vers `/dev/disk/by-path/pci-0000:00:1f.2-ata-2` :

```bash
sudo ln -sfn /dev/disk/by-path/pci-0000:00:1f.2-ata-2 /var/snap/microceph/common/data/osd/ceph-1/block
sudo systemctl reset-failed snap.microceph.osd.service
sudo snap restart microceph.osd
```

Sans Ceph, Glance ne fournit pas les images et Nova échoue (`HTTP 502`).

### 10.3 KVM imbriqué sans EPT

```bash
cat /sys/module/kvm_intel/parameters/ept
```

Attendu : `N` (fixé par `/etc/modprobe.d/kvm-intel-ept.conf`). Avec EPT actif, KVM dans VirtualBox peut corrompre la mémoire des instances, qui plantent au démarrage (`Kernel panic`, `BUG: kernel NULL pointer dereference` dans `openstack console log show`).

Si le résultat est `Y`, aucune instance ne doit tourner :

```bash
echo "options kvm_intel ept=0" | sudo tee /etc/modprobe.d/kvm-intel-ept.conf
sudo modprobe -r kvm_intel && sudo modprobe kvm_intel
```

Sur un PC **AMD**, `/sys/module/kvm_intel` n’existe pas : ce réglage n’a pas été validé, préviens le formateur.

### 10.4 MTU du réseau des instances (Lab-Services)

```bash
ip link show provider0 | grep -o "mtu [0-9]*"
```

Attendu : `mtu 1442`. Avec 1500, le NAT VirtualBox renvoie des paquets trop gros pour le réseau des instances : ping OK, mais `apt` et HTTPS bloqués dans les VM Nova. Correction : ajoute `mtu: 1442` sous `provider0` dans `/etc/netplan/60-lab.yaml`, puis `sudo netplan apply`.

### 10.5 DNS de Kubernetes

```bash
sudo k8s get dns
```

`upstream-nameservers` doit contenir **`192.168.56.20`**.

### 10.6 API OpenStack

```bash
source ~/demo-openrc
openstack token issue -c expires -f value
openstack network list
openstack subnet list
openstack router list
openstack image list
openstack flavor list
```

Les commandes doivent répondre sans erreur d’authentification, 500 ou 503.

**Ne relance jamais `sunbeam cluster bootstrap` ou `sunbeam configure` : les OVA contiennent déjà le cloud configuré.**

## 11. Se connecter depuis l’hôte

Horizon : [http://cloud.lab.test/openstack-horizon](http://cloud.lab.test/openstack-horizon)

| Champ | Valeur |
|---|---|
| Domaine | `users` |
| Utilisateur | `demo` |
| Mot de passe | Celui transmis par le formateur |

Keycloak : [https://sso.lab.test:8443/admin/](https://sso.lab.test:8443/admin/)

La connexion à Horizon utilise le compte local OpenStack : **la présence de Keycloak ne signifie pas que la fédération SSO est configurée.**

## 12. Premier test : une instance Nginx

Suis `demo/README.md`. Les commandes OpenStack sont identiques ; sur l’hôte Ubuntu, remplace les commandes PowerShell par :

| Action | Commande Ubuntu |
|---|---|
| Créer la clé | `ssh-keygen -t ed25519 -f ~/.ssh/web-key` |
| Copier la clé publique | `scp ~/.ssh/web-key.pub vagrant@192.168.56.10:~/web-key.pub` |
| Ping | `ping -c 3 192.168.57.1xx` |
| SSH | `ssh -i ~/.ssh/web-key ubuntu@192.168.57.1xx` |
| Page web | `curl http://192.168.57.1xx/` ou le navigateur |

Les règles du groupe de sécurité autorisent `192.168.57.1/32` : c’est l’adresse de `vboxnet1`, donc de l’hôte Ubuntu, comme pour Windows.

## 13. Arrêter et conserver le lab

En fin de séance :

1. Arrête les instances Nova (Horizon ou `openstack server stop`).
2. Dans **Lab-Sunbeam** : `sudo shutdown -h now`. Dans VirtualBox, attends que la VM passe à l’état **Éteinte**.
3. Dans **Lab-Services** : `sudo shutdown -h now`.

Tu peux aussi utiliser, dans la fenêtre de la VM, **Machine → Extinction par ACPI**. N’utilise pas **Éteindre** (coupure brutale) : les services OpenStack et Ceph doivent s’arrêter proprement.

Pour reprendre : **Services d’abord, Sunbeam ensuite**, puis attends 10 à 15 minutes et refais l’étape 10.1.

Après la première validation complète, crée un snapshot des **deux VM éteintes** : sélectionne la VM, bouton de menu à droite de son nom → **Instantanés** → **Prendre**, nommé par exemple `Base-importee-validee`. Pour revenir à cette base, restaure l’instantané des **deux** VM, éteintes.

---

## Dépannage rapide

| Symptôme | Cause probable | Étape |
|---|---|---|
| `VERR_VMX_IN_VMX_ROOT_MODE` au démarrage d’une VM | KVM de l’hôte occupe VT-x | 2 |
| `VERR_VMX_NO_VMX` ou `grep -c vmx` = 0 | VT-x désactivé, ou hôte virtuel sans virtualisation imbriquée | 1 |
| Adresse `192.168.57.1` refusée dans le Gestionnaire de réseau (« E_ACCESSDENIED ») | Plage absente de `/etc/vbox/networks.conf` | 4 |
| Module `vboxdrv` absent après installation | Clé MOK non enrôlée (Secure Boot) | 3 |
| `mgmt0` ou `provider0` absente, IP manquantes | MAC non conservées à l’import, ou carte sur le mauvais `vboxnet` | 5 et 6 |
| Horizon 503 ou ne répond pas juste après le démarrage | Services encore en cours de démarrage | Attendre 10 à 15 min |
| `openstack ...` : `HTTP 500` sur l’authentification | Unité `keystone-mysql-router` en erreur | 10.1 |
| Lancement d’instance : `ERROR`, `HTTP 502` Glance dans les logs Nova | OSD Ceph arrêté | 10.2 |
| Instance `ACTIVE` sans ping ni SSH, `Kernel panic` dans sa console | EPT actif | 10.3 |
| Instance : ping Internet OK, `apt`/HTTPS bloqués | MTU de `provider0` à 1500 | 10.4 |
| `juju status -m openstack-machines` : `not found` | Nom du modèle | `admin/openstack-machines` |
| Certificat Keycloak refusé dans le navigateur | CA non importée dans le navigateur | 9 |

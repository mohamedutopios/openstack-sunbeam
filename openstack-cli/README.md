# Le client OpenStack : récupérer des informations, déboguer, diagnostiquer

Ce guide montre comment utiliser la commande `openstack` pour **inventorier** un cloud, **comprendre** ce que fait une commande, et **diagnostiquer** une VM, un volume ou un refus. Il complète les guides `sunbeam-cli/` et `juju-cli/` : `sunbeam` installe et fait évoluer le cloud, `juju` en surveille les services, `openstack` le **consomme** et l’interroge.

Toutes les commandes marquées ✅ ont été **exécutées sur le lab** (client `openstack` 7.5.0, OpenStack 2024.1), avec leur sortie réelle, parfois abrégée. Elles se lancent **dans Lab-Sunbeam** :

```powershell
ssh vagrant@192.168.56.10
```

```bash
source ~/admin-openrc     # administrateur (projet admin)
source ~/demo-openrc      # utilisateur demo (projet demo)
```

Les démos qui créent des ressources les suppriment à la fin.

---

## 1. Syntaxe générale

```text
openstack [options globales] <objet> <action> [arguments] [options de sortie]
```

```bash
openstack server list
openstack network show demo-network
openstack --debug image list -f json
```

| Famille | Option | Effet |
|---|---|---|
| Aide | `openstack help server create` | Aide d’une commande |
| Aide | `openstack command list --group openstack.compute.v2` | Commandes d’un service |
| Identité | `--os-cloud NOM` | Profil d’un `clouds.yaml` (voir `sunbeam-cli/`, § 3.2) |
| Identité | `--os-project-name`, `--os-domain-name`, `--os-system-scope all` | Portée du jeton |
| Ciblage | `--os-interface public\|internal`, `--os-region-name` | Point d’accès du catalogue |
| Version | `--os-compute-api-version 2.96`, `--os-volume-api-version 3.3` | Microversion d’API |
| Débogage | `--debug` | Affiche chaque requête HTTP, sa réponse et son identifiant |
| Débogage | `--timing` | Durée de chaque appel HTTP |
| Sortie | `-f table\|json\|yaml\|value\|csv\|shell` | Format |
| Sortie | `-c COLONNE` (répétable) | Ne garder que certaines colonnes |
| Sortie | `--sort-column COLONNE [--sort-descending]` | Trier |
| Sortie | `--fit-width`, `--max-width N` | Adapter la largeur des tableaux |

Les greffons installés (Nova, Neutron, Cinder, Glance, Placement, Heat, Octavia…) :

```bash
openstack module list
```

---

## 2. Récupérer des informations

### 2.1 La plateforme (compte `admin`) ✅

```bash
openstack versions show -c "Service Type" -c Version -c Status -c "Max Microversion"
openstack catalog list
openstack compute service list
openstack network agent list
openstack volume service list
openstack hypervisor list --long
openstack hypervisor show sunbeam
```

```text
| nova-scheduler | nova-0  | internal | enabled | up    |
| nova-conductor | nova-0  | internal | enabled | up    |
| nova-compute   | sunbeam | nova     | enabled | up    |

| OVN Controller Gateway agent | sunbeam | :-) | UP | ovn-controller             |
| OVN Metadata agent           | sunbeam | :-) | UP | neutron-ovn-metadata-agent |

| cinder-scheduler | cinder-0                   | nova | enabled | up   | 2026-10-16T15:13:33 |
| cinder-volume    | sunbeam@cinder-volume-ceph | nova | enabled | down | 2026-10-16T12:49:19 |
```

Ces trois listes de **services** sont le premier contrôle à faire : un service `down` (ici `cinder-volume`, voir démo 6) explique à lui seul des familles entières de pannes. La colonne `Updated At` date le dernier signal de vie.

⚠ `hypervisor show` n’affiche plus les champs `vcpus`, `memory_mb`, `running_vms` avec les microversions récentes de Nova : ces chiffres sont désormais dans **Placement**.

### 2.2 Les ressources réellement disponibles : Placement ✅

```bash
openstack resource provider list
openstack resource provider inventory list "$(openstack resource provider list -f value -c uuid --name sunbeam)"
```

```text
| resource_class | allocation_ratio | min_unit | max_unit | reserved | total | used |
| VCPU           |              4.0 |        1 |       12 |        0 |    12 |    0 |
| MEMORY_MB      |              1.0 |        1 |    23141 |      512 | 23141 |    0 |
| DISK_GB        |              1.0 |        1 |      160 |        0 |   160 |    0 |
```

Lecture : 12 vCPU physiques, **surallocation × 4** (48 vCPU allouables) ; 23 141 Mo de RAM dont 512 réservés à l’hôte, sans surallocation ; 160 Go de disque local.

**Où une VM peut-elle aller ?**

```bash
openstack allocation candidate list --resource VCPU=1 --resource MEMORY_MB=512 -c "resource provider" -c "inventory used/capacity"
```

```text
| cc165330-eaa5-44bc-968c-e364028e8b1a | DISK_GB=4/160,MEMORY_MB=512/22629,VCPU=1/48 |
```

Une ligne par hôte capable d’accueillir la demande. Aucune ligne : aucune VM de cette taille ne pourra démarrer (démo 5).

### 2.3 Un projet : quotas, consommation, ressources ✅

```bash
source ~/demo-openrc
openstack quota show --usage "$(openstack token issue -f value -c project_id)" -f value | grep -E "^(instances|cores|ram|networks|ports|volumes|gigabytes|floating-ips) "
openstack limits show --absolute
openstack server list --long
openstack image list -f value -c ID -c Name
openstack image show ubuntu -c name -c size -c disk_format -c min_ram -c visibility -c properties
openstack flavor list --sort-column RAM --sort-descending -c Name -c RAM -c VCPUs -c Disk
```

```text
cores -1 0 0
instances -1 0 0
ram -1 0 0
volumes 10 0 0
gigabytes 1000 0 0
```

Colonnes : nom, limite, utilisé, réservé. `-1` = illimité : sur le lab, le projet `demo` n’a **aucune limite** de calcul, seuls les volumes sont bornés.

⚠ Un utilisateur non administrateur ne peut pas désigner son projet **par son nom** dans `quota show` (`403 … identity:list_projects`) : utiliser l’identifiant, obtenu par `openstack token issue -f value -c project_id`.

⚠ `quota show --usage` affiche un tableau `Resource | Limit | In Use | Reserved` : l’ancienne sélection `-c instances -c cores` échoue (`No recognized column names`).

**Consommation cumulée par projet (admin) :**

```bash
source ~/admin-openrc
openstack usage list
```

```text
| demo | None | 1702.54 | 1.71 | 16.54 |      (RAM Mo-heures, heures CPU, Go-heures de disque)
```

### 2.4 Exporter un inventaire ✅

```bash
openstack server list --all-projects --long -f csv -c ID -c Name -c Status -c Host -c Networks > ~/inventaire-vm.csv
openstack image list -f json > ~/inventaire-images.json
openstack flavor list -f value -c Name -c RAM -c VCPUs
```

`-f value` sans en-tête se prête aux boucles `bash` ; `-f json` aux scripts Python ; `-f csv` aux tableurs.

---

## 3. Déboguer une commande

### 3.1 Voir les requêtes HTTP : `--debug` ✅

```bash
openstack --debug network list 2>&1 | grep -E "^REQ: curl|^RESP: \[|X-Openstack-Request-Id" | cut -c1-170
```

```text
REQ: curl -g -i -X GET http://cloud.lab.test/openstack-keystone/v3 -H "Accept: application/json" -H "User-Agent: openstacksdk/4.4.0 keystoneauth1/5.10.0 …
RESP: [200] Content-Length: 268 Content-Type: application/json … X-Openstack-Request-Id: req-…
REQ: curl -g -i -X GET http://cloud.lab.test:80/openstack-neutron/v2.0/networks -H "Accept: application/json" …
RESP: [200] Content-Length: 1222 … X-Openstack-Request-Id: req-c3b2375a-39f3-4e18-98e4-dc83669d2ca6
```

Ce que montre `--debug` :

- l’**ordre des appels** : découverte de Keystone, obtention du jeton, puis appel au service lu dans le catalogue ;
- chaque requête sous forme de commande **`curl` rejouable** ;
- le **code** et l’**identifiant de requête** (`X-Openstack-Request-Id`) de chaque réponse, à retrouver dans les journaux des services ;
- le corps des réponses (`RESP BODY`).

⚠ `--debug` affiche le **jeton** (`X-Auth-Token`) : ne pas copier sa sortie brute dans un ticket.

### 3.2 Mesurer : `--timing` ✅

```bash
openstack --timing network list -f value -c Name
```

```text
"URL","Seconds"
"GET http://cloud.lab.test/openstack-keystone/v3",0.023944
"POST http://cloud.lab.test/openstack-keystone/v3/auth/tokens",0.806247
"GET http://cloud.lab.test:80/openstack-neutron/v2.0/networks",0.773378
"Total",1.603569
```

À retenir : la moitié du temps passe dans l’**obtention du jeton**. Un script qui lance cent commandes paie cent fois ce coût : préférer un jeton réutilisé (`openstack token issue`, puis `--os-auth-type token --os-token …`) ou un SDK.

### 3.3 Les microversions ✅

```bash
openstack --os-compute-api-version 2.200 server list
```

```text
A default microversion for service compute of 2.200 was requested, but the cloud only supports a minimum of 2.1 and a maximum of 2.96.
```

Chaque service expose une plage de microversions (`openstack versions show`, colonne `Max Microversion`). Une option ou un champ absent s’explique souvent par une microversion trop ancienne ou trop récente. Exemple réel : `volume message list` exige `--os-volume-api-version 3.3` (démo 6).

### 3.4 Lire un code d’erreur ✅

| Code | Sens | Exemple observé sur le lab |
|---|---|---|
| `400` | Requête invalide | `console url show` : `Unavailable console type novnc` |
| `401` | Identité refusée | mauvais mot de passe : `The request you have made requires authentication. (HTTP 401) (Request-ID: req-…)` |
| `403` | Identité connue, action interdite par la politique | `demo` lance `user list` : `You are not authorized to perform the requested action: identity:list_users` |
| `404` / « No … found » | Objet absent **dans ta portée** | `admin` cherche par nom la VM du projet `demo` : `No Server found for lab-diag` |
| `409` | Conflit, souvent un quota | (voir `labs-iam/03-…`) |
| `500` | Erreur côté service | `fault` d’une VM en `ERROR` (démo 5) |
| `503` | Service indisponible | Lab qui démarre (`guide apprenant`, étape 8) |

---

## 4. Diagnostiquer une instance

| Question | Commande | Compte |
|---|---|---|
| État, adresses, flavor, image | `openstack server show VM` | propriétaire |
| Que s’est-il passé, et quand ? | `openstack server event list VM` puis `server event show VM REQ` | propriétaire |
| Que dit la VM au démarrage ? | `openstack console log show VM --lines 30` | propriétaire |
| Ouvrir sa console graphique | `openstack console url show --spice VM` | propriétaire |
| Sur quel hôte, sous quel nom libvirt ? | `openstack server show ID -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name` | admin |
| Son port réseau est-il branché ? | `openstack port list --device-id ID` puis `port show PORT -c status -c binding_host_id -c binding_vif_type` | admin |
| Consommation CPU, disque, réseau | `openstack server show ID --diagnostics` | admin |
| Pourquoi est-elle en `ERROR` ? | `openstack server show ID -c fault -f yaml` | admin |

⚠ **L’administrateur cherche d’abord dans son propre projet.** Pour une VM d’un autre projet, utiliser son **identifiant**, ou `openstack server list --all-projects --name VM`.

⚠ Le lab propose une console **SPICE**, pas noVNC : `console url show` sans `--spice` échoue en `400`.

---

## 5. Pièges observés sur le lab

| Piège | Symptôme | Réflexe |
|---|---|---|
| Objet d’un autre projet cherché par nom | `No Server found for lab-diag` (admin) | Identifiant, ou `--all-projects` |
| Projet désigné par son nom sans droits sur Keystone | `403 … identity:list_projects` | `-c project_id` de `token issue` |
| Nouveau format de `quota show --usage` | `No recognized column names` | `-f value \| grep` |
| Console par défaut | `400 Unavailable console type novnc` | `--spice` |
| Commande d’une API récente | `--os-volume-api-version 3.3 or greater is required` | Ajouter la microversion |
| Champs disparus de `hypervisor show` | `vcpus`, `memory_mb` absents | Placement : `resource provider inventory list` |
| `fault` vide pour le propriétaire | Seul `status: ERROR` visible | `server show ID -c fault` avec le compte admin, ou `server event show` |
| `--debug` | Affiche le jeton | Filtrer avant de partager |

---

## 6. Démos sur le projet

### Démo 1 : l’état de la plateforme en 5 commandes (5 min) ✅

```bash
source ~/admin-openrc
openstack compute service list -c Binary -c Host -c State
openstack network agent list -c "Agent Type" -c Alive -c State
openstack volume service list -c Binary -c Host -c State -c "Updated At"
openstack resource provider inventory list "$(openstack resource provider list -f value -c uuid --name sunbeam)"
openstack usage list
```

À commenter : trois familles de services (calcul, réseau, stockage) ; lors du test, tout était `up` sauf `cinder-volume`, `down` depuis 12:49 (la démo 6 en cherche la cause) ; la capacité réelle vient de Placement (12 vCPU × 4, 23 Go).

### Démo 2 : ce que fait vraiment une commande (10 min) ✅

```bash
source ~/demo-openrc
openstack --debug network list 2>&1 | grep -E "^REQ: curl|^RESP: \[|X-Openstack-Request-Id" | cut -c1-170
openstack --timing network list -f value -c Name
```

Puis retrouver la requête dans le journal de Neutron, avec l’identifiant affiché :

```bash
REQ=$(openstack --debug network list 2>&1 | grep -o "X-Openstack-Request-Id: req-[0-9a-f-]*" | tail -1 | cut -d' ' -f2)
echo "$REQ"
sudo k8s kubectl -n openstack logs neutron-0 -c neutron-server --since=2m | grep "$REQ" | grep "neutron.wsgi" | sed -E 's/.*\] //' | cut -c1-160
```

```text
10.1.0.196,10.1.0.141 "GET /openstack-neutron/v2.0/networks HTTP/1.1" status: 200  len: 1398 time: 1.8276970
```

L’identifiant figure aussi dans des lignes d’audit (`oslo.messaging.notification.audit`) : filtrer sur `neutron.wsgi` garde la ligne de la requête HTTP, avec l’adresse du client, le code et la durée côté serveur.

À commenter : le client ne connaît que Keystone ; il découvre l’adresse de Neutron dans le catalogue ; l’identifiant de requête relie le client aux journaux du service. Le traceur (`2-trace/`) montre la même chose en direct.

### Démo 3 : lire les refus (5 min) ✅

```bash
source ~/demo-openrc
OS_PASSWORD=faux openstack token issue
openstack user list
openstack server show inexistant
openstack --os-compute-api-version 2.200 server list
openstack --debug user list 2>&1 | grep -E "RESP: \[403\]" | cut -c1-90
```

À commenter : `401` (qui es-tu ?) contre `403` (tu n’as pas le droit) ; le nom de la règle refusée (`identity:list_users`) est dans le message ; la plage de microversions est donnée par le serveur.

### Démo 4 : tout savoir d’une VM (15 min) ✅

```bash
source ~/demo-openrc
openstack server create lab-diag --image ubuntu --flavor m1.tiny --network demo-network --wait -c status -c addresses
openstack server event list lab-diag
openstack server event show lab-diag "$(openstack server event list lab-diag -f value -c 'Request ID' | tail -1)"
sleep 40
openstack console log show lab-diag --lines 6
openstack console url show --spice lab-diag
SID=$(openstack server show lab-diag -f value -c id)
```

```text
| create | 2026-10-16T15:17:42 |
events: event='compute__do_build_and_run_instance', result='Success', start_time='…15:17:50', finish_time='…15:18:00'
url: http://cloud.lab.test/openstack-nova-spiceproxy/spice_auto.html?path=…&token=…
```

Vue de l’administrateur, par l’identifiant :

```bash
source ~/admin-openrc
openstack server show "$SID" -c name -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name -c project_id
openstack port show "$(openstack port list --device-id "$SID" -f value -c ID)" -c status -c binding_host_id -c binding_vif_type -c device_owner
openstack server show "$SID" --diagnostics -c state -c uptime -c driver -c hypervisor -c nic_details -c disk_details
```

```text
| OS-EXT-SRV-ATTR:host          | sunbeam           |
| OS-EXT-SRV-ATTR:instance_name | instance-00000006 |
| binding_host_id  | sunbeam |
| binding_vif_type | ovs     |
| status           | ACTIVE  |
| driver | libvirt | hypervisor | kvm | state | running |
nic_details: rx_packets 180, tx_packets 247, rx_errors 0, rx_drop 0
```

À commenter : `server event show` horodate chaque étape (ici 10 secondes de construction sur l’hôte) ; `instance-00000006` est le nom à chercher dans libvirt (`sudo openstack-hypervisor.virsh list`) ; un port `ACTIVE`, lié à l’hôte en `ovs`, prouve que le réseau de la VM est branché ; `--diagnostics` donne les compteurs vus par l’hyperviseur, sans entrer dans la VM.

```bash
source ~/demo-openrc
openstack server delete lab-diag --wait
```

### Démo 5 : pourquoi une VM passe en `ERROR` (15 min) ✅

On demande une VM plus grosse que l’hôte.

```bash
source ~/admin-openrc
openstack flavor create lab-geant --vcpus 2 --ram 65536 --disk 10 --public -c name -c ram
source ~/demo-openrc
openstack server create lab-trop-gros --image ubuntu --flavor lab-geant --network demo-network --wait
```

```text
Error creating server: lab-trop-gros
```

**1. Ce que voit le propriétaire :**

```bash
openstack server show lab-trop-gros -c status -c fault
openstack server event show lab-trop-gros "$(openstack server event list lab-trop-gros -f value -c 'Request ID' | tail -1)"
```

```text
| status | ERROR |
events: event='conductor_schedule_and_build_instances', result='Error'
```

Le champ `fault` est **vide** pour le propriétaire ; l’événement dit seulement **quelle étape** a échoué : la planification, avant tout hôte.

**2. Ce que voit l’administrateur :**

```bash
source ~/admin-openrc
EID=$(openstack server list --all-projects --name lab-trop-gros -f value -c ID)
openstack server show "$EID" -c fault -f yaml | head -8
```

```text
fault:
  code: 500
  details: "Traceback … in schedule_and_build_instances … in _schedule_instances … select_destinations …"
```

**3. La cause, côté Placement :**

```bash
openstack allocation candidate list --resource MEMORY_MB=65536 --resource VCPU=2
openstack allocation candidate list --resource MEMORY_MB=512 --resource VCPU=1 -c "resource provider" -c "inventory used/capacity"
```

✅ **Attendu :** aucune ligne pour 64 Go ; une ligne pour 512 Mo. Confirmation dans le journal du Scheduler, avec l’identifiant de la requête :

```bash
REQ=$(openstack server event list "$EID" -f value -c "Request ID" | tail -1)
sudo k8s kubectl -n openstack logs nova-0 -c nova-scheduler --since=15m | grep "$REQ" | sed -E 's/.*\] //' | cut -c1-160
```

```text
Got no allocation candidates from the Placement API. This could be due to insufficient resources …
```

À commenter : la méthode **événement → faute → capacité → journal** ; une VM en `ERROR` à la planification n’a jamais touché l’hyperviseur.

```bash
source ~/demo-openrc
openstack server delete lab-trop-gros --wait
source ~/admin-openrc
openstack flavor delete lab-geant
```

### Démo 6 : une panne réelle du lab, le volume qui ne se crée pas (15 min) ✅

Observée telle quelle sur le lab après la réimportation de la VM Sunbeam.

```bash
source ~/demo-openrc
openstack volume create lab-vol --size 1 -c status
sleep 25
openstack volume show lab-vol -c status
openstack --os-volume-api-version 3.3 volume message list -c "User Message" -c "Created At"
```

```text
| status | creating |
| status | error    |
| schedule allocate volume:Could not find any available weighted backend. | 2026-10-16T15:26:17 |
```

**1. Le message utilisateur** (`volume message list`, microversion 3.3) dit ce qu’aucun autre champ ne dit : le planificateur de Cinder n’a **trouvé aucun stockage**.

**2. Les services de stockage :**

```bash
source ~/admin-openrc
openstack volume service list
openstack volume backend pool list
```

```text
| cinder-scheduler | cinder-0                   | up   | 2026-10-16T15:26:53 |
| cinder-volume    | sunbeam@cinder-volume-ceph | down | 2026-10-16T12:49:19 |
```

`cinder-volume` est `down` depuis 12:49 (redémarrage du lab), et aucun pool n’est annoncé.

**3. On passe de l’autre côté, dans Juju** (voir `juju-cli/`, démo 4) :

```bash
juju status -m admin/openstack-machines cinder-volume microceph --format short | grep -E "cinder-volume/|microceph/"
```

```text
- cinder-volume/0: 10.0.2.15 (agent:idle, workload:waiting)     ← Not all relations are ready
- microceph/0:     10.0.2.15 (agent:idle, workload:blocked)     ← Error in charm: timed out
```

Diagnostic : au démarrage, le charm MicroCeph a dépassé son délai en interrogeant `snapd`, encore surchargé ; il est resté `blocked` et ne fournit plus à cinder-volume les informations de Ceph ; cinder-volume ne démarre donc pas, et Cinder n’a plus de stockage. Ceph lui-même fonctionne (`sudo ceph -s` : `1 osds: 1 up`).

À commenter : **le symptôme est dans OpenStack, la cause dans l’exploitation** ; le client `openstack` mène jusqu’au service en panne, Juju jusqu’à sa cause.

```bash
source ~/demo-openrc
openstack volume delete lab-vol
```

**4. Résoudre** ✅ (fait sur le lab le 17 octobre)

Les deux charms ont mémorisé l’échec du démarrage : leur hook périodique `update-status` reprend l’état `blocked` ou `waiting` sans refaire l’étape « Bootstrapping ». **Redémarrer l’agent Juju de la machine (`sudo systemctl restart jujud-machine-0`) n’a pas suffi** (essayé : toujours `blocked` 14 minutes après). Ce qui a fonctionné : relancer le hook `config-changed` de chaque unité, MicroCeph d’abord, puis cinder-volume.

```bash
juju exec -m admin/openstack-machines --unit microceph/0 -- "JUJU_DISPATCH_PATH=hooks/config-changed ./dispatch"
juju status -m admin/openstack-machines --format short | grep -E "microceph/|cinder-volume/"
juju exec -m admin/openstack-machines --unit cinder-volume/0 -- "JUJU_DISPATCH_PATH=hooks/config-changed ./dispatch"
juju status -m admin/openstack-machines --format short | grep -E "microceph/|cinder-volume/"
snap services cinder-volume
```

```text
microceph/0 … workload:active          (après un dernier « timed out » de 16 s, le hook aboutit)
cinder-volume/0 … workload:active      (« Completed guarded section fully: 'Bootstrapping' »)
cinder-volume.cinder-volume  enabled  active
```

Vérification côté OpenStack :

```bash
source ~/admin-openrc
openstack volume service list -c Binary -c State
openstack volume backend pool list
source ~/demo-openrc
openstack volume create lab-vol-test --size 1 -c status
openstack volume show lab-vol-test -c status
sudo rbd ls -p cinder-volume-ceph
openstack volume delete lab-vol-test
```

```text
| cinder-volume | up |
| cinder-volume@cinder-volume-ceph#cinder-volume-ceph |
| status | available |
volume-50b9d24a-aafe-416a-8675-968ac73027df
```

À commenter : le volume OpenStack est une **image RBD** dans le pool Ceph `cinder-volume-ceph`. ⚠ Lancer un hook à la main est une opération d’exploitation, à réserver au cas où l’état d’un charm est figé alors que le service sous-jacent est sain (ici, `ceph -s` était correct depuis le début) ; elle n’est pas destinée aux apprenants.

---

## 7. Récapitulatif des tests

| Commande | Exécutée | Remarque |
|---|---|---|
| `module list`, `versions show`, `configuration show`, `command list` | ✅ | |
| `catalog list`, `compute service list`, `network agent list`, `volume service list` | ✅ | `cinder-volume` `down` sur le lab |
| `hypervisor list --long`, `hypervisor show` | ✅ | Champs de capacité absents (microversion) |
| `resource provider list`, `inventory list`, `allocation candidate list` | ✅ | |
| `quota show --usage`, `limits show --absolute`, `usage list` | ✅ | |
| `image list`/`show`, `flavor list --sort-column` | ✅ | |
| `-f table/json/yaml/csv/value`, `-c` | ✅ | |
| `--debug`, `--timing`, `--os-compute-api-version`, `--os-volume-api-version` | ✅ | |
| Erreurs `400`, `401`, `403`, « No … found » | ✅ | |
| `server create/show/delete`, `server event list/show`, `console log show`, `console url show --spice` | ✅ | |
| `port list --device-id`, `port show`, `server show --diagnostics`, `server show -c fault` | ✅ | |
| `flavor create/delete`, `volume create/show/delete`, `volume message list`, `volume backend pool list` | ✅ | |

État du lab à la fin des tests : aucune VM, aucun volume, aucune IP flottante ni flavor de test. La panne de stockage de la démo 6 a été **résolue** (MicroCeph et cinder-volume `active`, volume de test créé puis supprimé). Pour rejouer la démo 6 en formation, il faut une machine où le problème est présent, par exemple juste après la réimportation d’une OVA sur un poste lent.

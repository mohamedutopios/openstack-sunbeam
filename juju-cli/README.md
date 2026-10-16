# La CLI Juju sur un cloud Sunbeam : syntaxe, commandes utiles et démos

Ce guide montre comment **exploiter et diagnostiquer** un cloud Sunbeam avec `juju`. Il complète la fiche `demo-juju/Juju-Concepts-et-Commandes-Sunbeam.md` (concepts : contrôleur, modèle, application, unité, charm, relation, états), sans la répéter.

Toutes les commandes marquées ✅ ont été **exécutées sur le lab** (Juju 3.6.29, contrôleur `sunbeam-controller`, déploiement `star-heron`), avec leur sortie réelle, parfois abrégée. Elles se lancent **dans Lab-Sunbeam**, avec le compte `vagrant` :

```powershell
ssh vagrant@192.168.56.10
```

Avant toute session Juju, renouveler la connexion au contrôleur (sinon : `Failed to login to Juju Controller`) :

```bash
sunbeam utils juju-login
```

---

## 1. À quoi sert Juju sur Sunbeam

Sunbeam décide **quoi** déployer (via ses plans Terraform) ; Juju **réalise et surveille** : il déploie les charms, les relie, applique leur configuration et remonte leur état. Pour l’exploitant, Juju est la **vue d’administration** du cloud :

| Question d’exploitation | Commande Juju |
|---|---|
| Tous les services sont-ils en bonne santé ? | `juju status` |
| Depuis quand ce service est-il en panne, et qu’a-t-il fait ? | `juju show-status-log` |
| Pourquoi un hook a-t-il échoué ? | `juju debug-log` |
| Quels réglages expose le charm, quelle valeur a été modifiée ? | `juju config` |
| Quelle opération d’administration le charm propose-t-il ? | `juju actions`, `juju run` |
| Que se passe-t-il dans le conteneur du service ? | `juju ssh`, `juju exec` |
| Qui consomme les services d’un autre modèle ? | `juju offers`, section `SAAS` de `juju status` |

**Règle d’or :** Juju sert à **observer et dépanner**. Pour changer l’architecture (ajouter un service, un nœud, changer de version), passer par `sunbeam`. Une modification faite avec `juju` peut être **écrasée** par le prochain `sunbeam enable`, `sunbeam cluster refresh` ou `configure`, qui réappliquent les plans Terraform.

---

## 2. Syntaxe générale

```text
juju <commande> [-m [propriétaire/]modèle] [cible] [options]
```

| Élément | Forme | Exemple |
|---|---|---|
| Modèle | `-m nom` ou `-m propriétaire/nom` | `-m openstack`, `-m admin/openstack-machines` |
| Application | `nom` | `keystone` |
| Unité | `application/numéro` | `keystone/0` |
| Machine (modèle `manual`) | numéro | `0` |
| Format de sortie | `--format tabular\|short\|oneline\|summary\|yaml\|json` | `juju status --format short` |
| Aide | `juju help <commande>` ou `juju <commande> -h` | `juju help debug-log` |

### Les modèles du lab ✅

```bash
juju models
```

```text
Controller: sunbeam-controller

Model                     Cloud/Region              Type        Status     Machines  Cores  Units  Access
admin/controller          star-heron-k8s/localhost  kubernetes  available         0      -  1      admin
openstack                 star-heron-k8s/localhost  kubernetes  available         0      -  25     admin
admin/openstack-machines  star-heron/default        manual      available         1      8  7      admin
```

| Modèle | Type | Contenu |
|---|---|---|
| `openstack` | `kubernetes` | Les 25 applications du plan de contrôle (une par service, chacune réalisée par un pod) |
| `admin/openstack-machines` | `manual` | La machine `sunbeam` elle-même : k8s, microceph, openstack-hypervisor, cinder-volume… |
| `admin/controller` | `kubernetes` | Le contrôleur Juju |

⚠ **Piège :** le modèle des machines appartient à l’utilisateur Juju **`admin`**, alors qu’on est connecté en tant que `sunbeam` (`juju whoami`). `juju status -m openstack-machines` répond `model … not found` : il faut écrire **`-m admin/openstack-machines`**. (La fiche `demo-juju/` contient encore la forme courte, à corriger.)

### Les formats de `juju status` ✅

| Format | Usage |
|---|---|
| `tabular` (par défaut) | Lecture humaine, avec le message de chaque application |
| `short` (ou `oneline`) | Une ligne par unité : idéal pour filtrer avec `grep` |
| `summary` | Comptes globaux (unités actives, applications) |
| `yaml`, `json` | Scripts et supervision |

---

## 3. Les commandes utiles en entreprise

### 3.1 Se repérer ✅

```bash
juju version
juju whoami
juju controllers
juju models
juju show-model openstack
```

```text
3.6.29-genericlinux-amd64

Controller:  sunbeam-controller
Model:       <no-current-model>
User:        sunbeam
```

`juju show-model openstack` donne le propriétaire, le type (`caas` = Kubernetes) et le **coffre de secrets** du modèle (`secret-backends: internal: num-secrets: 54` sur le lab).

`juju controllers` montre aussi un contrôleur `localhost-localhost` : il est sans rapport avec OpenStack (contrôleur LXD local). Le contrôleur du cloud est `sunbeam-controller*` (l’astérisque marque le courant).

### 3.2 Contrôler l’état ✅

**Ce qui ne va pas, en une commande :**

```bash
juju status -m openstack --format short | grep -v "workload:active"
juju status -m admin/openstack-machines --format short | grep -v "workload:active"
```

```text
- cinder-volume/0: 10.0.2.15 (agent:idle, workload:waiting)
- microceph/0: 10.0.2.15 (agent:idle, workload:blocked)
```

Sur le lab : rien dans `openstack` (25 unités actives), deux anomalies sur la machine, expliquées en démo 4.

**Une ou plusieurs applications :**

```bash
juju status -m openstack keystone
juju status -m openstack nova neutron
```

⚠ Les jokers ne fonctionnent pas : `juju status keystone*` n’affiche aucune application.

**Avec les relations :**

```bash
juju status -m openstack --relations keystone
```

**Synthèse chiffrée :**

```bash
juju status -m openstack --format summary
```

```text
                    # Units:  (25)
                     active:   25
             # Applications:  (25)
```

**Suivi en continu :** `watch -n 10 juju status -m openstack`. ⚠ `juju status --watch 3s` n’existe que dans un **vrai terminal** : lancé depuis un script, il échoue (`unable to run Viddy … term not set`).

### 3.3 Les liens entre modèles : offres et SAAS ✅

```bash
juju offers -m admin/openstack-machines
juju status -m openstack | sed -n "/^SAAS/,/^$/p"
```

```text
Offer               User     Relation ID  Status  Endpoint         Interface       Role
cinder-volume       sunbeam  8            joined  storage-backend  cinder-backend  provider
microceph           sunbeam  9            joined  ceph             ceph-client     provider

SAAS           Status   Store  URL
cinder-volume  waiting  local  admin/openstack-machines.cinder-volume
microceph      blocked  local  admin/openstack-machines.microceph
```

Le modèle de la machine **offre** MicroCeph et cinder-volume ; le modèle `openstack` les **consomme** (lignes `SAAS`). C’est pourquoi l’état `blocked` de `microceph` apparaît aussi dans `juju status -m openstack`.

### 3.4 Détailler une application ou une unité ✅

```bash
juju show-application -m openstack keystone
juju show-unit -m openstack keystone/0
```

`show-unit` liste les relations de l’unité et les données échangées : base `keystone` via `keystone-mysql-router-service…:6446`, RabbitMQ via `rabbitmq-endpoints…`. ⚠ Ces données contiennent des **références de secrets** : à ne pas copier dans un ticket sans relecture.

### 3.5 Lire et modifier la configuration ✅

```bash
juju config -m openstack keystone                 # tout : options Juju et options du charm
juju config -m openstack keystone log-level       # une seule valeur
juju config -m openstack keystone log-level=INFO  # modifier
juju config -m openstack keystone --reset log-level  # revenir à la valeur par défaut
```

Liste compacte des options du charm, avec leur origine :

```bash
juju config -m openstack keystone --format json | python3 -c "import sys,json; [print(k, '=', v.get('value'), '(', v.get('source'), ')') for k,v in sorted(json.load(sys.stdin)['settings'].items())]"
```

```text
catalog-cache-expiration = 60 ( default )
debug = False ( default )
dogpile-cache-expiration = 60 ( default )
enable-telemetry-notifications = False ( default )
identity-backend = sql ( default )
log-level = WARNING ( default )
region = RegionOne ( default )
saml-x509-keypair =  ( user )
```

La colonne `source` dit si la valeur a été **changée** (`user`) ou non (`default`) : c’est le premier contrôle à faire sur un service au comportement inattendu.

### 3.6 Journaux et historique ✅

**Historique des états d’une unité :**

```bash
juju show-status-log -m openstack keystone-mysql-router/0
```

**Journaux des agents et des charms :**

```bash
juju debug-log -m openstack --replay --no-tail --level ERROR --include keystone-mysql-router/0
juju debug-log -m openstack --lines 20 --include keystone/0      # suivi en direct, Ctrl+C pour arrêter
```

| Option | Effet |
|---|---|
| `--include <unité>` | Ne garder qu’une unité (`keystone/0` ou `unit-keystone-0`) |
| `--level ERROR` | Ne garder que les erreurs (`WARNING`, `INFO`, `DEBUG`…) |
| `--replay` | Relire depuis le début de l’historique conservé |
| `--no-tail` | S’arrêter à la fin, sans attendre la suite |
| `--lines N` | Afficher les N dernières lignes puis suivre |

⚠ `--no-tail` et `--lines` sont **incompatibles** (`ERROR setting --no-tail and --lines not valid`). Pour les N dernières lignes d’un historique : `--replay --no-tail … | tail -N`.

Les journaux de Juju concernent le **charm** (hooks, relations, configuration). Les journaux du **service** lui-même (requêtes API) sont dans Kubernetes : `sudo k8s kubectl -n openstack logs keystone-0 -c keystone`.

### 3.7 Actions ✅

```bash
juju actions -m openstack mysql
juju actions -m openstack keystone
juju run -m openstack mysql/0 get-cluster-status
```

```text
status:
  clusterrole: primary
  defaultreplicaset:
    primary: mysql-0.mysql-endpoints.openstack.svc.cluster.local.:3306
    status: ok_no_tolerance
    statustext: cluster is not tolerant to any failures.
    topology:
      mysql-0: {memberrole: primary, mode: r/w, status: online}
```

Une action est une **opération d’administration fournie par le charm** : sauvegarde (`create-backup`), bascule (`promote-to-primary`), contrôle avant mise à jour (`pre-upgrade-check`)… Ici, `ok_no_tolerance` est attendu : une seule instance MySQL, aucune tolérance aux pannes.

⚠ Certaines actions **révèlent des secrets** : `keystone get-admin-account` (mot de passe administrateur), `rabbitmq get-operator-info`, `mysql get-password`. Les lancer seulement si nécessaire, jamais sur un écran projeté.

### 3.8 Entrer dans les unités ✅

```bash
juju ssh -m openstack --container keystone keystone/0 "hostname; grep -c . /etc/keystone/keystone.conf"
juju exec -m openstack --unit keystone/0 -- hostname
juju ssh -m admin/openstack-machines 0 "hostname; sudo microceph status | head -5"
```

```text
keystone-0
67
keystone-0
sunbeam
MicroCeph deployment summary:
- sunbeam (192.168.56.10)
  Services: mds, mgr, mon, rgw, osd
```

| Commande | Où elle s’exécute |
|---|---|
| `juju ssh --container <service> <unité>` | Dans le conteneur du **service** (ici `keystone`) |
| `juju ssh --container charm <unité>` | Dans le conteneur du **charm** |
| `juju exec --unit <unité> -- <commande>` | Dans le contexte du charm, sans session interactive |
| `juju ssh -m admin/openstack-machines 0` | Sur la **machine** du modèle `manual` |

⚠ Ce qu’on modifie à la main dans un conteneur **disparaît** au prochain redémarrage du pod ou à la prochaine reconfiguration du charm.

### 3.9 Secrets ✅

```bash
juju secrets -m openstack
juju show-secret -m openstack <ID>
```

```text
25nr8pif4f7j4j2es14g:
  revision: 1
  owner: mysql
  label: database.128.user.secret
  access:
  - target: application-nova-api-mysql-router
    role: view
```

Sans `--reveal`, `show-secret` montre seulement les **métadonnées** : propriétaire, rotation, qui y a accès. C’est ce qu’il faut pour un audit. `--reveal` affiche le contenu : à éviter en formation. On voit aussi la **rotation** : `daily` ou `monthly` pour certains secrets de Keystone (clés Fernet et de credentials).

### 3.10 Relancer une unité en erreur ⚠ (exécutée sur le lab lors d’incidents réels)

```bash
juju status -m openstack --format short | grep "agent:error"
juju resolved -m openstack keystone-mysql-router/0
```

À faire **après** avoir lu la cause dans `debug-log`. Voir la démo 3.

---

## 4. Pièges observés sur le lab

| Piège | Symptôme | Réflexe |
|---|---|---|
| Session expirée | `Failed to login to Juju Controller` | `sunbeam utils juju-login` |
| Modèle de la machine | `model sunbeam:sunbeam/openstack-machines not found` | `-m admin/openstack-machines` |
| Jokers dans les filtres | `juju status keystone*` n’affiche rien | Nommer les applications |
| `--lines` et `--no-tail` | `ERROR setting --no-tail and --lines not valid` | `--replay --no-tail \| tail -N` |
| `status --watch` dans un script | `unable to run Viddy … term not set` | `watch -n 10 juju status …` |
| Configuration modifiée à la main | Revient à la valeur d’avant après un `sunbeam` | Documenter, préférer `sunbeam` quand c’est possible |
| Reconfiguration d’un service | `blocked … healthcheck failed` quelques secondes | Normal ; faire ces changements en fenêtre de maintenance |
| Actions et `show-unit` | Affichent des secrets ou leurs références | Ne pas projeter, relire avant de partager |

---

## 5. Démos sur le projet

Chaque démo est testée sur le lab et laisse le cloud dans son état initial.

### Démo 1 : se repérer dans Juju (5 min) ✅

```bash
sunbeam utils juju-login
juju whoami
juju controllers
juju models
juju show-model openstack | grep -E "owner|type|num-secrets"
```

À commenter : on est l’utilisateur `sunbeam` du contrôleur `sunbeam-controller` ; trois modèles, dont un de type `manual` pour la machine physique ; le modèle `openstack` porte 54 secrets gérés par Juju (mots de passe des bases, clés).

### Démo 2 : un état de santé en 30 secondes (5 min) ✅

```bash
juju status -m openstack --format summary | head -6
juju status -m openstack --format short | grep -v "workload:active"
juju status -m admin/openstack-machines --format short | grep -v "workload:active"
juju offers -m admin/openstack-machines
```

À commenter : 25 unités actives côté OpenStack ; deux anomalies côté machine ; elles « remontent » dans le modèle `openstack` par les offres (`SAAS`). On en cherche la cause en démo 4.

### Démo 3 : reconstituer un incident réel (15 min) ✅

Contexte : après chaque redémarrage du lab, des unités `*-mysql-router` passaient en erreur et Keystone renvoyait `HTTP 500`. On reconstitue l’enquête.

```bash
juju show-status-log -m openstack keystone-mysql-router/0
```

```text
15 Oct 2026 18:27:49Z  juju-unit  error        hook failed: "update-status"
15 Oct 2026 18:31:23Z  workload   maintenance  MySQL Router starting
16 Oct 2026 12:39:34Z  juju-unit  error        hook failed: "start"
16 Oct 2026 12:40:34Z  juju-unit  error        hook failed: "start"
16 Oct 2026 12:41:58Z  juju-unit  error        hook failed: "start"
```

Quand, et quoi : le hook `start` a échoué trois fois au démarrage. Pourquoi :

```bash
juju debug-log -m openstack --replay --no-tail --level ERROR --include keystone-mysql-router/0 | tail -4
```

```text
unit-keystone-mysql-router-0: … cannot open api: cannot resolve "controller-service.controller-sunbeam-controller.svc.cluster.local":
  lookup … on 10.152.183.236:53: dial udp 10.152.183.236:53: connect: operation not permitted
unit-keystone-mysql-router-0: … cannot open api: unable to connect to API: dial tcp 192.168.56.200:17070: connect: operation not permitted
```

Diagnostic : au démarrage, l’agent de l’unité ne pouvait **ni résoudre** le nom du contrôleur (DNS de Kubernetes, `10.152.183.236`) **ni le joindre** (`192.168.56.200:17070`) : le réseau des pods n’était pas encore prêt. Le hook a échoué, et Juju ne le relance pas de lui-même. Correction, une fois le réseau rétabli :

```bash
juju resolved -m openstack keystone-mysql-router/0
```

À commenter : la méthode **quand → quoi → pourquoi → corriger** (`show-status-log`, `debug-log`, puis `resolved`), et la chaîne de conséquences : un routeur MySQL en erreur, donc Keystone sans base, donc `HTTP 500` sur toute authentification.

### Démo 4 : enquêter sur MicroCeph `blocked` (10 min) ✅

```bash
juju status -m admin/openstack-machines microceph
juju show-status-log -m admin/openstack-machines microceph/0 | tail -4
juju debug-log -m admin/openstack-machines --replay --no-tail --include microceph/0 | grep -B2 -A3 -m1 "Traceback"
juju ssh -m admin/openstack-machines 0 "sudo microceph status | head -5; sudo ceph -s | grep -E 'osd:|pgs:'"
```

```text
microceph/0*  blocked  idle  0  10.0.2.15  (workload) Error in charm (see logs): timed out
… ERROR unit.microceph/0.juju-log Exception raised in section 'Bootstrapping': timed out
… File ".../src/microceph.py", line 57, in is_ready
    osd: 1 osds: 1 up, 1 in
    pgs:     289 active+clean
```

Diagnostic : au démarrage, la vérification `is_ready` du charm a dépassé son délai (MicroCeph lent à répondre pendant que la machine démarrait) ; le charm est resté `blocked` depuis. Ceph, lui, est sain. À commenter : **l’état d’un charm n’est pas l’état du service** ; toujours confirmer avec un test du service lui-même (`ceph -s`, une API, une VM).

### Démo 5 : modifier une configuration, la voir appliquée, revenir en arrière (10 min) ✅

```bash
juju config -m openstack keystone log-level
sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- grep -E "^level" /etc/keystone/logging.conf | sed -n 2p
juju config -m openstack keystone log-level=INFO
watch -n 3 "juju status -m openstack keystone --format short"
```

Observé sur le lab, dans `juju show-status-log -m openstack keystone/0` :

```text
14:58:21Z  juju-unit  executing  running config-changed hook
14:58:42Z  workload   active
14:58:44Z  juju-unit  idle
```

Vérifier l’effet, puis revenir à la valeur par défaut :

```bash
sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- grep -E "^level" /etc/keystone/logging.conf | sed -n 2p
juju config -m openstack keystone --reset log-level
juju config -m openstack keystone log-level
```

✅ **Attendu :** `level=WARNING`, puis `level=INFO` environ 20 secondes après la commande, puis de nouveau `level=WARNING` après `--reset`.

À commenter :

1. la commande rend la main **tout de suite** ; le charm applique ensuite le changement (hook `config-changed`) ;
2. le réglage n’est pas dans `keystone.conf` mais dans `logging.conf` (référencé par `log_config_append`) : **le charm sait où écrire**, l’exploitant n’a pas à le savoir ;
3. lors d’un second essai, Keystone est passé `blocked (container:keystone) healthcheck failed` pendant 8 secondes, le temps de redémarrer : un changement de configuration **interrompt brièvement le service**, ici l’authentification de tout le cloud ;
4. `source` passe de `default` à `user` tant que la valeur est modifiée : c’est la trace d’un changement manuel.

### Démo 6 : utiliser une action (5 min) ✅

```bash
juju actions -m openstack mysql
juju run -m openstack mysql/0 get-cluster-status
```

```bash
juju operations -m openstack | tail -4
```

```text
67  completed  2026-10-16T14:22:21  …  get-admin-account run on unit-keystone-0
69  completed  2026-10-16T14:22:25  …  list-ca-certs run on unit-keystone-0
71  completed  2026-10-16T14:57:14  …  get-cluster-status run on unit-mysql-0
73  completed  2026-10-16T14:57:23  …  juju-exec run on unit-keystone-0
```

À commenter : l’action est le moyen **prévu par le charm** pour une opération d’administration, à préférer à une commande lancée à la main dans le conteneur ; elle laisse une **trace** dans `juju operations`. Les deux premières lignes n’ont été lancées par personne à la main : c’est **Sunbeam** qui les a déclenchées pendant un `sunbeam launch`, pour récupérer les accès administrateur. Sunbeam lui-même pilote le cloud par des actions Juju.

### Démo 7 : du charm au service (10 min) ✅

```bash
juju ssh -m openstack --container keystone keystone/0 "hostname; grep -c . /etc/keystone/keystone.conf"
juju exec -m openstack --unit keystone/0 -- hostname
juju ssh -m admin/openstack-machines 0 hostname
juju secrets -m openstack | head -5
juju show-secret -m openstack "$(juju secrets -m openstack --format json | python3 -c "import sys,json; print(next(k for k,v in json.load(sys.stdin).items() if v.get('owner')=='mysql'))")"
```

À commenter : une unité Kubernetes = un pod à plusieurs conteneurs (`charm`, `keystone`) ; une unité du modèle `manual` = la machine ; les mots de passe des bases sont des **secrets Juju** partagés par relation, consultables sans être révélés.

---

## 6. Récapitulatif des tests

| Commande | Exécutée | Remarque |
|---|---|---|
| `version`, `whoami`, `controllers`, `models`, `show-model`, `show-controller` | ✅ | `show-controller` affiche le certificat de la CA du contrôleur |
| `status` (`tabular`, `short`, `oneline`, `summary`, `yaml`, `json`, filtres, `--relations`) | ✅ | Pas de jokers ; `--watch` seulement dans un terminal |
| `offers`, section `SAAS` | ✅ | |
| `show-application`, `show-unit` | ✅ | `show-unit` montre des références de secrets |
| `config` (lecture, une clé, modification, `--reset`) | ✅ | Effet vérifié dans `logging.conf` ; brève indisponibilité de Keystone |
| `show-status-log`, `debug-log` (`--replay`, `--no-tail`, `--level`, `--include`, `--lines`) | ✅ | `--lines` incompatible avec `--no-tail` |
| `actions`, `run mysql/0 get-cluster-status` | ✅ | |
| `ssh --container`, `exec --unit`, `ssh` sur la machine 0 | ✅ | |
| `secrets`, `show-secret` (sans `--reveal`) | ✅ | |
| `resolved` | ✅ | Exécutée lors des incidents réels du lab, pas pendant ce test |
| `deploy`, `remove-application`, `integrate`, `refresh`, `scale-application` | ⛔ volontairement non exécutées | Réservées à `sunbeam` sur ce cloud |

État du lab à la fin des tests : identique à l’état initial (`log-level` de Keystone revenu à sa valeur par défaut, aucune ressource créée).

# La CLI Sunbeam : fonctionnement, commandes utiles et démos

Ce guide présente la commande `sunbeam`, l’outil qui installe et **exploite** un cloud OpenStack Sunbeam. Toutes les commandes marquées ✅ ont été **exécutées sur le lab** (snap `openstack` 2024.1, révision 1101, déploiement `star-heron`, nœud unique `sunbeam`), avec leur sortie réelle. Celles marquées ⚠ **non exécutée** modifient la topologie ou nécessitent une seconde machine : leur syntaxe vient de l’aide de la CLI (`-h`).

Toutes les commandes se lancent **dans Lab-Sunbeam**, avec le compte `vagrant` :

```powershell
ssh vagrant@192.168.56.10
```

---

## 1. Comment fonctionne Sunbeam

`sunbeam` ne parle presque jamais directement à OpenStack. Il pilote une chaîne d’outils, chacun avec son rôle :

```text
 sunbeam (CLI, snap « openstack »)
   │
   ├── clusterd ────────── base du cluster : nœuds, rôles, manifestes, configuration des fonctionnalités
   │
   ├── Terraform (plans) ─ décrit l'état voulu : quelles applications, quelle configuration
   │       │
   │       ▼
   └── Juju (contrôleur sunbeam-controller, lui-même dans Kubernetes)
           │  déploie et relie les charms
           ├── modèle « openstack »                 ─► pods Kubernetes : keystone, nova, neutron, glance, horizon, mysql, rabbitmq, ovn…
           └── modèle « admin/openstack-machines »  ─► la machine : k8s, microceph, openstack-hypervisor, cinder-volume
```

| Composant | Rôle | Où le voir |
|---|---|---|
| **snap `openstack`** | Fournit la commande `sunbeam` et `clusterd` | `snap list openstack` |
| **clusterd** | Base de données du cluster : nœuds, rôles (control, compute, storage), manifestes appliqués, état des fonctionnalités | `sunbeam cluster list`, `sunbeam manifest list` |
| **Plans Terraform** | Un plan par couche (k8s, machine, hyperviseur, OpenStack, MicroCeph…) : Sunbeam calcule l’état voulu, Terraform l’applique via Juju | `sunbeam plans list` |
| **Juju** | Orchestrateur de charms : déploie, configure, relie les applications et suit leur état | `juju status -m openstack` |
| **Charms** | Le code d’exploitation de chaque service (keystone-k8s, nova-k8s, microceph…) | `juju status` (colonne Charm) |
| **Manifeste** | Fichier YAML qui fixe les réglages (réseaux, adresses, canaux des charms, fonctionnalités) | `sunbeam manifest show latest` |
| **Fonctionnalités (features)** | Services optionnels qu’on active ou désactive : Heat, Vault, TLS, LDAP, observabilité… | `sunbeam list-features` |

**Conséquence pratique :** pour changer l’architecture du cloud (ajouter un nœud, activer un service, changer de version), on passe par `sunbeam`, pas par `juju deploy` ni par une modification à la main. Pour **utiliser** le cloud (projets, VM, réseaux), on passe par `openstack`. Juju sert à **observer** et à dépanner.

| Outil | À utiliser pour |
|---|---|
| `sunbeam` | Installer, ajouter ou retirer des nœuds, activer des fonctionnalités, mettre à jour, récupérer les accès |
| `juju` | Voir l’état des applications, lire les journaux, relancer une unité en erreur |
| `openstack` | Tout ce qui concerne les utilisateurs du cloud : identités, projets, ressources |
| `sudo k8s kubectl` | Voir les pods, lire les journaux des services |

---

## 2. Syntaxe générale

```text
sunbeam [-v | -q] <groupe> [<sous-commande>] [options]
```

| Option | Effet | Exemple |
|---|---|---|
| `-h`, `--help` | Aide du niveau courant | `sunbeam cluster -h`, `sunbeam enable tls -h` |
| `-v` | Mode bavard : affiche les étapes et les erreurs détaillées | `sunbeam -v cluster list` |
| `-q` | Mode silencieux | `sunbeam -q dashboard-url` |
| `-f table\|yaml` | Format de sortie (selon la commande) | `sunbeam cluster list -f yaml` |
| `-m FICHIER` | Applique un manifeste | `sunbeam enable -m manifest.yaml …` |
| `--hints / --no-hints` | Affiche ou masque les questions d’aide interactives | `sunbeam cloud-config --no-hints` |

⚠ **Piège : `sunbeam` est un snap confiné.** Son `/tmp` est privé : `sunbeam manifest generate -f /tmp/m.yaml` répond « Generated manifest is at /tmp/m.yaml » mais le fichier est **invisible** pour toi. Écris toujours dans ton répertoire personnel (`~/…`).

### Carte des commandes (version 2024.1 du lab)

| Groupe | Sous-commandes | Usage |
|---|---|---|
| `cluster` | `list`, `bootstrap`, `add`, `join`, `remove`, `resize`, `refresh` | Nœuds du cluster et cycle de vie |
| `deployment` | `list`, `show`, `add`, `switch`, `export`, `import`, `update-clusterd-credentials` | Déploiements connus de ce client |
| `configure` | `deployment`, `dpdk`, `sriov` | Configuration initiale du cloud (projet de démonstration, réseau externe) |
| `enable` / `disable` | `caas`, `dns`, `images-sync`, `instance-recovery`, `ldap`, `loadbalancer`, `observability`, `orchestration`, `pro`, `resource-optimization`, `secrets`, `telemetry`, `tls`, `validation`, `vault` | Fonctionnalités optionnelles |
| `list-features`, `list-feature-gates` | — | État des fonctionnalités ; fonctionnalités encore expérimentales |
| `manifest` | `list`, `show`, `generate` | Manifestes appliqués |
| `openrc`, `cloud-config` | — | Accès administrateur et `clouds.yaml` |
| `dashboard-url`, `dashboard` | `url`, `theme set`, `theme clear` | Horizon |
| `identity` | `provider add/list/remove/update/purge/get-oidc-redirect-url`, `set-saml-x509` | Fédération d’identité (OpenID Connect, SAML) |
| `storage` | `list`, `add`, `show`, `options`, `remove` | Baies de stockage externes pour Cinder |
| `proxy` | `show`, `set`, `clear` | Proxy HTTP d’entreprise |
| `plans` | `list`, `unlock`, `shell` | Plans Terraform : verrous, dépannage |
| `utils` | `juju-login` | Renouveler la session Juju |
| `juju` | `register-controller`, `unregister-controller` | Contrôleur Juju externe |
| `launch` | — | Lancer une VM de test dans le projet de démonstration |
| `prepare-node-script` | — | Script de préparation d’une nouvelle machine |

⚠ `sunbeam inspect`, cité dans d’anciennes documentations, **n’existe pas** dans cette version.

---

## 3. Les commandes utiles en entreprise

### 3.1 Observer l’état du cloud

**Nœuds et rôles** ✅

```bash
sunbeam cluster list
```

```text
                          openstack-machines
┏━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┓
┃ Node    ┃ Machine ┃ Cluster ┃ Compute ┃ Control ┃ Network ┃ Storage ┃
┡━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━┩
│ sunbeam │ running │ ONLINE  │ active  │ active  │         │ blocked │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

Une colonne par rôle. Ici, `Storage: blocked` vient de l’unité Juju `microceph/0` (voir § 5.3). `-f yaml` donne la même information pour un script.

**Déploiements connus** ✅

```bash
sunbeam deployment list
```

```text
│ star-heron* │ local    │ local │
```

L’astérisque marque le déploiement courant. Le nom (`star-heron`) est généré au bootstrap : c’est lui qu’attend `sunbeam deployment show`.

```bash
sunbeam deployment show star-heron
```

⚠ **Sensible :** la sortie contient les certificats, une clé privée et des fragments de mot de passe du contrôleur Juju. Ne pas la projeter ni la copier dans un ticket.

**Fonctionnalités** ✅

```bash
sunbeam list-features
sunbeam list-features -f yaml
sunbeam list-feature-gates
```

`list-features` montre ce qui est activé ; `list-feature-gates` liste les fonctionnalités encore **expérimentales** (pilotes de baies de stockage, multirégion, `baremetal`…), verrouillées tant qu’on ne les a pas explicitement déverrouillées.

**Manifestes appliqués** ✅

```bash
sunbeam manifest list
sunbeam manifest show latest
sunbeam manifest generate -f ~/manifest-actuel.yaml
```

```text
core:
  config:
    bootstrap:
      management_cidr: 192.168.56.0/24
    database: single
    endpoints:
      ingress-internal: {hostname: internal.lab.test, ip: 192.168.56.202}
      ingress-public:   {hostname: cloud.lab.test,    ip: 192.168.56.201}
      ingress-rgw:      {hostname: rgw.lab.test,      ip: 192.168.56.203}
    k8s-addons:
      loadbalancer: 192.168.56.200-192.168.56.220
    region: RegionOne
```

`manifest generate` produit le manifeste **complet** de la configuration déployée (612 lignes sur le lab). ⚠ Il contient des mots de passe : le traiter comme un secret. Usage en entreprise : documenter l’état d’un cloud, le comparer avant et après une modification (`diff`), ou le rejouer pour reconstruire un cloud identique.

**Plans Terraform** ✅

```bash
sunbeam plans list
```

```text
│ k8s-plan             │        │
│ sunbeam-machine-plan │        │
│ hypervisor-plan      │        │
│ demo-setup           │        │
│ openstack-plan       │        │
│ microceph-plan       │        │
│ cinder-volume-plan   │        │
```

Colonne `Locked` vide : aucune opération en cours. Un plan verrouillé alors que rien ne tourne (opération interrompue) bloque toute commande suivante : `sunbeam plans unlock <plan>` le libère (⚠ non exécutée, à faire seulement après avoir vérifié qu’aucune opération ne tourne).

**Proxy et stockage** ✅

```bash
sunbeam proxy show
sunbeam storage list
```

Vides sur le lab : pas de proxy, et le stockage des volumes est assuré par MicroCeph (pas de baie externe).

### 3.2 Récupérer les accès

**Jeu de variables de l’administrateur** ✅

```bash
sunbeam openrc > ~/admin-openrc.nouveau
```

```text
export OS_USERNAME=admin
export OS_PASSWORD=********
export OS_AUTH_URL=http://cloud.lab.test/openstack-keystone/v3
export OS_USER_DOMAIN_NAME=admin_domain
export OS_PROJECT_DOMAIN_NAME=admin_domain
export OS_PROJECT_NAME=admin
export OS_AUTH_VERSION=3
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
```

**Fichier `clouds.yaml`** ✅ (plus pratique que les fichiers `openrc` : plusieurs comptes dans un seul fichier)

```bash
sunbeam cloud-config -u -f ~/clouds-lab.yaml --no-hints
sunbeam cloud-config -a -c sunbeam-admin -u -f ~/clouds-lab.yaml --no-hints
```

| Option | Rôle |
|---|---|
| `-u` | **Écrit** dans le fichier. Sans `-u`, la commande affiche seulement le YAML |
| `-a` | Compte administrateur (sans `-a` : l’utilisateur `demo` créé par `configure`) |
| `-c NOM` | Nom de l’entrée dans `clouds.yaml` (par défaut `sunbeam`) |
| `-f FICHIER` | Fichier visé (par défaut `~/.config/openstack/clouds.yaml`) |

Résultat : deux entrées, `sunbeam` (demo) et `sunbeam-admin`. Utilisation :

```bash
OS_CLIENT_CONFIG_FILE=~/clouds-lab.yaml openstack --os-cloud sunbeam server list
OS_CLIENT_CONFIG_FILE=~/clouds-lab.yaml openstack --os-cloud sunbeam-admin hypervisor list
```

⚠ Si des variables `OS_*` sont déjà chargées (`source ~/admin-openrc`), elles ont priorité sur `--os-cloud` : ouvrir une session neuve, ou les retirer (`env -u OS_AUTH_URL …`).

**Adresse d’Horizon** ✅

```bash
sunbeam dashboard-url
```

```text
http://cloud.lab.test:80/openstack-horizon
```

**Renouveler la session Juju** ✅

```bash
sunbeam utils juju-login
```

```text
Juju re-login complete.
```

À connaître absolument : quand une commande `sunbeam` ou `juju` répond **`Failed to login to Juju Controller`** (observé sur le lab après un redémarrage), c’est la session Juju qui a expiré, pas une panne.

### 3.3 Faire évoluer le cloud

**Ajouter un nœud** ⚠ non exécutée (lab à nœud unique)

```bash
sunbeam cluster add noeud2.lab.test --output ~/noeud2.token      # sur un nœud existant
sunbeam cluster join --role compute --role storage "$(cat noeud2.token)"   # sur le nouveau nœud
sunbeam cluster list
```

Le nom doit être un FQDN. Le jeton n’est valable que pour ce nœud. Le nouveau nœud doit d’abord être préparé :

```bash
sunbeam prepare-node-script | bash -x     # sur le nouveau nœud, compte non root avec sudo
```

✅ `sunbeam prepare-node-script` (sans `| bash`) affiche le script : vérification d’Ubuntu 24.04 (« noble »), refus de s’exécuter en root, installation des prérequis. **À relire avant de l’exécuter**.

**Étendre le plan de contrôle** ⚠ non exécutée

```bash
sunbeam cluster resize --topology multi
```

À lancer après l’ajout de nœuds de contrôle, pour passer les services en haute disponibilité (`auto`, `single`, `multi`, `large`).

**Retirer un nœud** ⚠ non exécutée

```bash
sunbeam cluster remove noeud2.lab.test
```

Migrer d’abord les instances du nœud. `--force` ignore les contrôles de sécurité : à éviter.

**Mettre à jour** ⚠ non exécutée

```bash
sunbeam cluster refresh                 # aligne les charms sur la révision du snap
sunbeam cluster refresh mysql           # met à jour un composant précis (k8s, mysql, vault)
```

Procédure d’entreprise : `sunbeam manifest generate -f ~/avant.yaml`, snapshot des VM du lab (ou sauvegarde), `refresh`, puis `juju status` et les contrôles du guide apprenant (étape 10).

**Proxy d’entreprise** ⚠ non exécutée

```bash
sunbeam proxy set --http-proxy http://proxy.entreprise:3128 --https-proxy http://proxy.entreprise:3128 \
  --no-proxy "localhost,127.0.0.1,192.168.56.0/24,.lab.test"
```

Le `--no-proxy` doit couvrir le réseau de gestion et les noms internes, sinon les services ne se parlent plus.

**Fédération d’identité** ⚠ non exécutée (voir module 11)

```bash
sunbeam identity provider list                        # ✅ exécutée : seul « Keystone Credentials » (Built-in)
sunbeam identity provider get-oidc-redirect-url -h
sunbeam identity provider add -h
```

---

## 4. Les fonctionnalités optionnelles

### 4.1 Le principe

```bash
sunbeam enable <fonctionnalité> [options]
sunbeam disable <fonctionnalité>
sunbeam list-features
```

Activer une fonctionnalité déploie ses charms dans le modèle `openstack`, les relie aux services existants et ajoute ses points d’accès au catalogue Keystone. Désactiver retire le tout.

| Fonctionnalité | Service OpenStack | Usage en entreprise | Dépendances, remarques |
|---|---|---|---|
| `orchestration` | Heat | Décrire une infrastructure en modèles (HOT), la recréer à l’identique | — |
| `loadbalancer` | Octavia | Répartiteurs de charge pour les applications | Lourd (VM amphores) |
| `secrets` | Barbican | Stocker clés et certificats des applications | Nécessite `vault` |
| `vault` | Vault | Coffre-fort de la plateforme | `--dev-mode` réservé aux tests |
| `tls` | — | Chiffrer les API : `self-signed`, `ca` (CA d’entreprise) ou `vault` | Corrige l’Horizon en HTTP du lab |
| `ldap` | Keystone | Brancher un annuaire LDAP ou Active Directory sur un domaine | — |
| `dns` | Designate | DNS des instances | Demande des serveurs de noms délégués |
| `telemetry` | Ceilometer, Gnocchi, Aodh | Métrologie, refacturation | Lourd |
| `observability` | Prometheus, Grafana, Loki (COS) | Supervision de la plateforme | `embedded` : très lourd ; `external` : vers une COS existante |
| `instance-recovery` | Masakari | Redémarrage des VM après panne d’un hôte | Utile en multinœud |
| `resource-optimization` | Watcher | Optimisation du placement | — |
| `images-sync` | — | Synchronisation d’images Ubuntu | Accès Internet |
| `validation` | Tempest | Tests de conformité du cloud | — |
| `caas` | Magnum | Clusters Kubernetes à la demande | Nécessite `orchestration`, `loadbalancer`, `secrets` |
| `pro` | — | Ubuntu Pro sur tous les nœuds | Jeton Ubuntu Pro |

### 4.2 Ce qu’une activation fait réellement

Observé sur le lab avec `sunbeam enable orchestration` (démo 5) :

1. Juju déploie deux applications dans le modèle `openstack` : `heat` (pod `heat-0`) et `heat-mysql-router` (accès à MySQL) ;
2. elles passent par `maintenance` (installation), puis `waiting` (relations), puis `active` ;
3. Keystone reçoit deux nouveaux services au catalogue, `heat` (`orchestration`) et `heat-cfn` (`cloudformation`) ;
4. `sunbeam list-features` affiche `orchestration … X`.

Pendant l’opération, suivre l’avancement dans une seconde session :

```bash
watch -n 10 "juju status -m openstack --color=false | grep -i heat"
```

Voir la démo 5 pour la procédure complète.

---

## 5. Pièges et diagnostics observés sur le lab

### 5.1 `Failed to login to Juju Controller`

Symptôme : `sunbeam cluster list` échoue, la même commande réussit après `sunbeam utils juju-login`. Cause : session Juju expirée (fréquent après un redémarrage). Réflexe : `sunbeam utils juju-login`.

### 5.2 Fichiers écrits « nulle part »

Symptôme : `Generated manifest is at /tmp/…`, fichier introuvable. Cause : confinement du snap. Réflexe : chemins dans `~/`.

### 5.3 `Storage: blocked` dans `sunbeam cluster list`

```bash
juju status -m admin/openstack-machines microceph
juju show-status-log -m admin/openstack-machines microceph/0
sudo ceph -s | grep -E "health|osd:|pgs:"
```

Observé sur le lab : `microceph/0 blocked (workload) Error in charm (see logs): timed out`, alors que Ceph est sain (`1 osds: 1 up`, `289 active+clean`). Le charm n’obtient pas à temps la réponse de MicroCeph lors de son contrôle périodique ; Ceph continue de servir Glance et Cinder. À surveiller : si `ceph -s` se dégrade, c’est un vrai problème (voir l’étape 10.2 du guide apprenant).

### 5.4 `sunbeam deployment show` et `manifest generate` exposent des secrets

Ne pas les projeter, ne pas les joindre à un ticket sans les expurger.

---

## 6. Démos sur le projet

Chaque démo est autonome, testée sur le lab, et laisse le cloud dans son état initial. Elles portent sur la commande `sunbeam` ; les quelques commandes `openstack` qui y figurent servent seulement à vérifier un résultat ou à nettoyer (signalées par 🔎). Les commandes `juju` et `openstack` feront l’objet d’un guide à part.

### Démo 1 : faire le tour d’un cloud Sunbeam en 5 commandes (5 min) ✅

```bash
sunbeam cluster list
sunbeam deployment list
sunbeam list-features
sunbeam manifest show latest
sunbeam plans list
```

À commenter : un nœud, trois rôles actifs ; un déploiement `star-heron` ; aucune fonctionnalité optionnelle ; les adresses du manifeste sont celles du guide apprenant (`.201`, `.202`, `.203`, plage `.200–.220`) ; sept plans Terraform, un par couche.

### Démo 2 : préparer les accès d’une équipe (10 min) ✅

```bash
sunbeam openrc > ~/admin-openrc.nouveau
sunbeam cloud-config -u -f ~/clouds-equipe.yaml --no-hints
sunbeam cloud-config -a -c sunbeam-admin -u -f ~/clouds-equipe.yaml --no-hints
OS_CLIENT_CONFIG_FILE=~/clouds-equipe.yaml openstack --os-cloud sunbeam server list
OS_CLIENT_CONFIG_FILE=~/clouds-equipe.yaml openstack --os-cloud sunbeam-admin hypervisor list
sunbeam dashboard-url
rm -f ~/admin-openrc.nouveau ~/clouds-equipe.yaml
```

À commenter : les deux lignes `openstack --os-cloud` (🔎) vérifient seulement que le fichier fonctionne. Un seul fichier `clouds.yaml` pour plusieurs comptes ; `-u` obligatoire pour écrire ; les fichiers contiennent des mots de passe (droits `rw-rw----` sur le lab : à restreindre en `chmod 600`).

### Démo 3 : une VM de test en une commande (5 min) ✅

```bash
sunbeam launch --name lab-launch
```

```text
Launching an OpenStack instance ...
Access the instance by running the following command:
`ssh -i /home/vagrant/snap/openstack/1101/sunbeam ubuntu@192.168.57.199`
```

Mesuré sur le lab : 1 min 25 s. La commande prend l’image `ubuntu`, le flavor `m1.tiny`, le réseau `demo-network`, le groupe `default` du projet `demo`, crée la clé `sunbeam` (fichier dans `~/snap/openstack/<révision>/`) et associe une IP flottante. Connexion vérifiée depuis Windows :

```powershell
ping 192.168.57.199
```

🔎 Nettoyage (le projet `demo` est celui de `~/demo-openrc`) :

```bash
source ~/demo-openrc
FIP=$(openstack server show lab-launch -f json -c addresses | python3 -c "import sys,json; a=json.load(sys.stdin)['addresses']; print([i for v in a.values() for i in v if i.startswith('192.168.57.')][0])")
openstack server delete lab-launch --wait
openstack floating ip delete "$FIP"
openstack keypair delete sunbeam
rm -f ~/snap/openstack/*/sunbeam
```

⚠ `sunbeam launch` sert à **vérifier** un cloud, pas à créer des VM de production : il utilise toujours le projet de démonstration.

### Démo 4 : documenter la configuration (5 min) ✅

```bash
sunbeam manifest generate -f ~/manifest-lab.yaml
wc -l ~/manifest-lab.yaml
grep -n -E "^[a-z]+:|^  [a-z-]+:$" ~/manifest-lab.yaml | head -25
grep -n -A4 "^  orchestration:" ~/manifest-lab.yaml
rm -f ~/manifest-lab.yaml
```

À commenter : le manifeste a deux parties, `core` (réseaux, adresses, base de données, canaux des charms) et `features` (une section par fonctionnalité). Les sections des fonctionnalités sont des **modèles commentés** (`# config: null`) : on les décommente pour préparer un manifeste qu’on appliquera avec `-m`. **Activer une fonctionnalité ne modifie pas le manifeste** (vérifié avec Heat) : l’état activé ou non est conservé par clusterd, et se lit avec `sunbeam list-features`. Le fichier contient des mots de passe : le supprimer après la démo.

### Démo 5 : activer Heat, déployer une pile, désactiver (30 min) ✅

Durées mesurées sur le lab : **activation 9 min 50 s**, désactivation 4 min 08 s. Lancer l’activation au début d’une pause, ou pendant une autre explication.

**1. Activer**

```bash
sunbeam list-features | grep orchestration
sunbeam enable orchestration
```

```text
OpenStack orchestration application enabled.
```

Dans une seconde session, pendant l’activation :

```bash
watch -n 10 "juju status -m openstack --color=false | grep -i heat"
```

On voit `heat` passer de `maintenance (install) installing charm software` à `active`, et `heat-mysql-router` de `waiting` à `active`.

**2. 🔎 Vérifier**

```bash
sunbeam list-features | grep orchestration
source ~/admin-openrc
openstack service list | grep -i -E "heat|orchestration"
openstack orchestration service list -c Binary -c Status
```

✅ **Attendu :** `orchestration … X` ; services `heat` (`orchestration`) et `heat-cfn` (`cloudformation`) ; quatre `heat-engine` à l’état `up`.

**3. 🔎 Vérifier avec une pile** (projet `demo`)

```bash
cat > ~/lab-pile.yaml <<'EOF'
heat_template_version: 2021-04-16
description: Reseau et groupe de securite de demonstration
parameters:
  cidr:
    type: string
    default: 10.40.0.0/24
resources:
  reseau:
    type: OS::Neutron::Net
    properties: {name: lab-heat-net}
  sous_reseau:
    type: OS::Neutron::Subnet
    properties: {name: lab-heat-subnet, network: {get_resource: reseau}, cidr: {get_param: cidr}}
  groupe:
    type: OS::Neutron::SecurityGroup
    properties:
      name: lab-heat-sg
      rules: [{protocol: tcp, port_range_min: 22, port_range_max: 22, remote_ip_prefix: 192.168.57.1/32}]
outputs:
  reseau_id: {value: {get_resource: reseau}}
EOF
source ~/demo-openrc
openstack stack create lab-pile -t ~/lab-pile.yaml --wait
openstack stack resource list lab-pile -c resource_name -c resource_type -c resource_status
openstack stack output show lab-pile reseau_id -c output_value -f value
```

```text
[lab-pile.reseau]: CREATE_COMPLETE  state changed
[lab-pile.groupe]: CREATE_COMPLETE  state changed
[lab-pile.sous_reseau]: CREATE_COMPLETE  state changed
[lab-pile]: CREATE_COMPLETE  Stack CREATE completed successfully
```

À commenter : Heat calcule les dépendances (le sous-réseau attend le réseau, le groupe se crée en parallèle) ; la pile entière se crée en 10 secondes, et se supprime d’une seule commande, sans oubli.

```bash
openstack stack delete lab-pile --yes --wait
openstack network list --name lab-heat-net
rm -f ~/lab-pile.yaml
```

✅ **Attendu :** `DELETE_COMPLETE`, et plus aucun réseau `lab-heat-net`.

**4. Désactiver**

```bash
sunbeam disable orchestration
sunbeam list-features | grep orchestration
juju status -m openstack --color=false | grep -c -i heat
```

```text
OpenStack orchestration application disabled.
```

✅ **Attendu :** la fonctionnalité n’est plus cochée ; `0` application Heat dans Juju.

**5. 🔎 ⚠ Nettoyer ce que la désactivation laisse dans Keystone**

Observé sur le lab : `sunbeam disable orchestration` retire les applications, mais **pas** les objets que Heat avait créés dans Keystone. Le catalogue annonce encore un service Orchestration qui n’existe plus.

```bash
source ~/admin-openrc
openstack service list | grep -i heat
openstack domain list | grep heat
openstack user list --domain service_domain | grep -i heat
openstack role list | grep -i heat
```

| Résidu | Exemple sur le lab |
|---|---|
| Services et leurs 6 points d’accès | `heat` (orchestration), `heat-cfn` (cloudformation) |
| Domaine des utilisateurs de piles | `heat` (avec le compte `heat_domain_admin-…`) |
| Compte de service | `svc_heat-…` dans `service_domain` |
| Rôle | `heat_stack_user` |

Suppression :

```bash
openstack service delete heat heat-cfn
for u in $(openstack user list --domain service_domain -f value -c Name | grep "^svc_heat"); do
  openstack user delete --domain service_domain "$u"
done
openstack role delete heat_stack_user
openstack domain set --disable heat
openstack domain delete heat
openstack catalog list -f value -c Name
```

✅ **Attendu :** plus aucune entrée Heat au catalogue ; domaines `admin_domain`, `service_domain`, `Default`, `users`. Supprimer un service supprime aussi ses points d’accès.

À commenter : en entreprise, vérifier le catalogue et les comptes **après** chaque désactivation ; une fonctionnalité retirée ne doit pas laisser de compte de service actif.

---

## 7. Récapitulatif des tests

| Commande | Exécutée sur le lab | Résultat |
|---|---|---|
| `sunbeam -h` et l’aide de chaque groupe | ✅ | Carte du § 2 |
| `cluster list` (`-f table`, `-f yaml`) | ✅ | Après `utils juju-login` |
| `deployment list`, `deployment show star-heron` | ✅ | `show` expose des secrets |
| `list-features`, `list-feature-gates` | ✅ | |
| `manifest list`, `manifest show latest`, `manifest generate -f ~/…` | ✅ | Fichier invisible si écrit dans `/tmp` |
| `plans list` | ✅ | |
| `proxy show`, `storage list`, `identity provider list` | ✅ | Vides, sauf le fournisseur intégré |
| `openrc`, `cloud-config -u`, `cloud-config -a -c … -u` | ✅ | `clouds.yaml` utilisable par `openstack --os-cloud` |
| `dashboard-url`, `dashboard url` | ✅ | |
| `utils juju-login` | ✅ | Corrige « Failed to login to Juju Controller » |
| `prepare-node-script` (affichage seul) | ✅ | |
| `launch --name …` | ✅ | VM joignable depuis Windows, 1 min 25 s |
| `enable orchestration`, `disable orchestration` | ✅ | 9 min 50 s et 4 min 08 s ; résidus Keystone à nettoyer |
| `cluster add`, `join`, `remove`, `resize`, `refresh`, `bootstrap` | ⚠ non exécutées | Nœud unique, ou risque de modifier le lab |
| `proxy set`, `identity provider add`, `storage add`, `plans unlock`, autres `enable` | ⚠ non exécutées | Syntaxe issue de l’aide (`-h`) |

État du lab à la fin des tests : identique à l’état initial (aucune fonctionnalité activée, aucune ressource de test, catalogue sans Heat).

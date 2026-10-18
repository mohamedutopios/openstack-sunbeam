# Démo 6 : audit et traçabilité

> **Statut : 6.1 à 6.4 rejouées sur le lab le 18 octobre 2026** ; les sorties citées sont celles de ce jour. La centralisation (6.5) n’a **pas** été activée : elle ajoute une pile lourde (Loki, Prometheus, Grafana) à un lab déjà à la limite de sa mémoire. Support : module 13.

## Objectif

Répondre à la question de tout audit : **qui a fait quoi, quand, d’où, avec quel résultat ?** On apprend où sont les journaux d’un cloud Sunbeam, comment **corréler** les lignes de plusieurs services grâce aux identifiants de requête, puis comment **centraliser** les journaux pour ne plus aller les chercher pod par pod.

## 6.1 Où sont les journaux d’un cloud Sunbeam

| Source | Ce qu’elle trace | Comment la lire |
|---|---|---|
| **Services OpenStack** (pods Kubernetes) | Chaque appel d’API : requête, utilisateur, projet, code de réponse, durée | `sudo k8s kubectl -n openstack logs <pod> -c <conteneur>` |
| **Services de la machine** (snaps) | nova-compute, OVN, cinder-volume | `sudo journalctl -u 'snap.openstack-hypervisor.*'`, `sudo journalctl -u snap.cinder-volume.cinder-volume` |
| **Charms** (Juju) | Les opérations d’exploitation : configurations, relations, erreurs | `juju debug-log`, `juju show-status-log` (guide `4-juju-cli/`) |
| **Actions sur les VM** | Création, arrêt, suppression d’une VM, avec l’identifiant de requête | `openstack server event list` |
| **Keycloak** | Connexions, échecs, verrouillages des personnes fédérées | Console : **Events** (démo 3, étape 3.3) |

Trouver le pod et le conteneur d’un service : `sudo k8s kubectl -n openstack get pod <service>-0 -o jsonpath='{.spec.containers[*].name}'` (guide `4-juju-cli/DEMOS-PANNES.md`, partie 2).

## 6.2 Lire une ligne de journal d’API

Générer une action, avec le compte `demo`, et garder l’identifiant de requête que Neutron renvoie :

```bash
source ~/demo-openrc
REQ=$(openstack --debug network create lab-sec-trace 2>&1 | grep -o "X-Openstack-Request-Id: req-[0-9a-f-]*" | tail -1 | cut -d" " -f2)
echo "$REQ"
```

**La commande, mot par mot :** `--debug` affiche les échanges HTTP du client, dont l’en-tête de réponse `X-Openstack-Request-Id` ; `grep -o` l’isole ; `cut -d" " -f2` ne garde que la valeur.

✅ **Attendu** (18 octobre) : `req-60273723-1489-41cf-aa79-988883124a80`.

Les lignes de Neutron pour cette requête :

```bash
sudo k8s kubectl -n openstack logs neutron-0 -c neutron-server --since=10m | grep "$REQ" | cut -c1-200
```

✅ **Attendu** (18 octobre, extrait) :

```text
… INFO oslo.messaging.notification.audit.http.request [None req-f8e6c1da-… - - - - - -] {"message_id": …
… WARNING neutron_lib.context [None req-60273723-… 18e26245… acb09763… - - fc1d71a3… fc1d71a3…] …
… INFO neutron.db.segments_db [None req-60273723-… 18e26245… acb09763… - - fc1d71a3… fc1d71a3…] …
… INFO oslo.messaging.notification.audit.http.response [None req-60273723-… 18e26245… acb09763… …] …
… INFO neutron.wsgi [None req-60273723-… 18e26245… acb09763… …] … "POST /openstack-neutron/v2.0/networks HTTP/1.1" status: 201 …
```

| Ligne | Contenu |
|---|---|
| `audit.http.request` | L’**événement d’audit** de la requête, au format **CADF**, écrit à l’arrivée. Son préfixe porte un autre identifiant et des `-` : le contexte de la requête n’est pas encore établi ; c’est son contenu JSON qui contient l’identifiant cherché |
| Lignes de traitement (`neutron.db…`, `neutron_lib.context`…) | Ce que Neutron a fait, ligne par ligne, toutes avec le même identifiant |
| `audit.http.response` | L’événement d’audit de la réponse, avec son résultat |
| `neutron.wsgi` | La ligne d’accès HTTP : méthode, chemin, **code de réponse** (`201` = créé), durée |

**Comment lire le préfixe `[None req-60273723-… 18e26245… acb09763… - - fc1d71a3… fc1d71a3…]` :**

| Champ | Valeur | Signification |
|---|---|---|
| 1 | `None` | L’identifiant de requête **global** : vide pour un appel direct d’un client ; rempli quand un service en appelle un autre (6.4) |
| 2 | `req-60273723-…` | L’identifiant de **cette** requête : celui que le client a reçu |
| 3, 4 | `18e26245…`, `acb09763…` | **Qui** : l’utilisateur et le projet du jeton |
| 5, 6 | `-`, `-` | Le domaine et la portée système du jeton : vides pour un jeton de projet |
| 7, 8 | `fc1d71a3…` (deux fois) | Le domaine de l’utilisateur, puis celui du projet |

**Qui est ce `user_id` ?**

```bash
source ~/admin-openrc
openstack user show 18e2624580f345e895073662440271da -c name -c domain_id     # 🔁 l'identifiant lu dans la ligne
openstack domain show users -c id -f value
```

✅ **Attendu** (18 octobre) : `demo`, dans le domaine `fc1d71a3e63c456bbf235a0bef55af1c`, qui est bien l’identifiant du domaine `users`.

### Le format CADF

Les lignes `audit.http.*` sont produites par l’**intergiciel d’audit** (*audit middleware*) placé devant l’API (fichier `api_audit_map.conf` présent dans les conteneurs, par exemple `/etc/glance/api_audit_map.conf`). Elles suivent le format **CADF** (*Cloud Auditing Data Federation*), qui répond exactement aux questions d’un audit. Pour les lire, extraire le JSON de la ligne et n’en garder que les champs utiles :

```bash
sudo k8s kubectl -n openstack logs neutron-0 -c neutron-server --since=15m | grep "$REQ" | grep "audit.http" \
  | sed 's/^[^{]*//' | python3 -c '
import sys, json
for line in sys.stdin:
    m, _ = json.JSONDecoder().raw_decode(line)
    p = m["payload"]; i = p["initiator"]
    print(m["event_type"], "|", p["eventTime"], "|", p["action"], "|", p["outcome"], "|", p.get("reason", ""))
    print("   initiator:", i.get("name"), i.get("id"), "projet", i.get("project_id"), "depuis", i.get("host", {}).get("address"))
    print("   target:", p["target"].get("typeURI"), "| chemin:", p.get("requestPath"))'
```

**La commande, mot par mot :** `sed 's/^[^{]*//'` retire tout ce qui précède le premier `{` (horodatage, niveau, préfixe) ; `raw_decode` lit le premier objet JSON de la ligne, sans se soucier de ce qui suit ; le programme affiche les champs CADF.

✅ **Attendu** (18 octobre) :

```text
audit.http.request | 2026-10-18T10:38:16.584955+0000 | create | pending |
   initiator: demo 18e2624580f345e895073662440271da projet acb0976350464f5baa07665d23aad177 depuis 10.1.0.196
   target: service/network/networks | chemin: /openstack-neutron/v2.0/networks
audit.http.response | 2026-10-18T10:38:16.584955+0000 | create | success | {'reasonType': 'HTTP', 'reasonCode': '201'}
   initiator: demo 18e2624580f345e895073662440271da projet acb0976350464f5baa07665d23aad177 depuis 10.1.0.196
   target: service/network/networks | chemin: /openstack-neutron/v2.0/networks
```

| Champ CADF | Question | Sur le lab |
|---|---|---|
| `initiator` | **Qui** (utilisateur, projet, adresse IP) | `demo`, son projet, `10.1.0.196` |
| `action` | **Quoi** (`create`, `read`, `update`, `delete`) | `create` |
| `target` | **Sur quoi** (le type de la ressource) | `service/network/networks` |
| `eventTime` | **Quand** | `2026-10-18T10:38:16` |
| `outcome`, `reason` | **Avec quel résultat** | `pending` à l’arrivée, `success` et `201` à la réponse |

`10.1.0.196` est l’adresse de Lab-Sunbeam côté Kubernetes (interface `cilium_host`) : la commande a été lancée depuis Lab-Sunbeam lui-même. Depuis une autre machine, c’est son adresse qui apparaît (6.3).

## 6.3 Tracer les refus et les échecs d’authentification

**Un refus d’autorisation (403) :**

```bash
source ~/demo-openrc
openstack --debug user list 2>&1 | grep -o "X-Openstack-Request-Id: req-[0-9a-f-]*" | tail -1
sudo k8s kubectl -n openstack logs keystone-0 -c keystone --since=5m | grep -E "ForbiddenAction|\" 403 " | tail -2 | cut -c1-260
```

✅ **Attendu** (18 octobre) :

```text
… ERROR keystone.server.flask.application keystone.exception.ForbiddenAction: You are not authorized to perform the requested action: identity:list_users.
… 10.1.0.196 - - [18/Oct/2026:10:39:19 +0000] "GET /openstack-keystone/v3/users HTTP/1.1" 403 132 750333 "-" "openstacksdk/4.4.0 …"
```

On sait **quoi** a été tenté, **quelle règle** a refusé (`identity:list_users`), **d’où** et **quand**. Le **qui** se lit dans les lignes de Keystone qui portent l’identifiant de requête affiché par la première commande.

**Un échec d’authentification (401) :**

```bash
OS_PASSWORD=faux openstack token issue
sudo k8s kubectl -n openstack logs keystone-0 -c keystone --since=5m | grep -E "\" 401 |Authorization failed" | tail -2 | cut -c1-260
```

✅ **Attendu** (18 octobre) :

```text
The request you have made requires authentication. (HTTP 401) (Request-ID: req-a39084e5-…)
… WARNING keystone.server.flask.application [None req-a39084e5-… - - - - - -] Authorization failed. The request you have made requires authentication. from 10.1.0.196: …
… 10.1.0.196 - - [18/Oct/2026:10:39:27 +0000] "POST /openstack-keystone/v3/auth/tokens HTTP/1.1" 401 109 …
```

**Comment le lire :** la ligne dit **d’où** vient l’échec, mais **pas quel compte** était visé : Keystone n’écrit pas le nom d’utilisateur d’un échec. L’adresse est bien celle du client réel : depuis Lab-Services, le même essai (`curl` vers `/v3/auth/tokens` avec un mauvais mot de passe) apparaît avec `192.168.56.20` (constaté). Pour compter les échecs **par compte**, il faut la source qui les nomme : les événements de Keycloak pour les comptes fédérés (démo 3, `LOGIN_ERROR` avec l’utilisateur).

## 6.4 Corréler : retrouver qui a supprimé une VM

**Le scénario** (support, module 13) : une VM a disparu. Retrouver **qui** l’a supprimée, **quand**, et **toutes** les lignes associées.

**Préparer** (formateur, avec le compte `demo`) :

```bash
source ~/demo-openrc
openstack server create lab-sec-victime --image ubuntu --flavor m1.tiny --network demo-network --wait -c id -f value
openstack server delete lab-sec-victime --wait
```

✅ **Attendu** (18 octobre) : une ligne vide, l’identifiant de la VM, puis la main rendue (40 secondes en tout).

**Enquêter** (apprenant, avec le compte `admin`) :

1. Retrouver la VM supprimée, par son nom :

   ```bash
   source ~/admin-openrc
   openstack server list --all-projects --deleted --name lab-sec-victime -c ID -c Name -c Status
   VM=$(openstack server list --all-projects --deleted --name lab-sec-victime -f value -c ID | head -1)
   ```

   ✅ **Attendu** (18 octobre) : une ligne, `5a73c6e0-5029-4994-839d-0556753e44b4 | lab-sec-victime | DELETED`.

2. Son historique, avec les identifiants de requête :

   ```bash
   openstack server event list "$VM"
   REQ=$(openstack server event list "$VM" -f value -c 'Request ID' -c Action | awk '$2=="delete"{print $1}')
   openstack server event show "$VM" "$REQ"
   ```

   ✅ **Attendu** (18 octobre) :

   ```text
   | req-9d4e7e12-0aef-4c78-990f-7714c7d1f4d6 | 5a73c6e0-… | delete | 2026-10-18T10:40:43.000000 |
   | req-a306bcd2-5d30-4e7e-9849-c8a8b41830c1 | 5a73c6e0-… | create | 2026-10-18T10:40:17.000000 |
   ```

   puis, pour la suppression : `action delete`, l’événement `compute_terminate_instance` (`result='Success'`, `host='sunbeam'`), `project_id acb09763…` et **`user_id 18e26245…`**.

3. Les lignes de Nova pour cette requête :

   ```bash
   sudo k8s kubectl -n openstack logs nova-0 -c nova-api --since=1h | grep "$REQ" | cut -c1-200
   sudo journalctl --since "-1h" -u 'snap.openstack-hypervisor.*' | grep "$REQ" | cut -c1-200
   ```

   ✅ **Attendu** (18 octobre) : dans `nova-api`, les lignes `audit.http.request`, `audit.http.response` et `nova.api.openstack.requestlog` de la suppression ; dans le journal de la machine, 7 lignes de `nova-compute` (`nova.compute.manager`, `os_vif`, `nova.virt.libvirt.driver`…) : l’API a reçu l’ordre, l’hyperviseur l’a exécuté.

   Ces lignes montrent aussi l’enquêteur : `GET /openstack-nova/v2.1/servers/5a73c6e0-…/os-instance-actions/req-9d4e7e12-…` par l’utilisateur `admin`. **L’audit est lui-même tracé.**

4. **Suivre la requête dans les autres services.** Quand Nova supprime une VM, il demande à Neutron de supprimer son port. Il transmet alors son identifiant de requête comme identifiant **global**, en tête du préfixe :

   ```bash
   sudo k8s kubectl -n openstack logs neutron-0 -c neutron-server --since=1h | grep "$REQ" | cut -c1-200
   ```

   ✅ **Attendu** (18 octobre) :

   ```text
   … neutron.wsgi [req-9d4e7e12-0aef-4c78-990f-7714c7d1f4d6 req-3d3ca3d1-8b5c-43b9-b1eb-5c24c573eb4f 18e26245… acb09763… …
   ```

   **Comment le lire :** en tête, `req-9d4e7e12…`, l’identifiant **global** de la suppression demandée à Nova ; ensuite `req-3d3ca3d1…`, celui de la requête propre à Neutron ; puis l’utilisateur `demo` et son projet. C’est la **corrélation entre services** : un seul identifiant pour suivre une action à travers tout le cloud.

5. Nommer l’auteur :

   ```bash
   openstack user show 18e2624580f345e895073662440271da -c name -c domain_id     # 🔁 le user_id de l'étape 2
   ```

✅ **Conclusion** (18 octobre) : « la VM `lab-sec-victime` a été supprimée par `demo` (domaine `users`) le 18 octobre à 10:40:43, par la requête `req-9d4e7e12…` ; `nova-compute` l’a détruite sur l’hôte `sunbeam`, et Nova a supprimé son port dans Neutron sous le même identifiant global ».

## 6.5 Centraliser les journaux

**Le problème :** les journaux vivent dans chaque pod et sur chaque machine. Ils **disparaissent** avec un pod recréé, et l’enquête de 6.4 demande une commande par service. En entreprise, on les **centralise** : un seul endroit, une conservation maîtrisée, des recherches et des alertes.

Sunbeam propose la fonctionnalité `observability` :

| Variante | Effet |
|---|---|
| `observability.embedded` | Déploie, dans le cloud lui-même, une pile d’observabilité de Canonical (COS) : **Loki** (journaux), Prometheus (métriques), **Grafana** (consultation) |
| `observability.external` | Envoie journaux et métriques vers une pile COS **déjà existante** |

⚠ **Ressources :** la variante `embedded` ajoute plusieurs applications au cloud. Sur le lab, dont le PC n’avait plus que 0,6 Go de mémoire libre le 17 octobre, elle risque de le déstabiliser. À tester d’abord sur un instantané, ou à présenter en démonstration commentée.

### Activer (⚠ non testé sur le lab)

```bash
sunbeam enable observability embedded --help
sunbeam enable observability embedded
sunbeam list-features | grep observability
juju models
```

✅ **Attendu :** la fonctionnalité cochée, et un modèle Juju supplémentaire pour la pile d’observabilité (⚠ **à vérifier** : son nom exact dans `juju models`).

Adresse de Grafana et mot de passe d’administration :

```bash
sunbeam observability dashboard-url
juju run --model <modele d observabilite> grafana/leader get-admin-password
```

### Chercher dans Loki

Dans Grafana : **Explore** → source **Loki**. Requêtes en LogQL (⚠ **à vérifier** : les noms d’étiquettes réellement posées par l’agent de collecte, à lire dans la liste des étiquettes de l’explorateur) :

| But | Requête LogQL |
|---|---|
| Toutes les lignes d’une requête, tous services confondus | `{juju_model="openstack"} \|= "req-…"` |
| Les refus d’autorisation de Keystone | `{juju_model="openstack", juju_application="keystone"} \|= "ForbiddenAction"` |
| Les suppressions de VM | `{juju_model="openstack", juju_application="nova"} \|= "DELETE /openstack-nova"` |

**Comment le lire :** l’enquête de 6.4, qui demandait une commande par pod, se fait en **une** recherche, et reste possible même après la recréation d’un pod.

## 6.6 Exploiter : alertes et conservation

**Des règles d’alerte utiles**, à écrire dans le gestionnaire d’alertes de la pile centralisée :

| Alerte | Signal | Pourquoi |
|---|---|---|
| Essais de mots de passe | Plus de 5 `401` en 5 minutes depuis une même adresse (lignes d’accès de Keystone), ou plus de 5 `LOGIN_ERROR` pour un même compte (événements de Keycloak) | Attaque par essais (démo 3). Keystone ne nomme pas le compte visé (6.3) |
| Escalade de privilèges | Une affectation du rôle `admin` (`POST`/`PUT` sur `…/role_assignments` ou `…/roles/<admin>`) | `admin` ne se donne jamais à la légère (`labs-iam/`, TP 2) |
| Ouverture réseau large | Une règle de groupe de sécurité avec `0.0.0.0/0` sur le port 22 | Contraire à la micro-segmentation (démo 5) |
| Refus en série | Beaucoup de `ForbiddenAction` d’un même utilisateur | Compte compromis qui sonde ses droits |

**La conservation :** selon la politique de l’entreprise et la réglementation. La CNIL recommande en général une durée de **six mois à un an** pour les journaux de traçabilité ; le guide de l’ANSSI sur l’architecture d’un système de journalisation détaille la collecte, la protection et l’horodatage. Deux exigences pratiques :

- **horloges synchronisées** sur toutes les machines : sans elles, impossible d’ordonner les événements (constaté sur le lab après une veille du PC : un saut d’horloge d’une heure et demie) ;
- **journaux protégés** contre la modification : ils sont envoyés hors des machines surveillées, dans la pile centralisée.

## Nettoyage

```bash
source ~/demo-openrc
openstack network delete lab-sec-trace
```

La VM `lab-sec-victime` est déjà supprimée. Si la centralisation a été activée et qu’on veut revenir en arrière : `sunbeam disable observability embedded` (⚠ à vérifier), ou restauration de l’instantané.

## À retenir

- Les journaux d’un cloud Sunbeam sont **dispersés** : pods, snaps de la machine, charms, Keycloak.
- L’**identifiant de requête** relie ce que voit l’utilisateur aux lignes du serveur ; l’identifiant **global** suit une action d’un service à l’autre.
- Les lignes **CADF** répondent directement à l’audit : qui, quoi, sur quoi, quand, résultat.
- La **centralisation** (Loki, Grafana) rend l’enquête rapide et les journaux durables ; elle permet les **alertes**, et impose une **conservation** maîtrisée et des horloges synchronisées.

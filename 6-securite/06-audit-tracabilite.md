# Démo 6 : audit et traçabilité

> **Statut : rédigé, pas encore rejoué sur le lab.** Les formats de journaux cités (lignes d’audit, identifiants de requête) ont été observés sur le lab les 16 et 17 octobre 2026 (guides `5-openstack-cli/`, `4-juju-cli/DEMOS-PANNES.md`, TP 1 des `labs-iam/`). La centralisation (6.5) n’a pas été activée sur le lab. Support : module 13.

## Objectif

Répondre à la question de tout audit : **qui a fait quoi, quand, d’où, avec quel résultat ?** On apprend où sont les journaux d’un cloud Sunbeam, comment **corréler** les lignes de plusieurs services grâce aux identifiants de requête, puis comment **centraliser** les journaux pour ne plus aller les chercher pod par pod.

## 6.1 Où sont les journaux d’un cloud Sunbeam

| Source | Ce qu’elle trace | Comment la lire |
|---|---|---|
| **Services OpenStack** (pods Kubernetes) | Chaque appel d’API : requête, utilisateur, projet, code de réponse, durée | `sudo k8s kubectl -n openstack logs <pod> -c <conteneur>` |
| **Services de la machine** (snaps) | nova-compute, OVN, cinder-volume | `sudo journalctl -u 'snap.openstack-hypervisor.*'`, `sudo journalctl -u snap.cinder-volume.cinder-volume` |
| **Charms** (Juju) | Les opérations d’exploitation : configurations, relations, erreurs | `juju debug-log`, `juju show-status-log` (guide `4-juju-cli/`) |
| **Actions sur les VM** | Création, arrêt, suppression d’une VM, avec l’identifiant de requête | `openstack server event list` |
| **Keycloak** | Connexions, échecs, verrouillages des personnes fédérées | Console : **Events** (démo 2, étape 2.3) |

Trouver le pod et le conteneur d’un service : `sudo k8s kubectl -n openstack get pod <service>-0 -o jsonpath='{.spec.containers[*].name}'` (guide `4-juju-cli/DEMOS-PANNES.md`, partie 2).

## 6.2 Lire une ligne de journal d’API

Générer une action, avec le compte `demo` :

```bash
source ~/demo-openrc
openstack --debug network create lab-sec-trace 2>&1 | grep -o "X-Openstack-Request-Id: req-[0-9a-f-]*" | tail -1
```

✅ **Attendu :** l’identifiant de requête renvoyé par Neutron, par exemple `X-Openstack-Request-Id: req-…`. Le garder :

```bash
REQ=req-…        # 🔁 coller l'identifiant obtenu
```

Les lignes de Neutron pour cette requête :

```bash
sudo k8s kubectl -n openstack logs neutron-0 -c neutron-server --since=10m | grep "$REQ" | cut -c1-260
```

✅ **Attendu**, trois sortes de lignes (formats observés sur le lab) :

| Ligne | Contenu |
|---|---|
| `… INFO neutron.wsgi [None req-… <user_id> <project_id> - - …] 10.1.0.x,… "POST /openstack-neutron/v2.0/networks HTTP/1.1" status: 201 …` | La requête HTTP : méthode, chemin, **code de réponse** (`201` = créé), durée |
| `… oslo.messaging.notification.audit.http.request [None req-… <user_id> <project_id> …]` | L’**événement d’audit** de la requête, au format **CADF** |
| `… oslo.messaging.notification.audit.http.response [None req-… …]` | L’événement d’audit de la réponse, avec son résultat |

**Comment lire le préfixe `[None req-… <user_id> <project_id> - - <domaine>]` :**

| Champ | Signification |
|---|---|
| `None` | L’identifiant de requête **global** : vide pour un appel direct d’un client ; rempli quand un service en appelle un autre (6.4) |
| `req-…` | L’identifiant de **cette** requête : celui que le client a reçu |
| `<user_id>`, `<project_id>` | **Qui** : l’utilisateur et le projet du jeton |
| `<domaine>` | Le domaine du projet |

**Qui est ce `user_id` ?**

```bash
source ~/admin-openrc
openstack user show <user_id> -c name -c domain_id     # 🔁 l'identifiant lu dans la ligne
```

✅ **Attendu :** `demo`, dans le domaine `users`.

### Le format CADF

Les lignes `audit.http.*` sont produites par l’**intergiciel d’audit** (*audit middleware*) placé devant l’API (fichier `api_audit_map.conf` présent dans les conteneurs, par exemple `/etc/glance/api_audit_map.conf`). Elles suivent le format **CADF** (*Cloud Auditing Data Federation*), qui répond exactement aux questions d’un audit :

| Champ CADF | Question |
|---|---|
| `initiator` | **Qui** (utilisateur, projet, adresse IP) |
| `action` | **Quoi** (`create`, `read`, `update`, `delete`) |
| `target` | **Sur quoi** (le type et l’identifiant de la ressource) |
| `eventTime` | **Quand** |
| `outcome` | **Avec quel résultat** (`success`, `failure`) |
| `observer` | **Qui l’a constaté** (le service) |

⚠ **à vérifier :** le contenu complet des lignes `audit.http.*` sur le lab (`grep "$REQ" … | grep audit | head -1`), pour montrer ces champs aux apprenants.

## 6.3 Tracer les refus et les échecs d’authentification

**Un refus d’autorisation (403) :**

```bash
source ~/demo-openrc
openstack --debug user list 2>&1 | grep -o "X-Openstack-Request-Id: req-[0-9a-f-]*" | tail -1
sudo k8s kubectl -n openstack logs keystone-0 -c keystone --since=5m | grep -E "ForbiddenAction|\" 403 " | tail -2 | cut -c1-240
```

✅ **Attendu** (observé au TP 1 des `labs-iam/`, § 1.7) : une ligne `ERROR keystone.server.flask.application [None req-… <user_id> <project_id> …] keystone.exception.ForbiddenAction: You are not authorized to perform the requested action: identity:list_users.` et la ligne d’accès `"GET /openstack-keystone/v3/users HTTP/1.1" 403`. On sait **qui** a tenté **quoi**, et **quelle règle** a refusé.

**Un échec d’authentification (401) :**

```bash
OS_PASSWORD=faux openstack token issue
sudo k8s kubectl -n openstack logs keystone-0 -c keystone --since=5m | grep -E "\" 401 |Authorization failed|Unauthorized" | tail -3 | cut -c1-240
```

⚠ **à vérifier :** le texte exact de la ligne. On attend la ligne d’accès `"POST /openstack-keystone/v3/auth/tokens HTTP/1.1" 401` et un message de Keystone nommant l’utilisateur, avec l’adresse source. Une série de `401` pour un même compte est le signe d’une attaque par essais : c’est la règle d’alerte de 6.6.

## 6.4 Corréler : retrouver qui a supprimé une VM

**Le scénario** (support, module 13) : une VM a disparu. Retrouver **qui** l’a supprimée, **quand**, et **toutes** les lignes associées.

**Préparer** (formateur, avec le compte `demo`) :

```bash
source ~/demo-openrc
openstack server create lab-sec-victime --image ubuntu --flavor m1.tiny --network demo-network --wait -c id -f value
openstack server delete lab-sec-victime --wait
```

**Enquêter** (apprenant, avec le compte `admin`) :

1. Retrouver la VM supprimée, par son nom :

   ```bash
   source ~/admin-openrc
   openstack server list --all-projects --deleted --name lab-sec-victime -c ID -c Name -c Status
   ```

   ✅ **Attendu :** une ligne, `DELETED`, avec l’identifiant de la VM.

2. Son historique, avec les identifiants de requête :

   ```bash
   VM=<id>                                  # 🔁 l'identifiant obtenu
   openstack server event list "$VM"
   openstack server event show "$VM" "$(openstack server event list "$VM" -f value -c 'Request ID' -c Action | awk '$2=="delete"{print $1}')"
   ```

   ✅ **Attendu :** deux actions, `create` et `delete`, avec leur heure et leur `Request ID` ; le détail de la suppression donne `user_id` et `project_id` (⚠ **à vérifier** : présence de `user_id` dans `server event show` pour l’administrateur).

3. Les lignes de Nova pour cette requête :

   ```bash
   REQ=<request id de la suppression>       # 🔁
   sudo k8s kubectl -n openstack logs nova-0 -c nova-api --since=1h | grep "$REQ" | cut -c1-240
   ```

4. **Suivre la requête dans les autres services.** Quand Nova supprime une VM, il demande à Neutron de supprimer son port. Il transmet alors son identifiant de requête comme identifiant **global**, en tête du préfixe (`[req-<nova> req-<neutron> …]` au lieu de `[None req-… …]`) :

   ```bash
   sudo k8s kubectl -n openstack logs neutron-0 -c neutron-server --since=1h | grep "$REQ" | cut -c1-240
   ```

   ⚠ **à vérifier :** la présence de l’identifiant de Nova dans les lignes de Neutron. C’est la **corrélation entre services** : un seul identifiant pour suivre une action à travers tout le cloud.

5. Nommer l’auteur :

   ```bash
   openstack user show <user_id> -c name -c domain_id
   ```

✅ **Conclusion attendue :** « la VM `lab-sec-victime` a été supprimée par `demo` (domaine `users`), à telle heure, par la requête `req-…` ; Nova a ensuite supprimé son port dans Neutron ».

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
| Essais de mots de passe | Plus de 5 `401` en 5 minutes pour un même compte | Attaque par essais (démo 2) |
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

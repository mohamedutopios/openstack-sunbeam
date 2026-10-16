# TP 2 : les domaines

> Statut : rédigé, **pas encore rejoué sur le lab**. Voir les conventions dans [README.md](README.md).

**Module du support :** 9, les domaines dans OpenStack.
**Durée :** 2 h.
**Objectifs :** créer des espaces d’identité indépendants, constater ce qui est unique par domaine et ce qui est global, déléguer l’administration d’un domaine à un gestionnaire, et vérifier qu’il ne peut pas en sortir.

**Prérequis :** TP 1 fait (en particulier le relevé de Secure RBAC, § 1.6.1). Session préparée :

```bash
source ~/labs-iam/lab-env.sh
```

---

## 2.1 Créer les domaines du fil rouge

```bash
openstack domain create acme --description "Acme : SI interne"
openstack domain create partenaire --description "Partenaire externe, totalement separe"
openstack domain list
```

✅ **Attendu :** six domaines : les quatre de Sunbeam, plus `acme` et `partenaire`, tous `Enabled = True`.

❓ **Question :** à quoi sert un domaine qui n’a encore aucun utilisateur ? *Réponse : c’est un espace de noms et une frontière d’administration ; il accueillera ses utilisateurs, groupes, projets, et éventuellement sa propre source d’identité.*

---

## 2.2 Des espaces d’identité indépendants

### Scénario : deux Bruno

Acme et le partenaire ont chacun un collaborateur nommé Bruno, deux personnes différentes.

```bash
openstack user create bruno --domain acme --password "$LAB_PW" --description "Bruno, Acme"
openstack user create bruno --domain partenaire --password "$LAB_PW" --description "Bruno, partenaire"
openstack user list --domain acme
openstack user list --domain partenaire
```

✅ **Attendu :** les deux créations réussissent : le nom n’est unique **qu’au sein d’un domaine**. Les deux comptes ont deux identifiants différents.

```bash
openstack user show bruno
```

⛔ **Refus attendu :** `More than one user exists with the name 'bruno'`. Un nom seul est ambigu.

```bash
openstack user show bruno --domain acme -c id -c domain_id -c description
openstack user show bruno --domain partenaire -c id -c domain_id -c description
```

✅ **Attendu :** chaque commande désigne un compte précis.

⚠ **Piège :** dans les scripts, désigner un compte par **identifiant**, ou par **nom + domaine**, jamais par nom seul. La même règle vaut pour les projets et les groupes.

### Simulation : un même nom de projet dans deux domaines

```bash
openstack project create app --domain acme
openstack project create app --domain partenaire
openstack project create app --domain acme
```

✅ **Attendu :** les deux premières réussissent. ⛔ **Refus attendu** pour la troisième : `409 Conflict`, le nom `app` existe déjà dans `acme`.

```bash
openstack project delete app --domain acme
openstack project delete app --domain partenaire
```

### Ce qui est global, ce qui appartient à un domaine

```bash
openstack role list
openstack role create --domain acme acme-facturation
openstack role list --domain acme
```

✅ **Attendu :** les rôles standards (`reader`, `member`, `manager`, `admin`) sont **globaux** : ils existent pour tous les domaines. Le rôle `acme-facturation` n’existe que dans `acme`.

| Global (toute la plateforme) | Propre à un domaine |
|---|---|
| Rôles standards, services, catalogue, régions, quotas par défaut | Utilisateurs, groupes, projets, rôles propres au domaine, source d’identité (si configurée par domaine) |

❓ **Question :** un rôle propre au domaine sert à quoi, s’il n’est pas utilisé par les politiques des services ? *Réponse : un rôle propre au domaine n’est jamais vu par les services : il doit **impliquer** un rôle global (ex. `acme-facturation` implique `reader`) pour donner un droit. Il sert à nommer un profil métier propre au domaine.*

```bash
openstack implied role create acme-facturation --implied-role reader
openstack implied role list
```

---

## 2.3 L’administration globale

L’équipe plateforme (`admin`, projet `admin` du domaine `admin_domain`) voit et administre tous les domaines.

```bash
openstack project list --long
openstack user list --long | grep -E "acme|partenaire"
```

✅ **Attendu :** les objets d’`acme` et de `partenaire` sont visibles.

### Le piège du rôle `admin`

❓ **Question :** pour qu’Acme gère ses comptes, on envisage de donner à Bruno le rôle `admin` sur un projet d’Acme. Que se passe-t-il ?

```bash
openstack project create acme-si --domain acme --description "Racine de l'arbre d'Acme"
openstack role add --user bruno --user-domain acme --project acme-si --project-domain acme admin
os_as bruno acme projet acme-si acme -- user list --domain partenaire
os_as bruno acme projet acme-si acme -- project list --domain admin_domain
openstack role remove --user bruno --user-domain acme --project acme-si --project-domain acme admin
```

⚖ **dépend de Secure RBAC :** les commandes de Bruno **réussissent** dans la plupart des configurations. Le rôle `admin` n’est pas contenu par la cible sur laquelle il est donné : Bruno voit le partenaire et le domaine de la plateforme. C’est la démonstration de « admin quelque part, admin partout ». On retire tout de suite ce rôle.

⚠ **Piège :** ne jamais donner `admin` à un client ou à un gestionnaire de domaine. Le bon outil est le rôle `manager` sur le domaine (§ 2.4).

---

## 2.4 L’administration déléguée

### Scénario : Bruno devient gestionnaire du domaine Acme

```bash
openstack role list -f value -c Name | grep -x manager || echo "ATTENTION : role manager absent"
openstack role add --user bruno --user-domain acme --domain acme manager
openstack role assignment list --user bruno --user-domain acme --names
```

✅ **Attendu :** `bruno@acme` → `manager` sur le domaine `acme`. Si le rôle `manager` est absent, voir la variante en fin de paragraphe.

Bruno obtient un **jeton de domaine** :

```bash
os_as bruno acme domaine acme -- token issue -c domain_id -c expires
```

### Ce que Bruno doit pouvoir faire

⚖ **dépend de Secure RBAC** pour tout ce paragraphe : les résultats ci-dessous sont ceux du profil « gestionnaire de domaine » de Keystone 2024.1.

```bash
os_as bruno acme domaine acme -- user list --domain acme
os_as bruno acme domaine acme -- user create chloe --domain acme --password "$LAB_PW" --description "Chloe, developpeuse"
os_as bruno acme domaine acme -- user create emma --domain acme --password "$LAB_PW" --description "Emma, exploitation"
os_as bruno acme domaine acme -- user create david --domain acme --password "$LAB_PW" --description "David, auditeur"
os_as bruno acme domaine acme -- project create acme-dev --domain acme --parent acme-si
os_as bruno acme domaine acme -- role add --user chloe --user-domain acme --project acme-dev --project-domain acme member
os_as bruno acme domaine acme -- role assignment list --domain acme --names
```

✅ **Attendu :** Bruno crée les comptes et les projets d’Acme, et donne `member` à Chloé.

### Ce que Bruno ne doit pas pouvoir faire

```bash
os_as bruno acme domaine acme -- user list --domain partenaire
os_as bruno acme domaine acme -- user create intrus --domain partenaire --password "$LAB_PW"
os_as bruno acme domaine acme -- role add --user chloe --user-domain acme --project acme-dev --project-domain acme admin
os_as bruno acme domaine acme -- role add --user bruno --user-domain acme --domain acme admin
os_as bruno acme domaine acme -- domain create acme2
os_as bruno acme domaine acme -- hypervisor list
```

⛔ **Refus attendu (403)** pour chacune :

| Tentative | Pourquoi c’est refusé |
|---|---|
| Lister ou créer des comptes du partenaire | Hors de son domaine |
| Donner `admin` à Chloé ou à lui-même | Un gestionnaire ne délègue que `reader`, `member` et `manager` : il ne peut pas fabriquer d’administrateur |
| Créer un domaine | Action de la plateforme (portée système) |
| Lister les hyperviseurs | Action de la plateforme |

❓ **Question :** pourquoi est-il essentiel que Bruno ne puisse pas se donner `admin` ? *Réponse : sinon la délégation n’a pas de plafond ; un gestionnaire compromis deviendrait administrateur de toute la plateforme (§ 2.3).*

⚠ **Variante si les refus n’ont pas lieu :** relève dans le tableau du § 1.6.1 si `enforce_scope` et `enforce_new_defaults` sont actifs pour Keystone. Sans eux, le rôle `manager` n’a pas de règles propres et la délégation n’est pas bornée : c’est exactement ce que Secure RBAC corrige. Note l’écart pour le présenter aux apprenants.

**Variante si `manager` n’existe pas :** `openstack role create manager`, puis `openstack implied role create manager --implied-role member`. Les règles de Keystone 2024.1 reconnaissent le rôle par son nom.

### Le partenaire, par symétrie

```bash
openstack role add --user bruno --user-domain partenaire --domain partenaire manager
os_as bruno partenaire domaine partenaire -- project create partenaire-app --domain partenaire
os_as bruno partenaire domaine partenaire -- user list --domain acme
```

✅ **Attendu :** le Bruno du partenaire gère son domaine. ⛔ **Refus attendu** sur la liste des comptes d’Acme. Deux gestionnaires homonymes, deux périmètres étanches.

---

## 2.5 Répartir l’administration : global ou délégué

Exercice en groupe : pour chaque tâche, les apprenants indiquent qui la fait, puis le vérifient sur le lab quand c’est possible.

| Tâche | Plateforme (admin) | Gestionnaire de domaine (manager) | Vérification |
|---|---|---|---|
| Créer le domaine `acme` | ✅ | ⛔ | § 2.4 |
| Créer un utilisateur dans `acme` | ✅ | ✅ | § 2.4 |
| Créer un projet dans `acme` | ✅ | ✅ | § 2.4 |
| Donner `member` sur un projet d’Acme | ✅ | ✅ | § 2.4 |
| Donner `admin` | ✅ | ⛔ | § 2.4 |
| Fixer les quotas d’un projet d’Acme | ✅ | ⛔ | TP 3 |
| Créer un réseau partagé, un flavor public | ✅ | ⛔ | TP 4 |
| Ajouter un hyperviseur, modifier le catalogue | ✅ | ⛔ | — |
| Brancher l’annuaire d’Acme (LDAP, fédération) | ✅ | ⛔ | module 11 |

---

## 2.6 Cycle de vie d’un domaine

### Simulation : désactiver un domaine

```bash
openstack domain set --disable partenaire
os_as bruno partenaire domaine partenaire -- token issue
openstack domain set --enable partenaire
os_as bruno partenaire domaine partenaire -- token issue -c expires
```

⛔ **Refus attendu (401)** pendant que le domaine est désactivé : aucun de ses comptes ne peut s’authentifier. ✅ **Attendu** après réactivation : le jeton est de nouveau délivré.

### Simulation : supprimer un domaine (sur un domaine jetable)

```bash
openstack domain create lab-ephemere
openstack user create jetable --domain lab-ephemere --password "$LAB_PW"
openstack project create jetable --domain lab-ephemere
openstack domain delete lab-ephemere
```

⛔ **Refus attendu :** un domaine actif ne se supprime pas (`403` ou `400` « Cannot delete a domain that is enabled »).

```bash
openstack domain set --disable lab-ephemere
openstack domain delete lab-ephemere
openstack user list --long | grep -c jetable
```

✅ **Attendu :** suppression réussie ; `0` : **les utilisateurs, groupes et projets du domaine disparaissent avec lui**.

⚠ **Piège :** les ressources (VM, volumes, réseaux) des projets supprimés ne sont **pas** supprimées : elles deviennent orphelines. Avant de supprimer un domaine, vider ses projets.

---

## 2.7 Les domaines dans Horizon

Depuis Windows, ouvre `http://cloud.lab.test/openstack-horizon` et connecte-toi avec :

| Champ | Valeur |
|---|---|
| Domaine | `acme` |
| Utilisateur | `bruno` |
| Mot de passe | `LAB_PW` |

❓ **Question :** Bruno voit-il le menu **Identité** ? Quels domaines voit-il ? *Attendu : il voit le menu Identité restreint à son domaine (selon la version d’Horizon et les politiques). Comparer avec une connexion de `demo` (domaine `users`).*

---

## État à la fin du TP

| Objet | État |
|---|---|
| Domaines | `acme`, `partenaire` |
| Projets | `acme-si`, `acme-dev` (enfant d’`acme-si`) ; `partenaire-app` |
| Utilisateurs | `bruno`, `chloe`, `emma`, `david` dans `acme` ; `bruno` dans `partenaire` |
| Affectations | `bruno@acme` manager d’`acme` ; `chloe` member d’`acme-dev` ; `bruno@partenaire` manager de `partenaire` |
| Rôle propre | `acme-facturation` (implique `reader`) |

Conserve cet état pour le TP 3. Pour tout effacer : `bash ~/labs-iam/nettoyage.sh`.

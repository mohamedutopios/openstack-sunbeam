# TP 3 : projets, hiérarchie et quotas

> Statut : **pas encore rejoué sur le lab.** Relu et mis en cohérence avec le TP 1, rejoué le 17 octobre 2026 : sur ce lab, Keystone réserve la création de comptes, de projets et d’affectations au rôle `admin` (TP 1, § 1.6.2). Les résultats marqués « à confirmer » en sont déduits. Voir les conventions dans [README.md](README.md).

**Module du support :** 10, projets, hiérarchie et quotas.
**Durée :** 2 h 30.
**Objectifs :** constater l’étanchéité d’un projet, construire un arbre de projets, utiliser les rôles hérités, fixer des quotas et les voir bloquer une demande, puis choisir un modèle d’organisation pour un SI cloisonné.

**Prérequis :** état de fin du TP 2 (domaine `acme`, projets `acme-si` et `acme-dev`, comptes `bruno`, `chloe`, `emma`, `david`, `chloe` membre d’`acme-dev`). Sur ce lab, les comptes et `acme-dev` ont été créés par la plateforme (TP 2, § 2.4). Toutes les actions d’administration de ce TP (projets, affectations, quotas) se font donc avec le compte `admin`. Session préparée :

```bash
source ~/labs-iam/lab-env.sh
```

---

## 3.1 Le projet, conteneur étanche

### Scénario : Chloé travaille dans `acme-dev`

```bash
os_as chloe acme projet acme-dev acme -- network create acme-dev-net
os_as chloe acme projet acme-dev acme -- security group create acme-dev-web
os_as chloe acme projet acme-dev acme -- network list
```

✅ **Attendu :** le réseau `acme-dev-net` et le groupe `acme-dev-web` sont créés dans `acme-dev`. La liste montre aussi `external-network` (réseau externe partagé par la plateforme), mais pas `demo-network`.

Emma n’a encore aucun rôle sur `acme-dev` :

```bash
os_as emma acme projet acme-dev acme -- network list
```

⛔ **Refus attendu (401) :** Emma ne peut même pas obtenir de jeton sur un projet où elle n’a aucun rôle.

❓ **Question :** qu’est-ce qui est étanche entre deux projets, et qu’est-ce qui est partagé ? *Réponse : étanches : VM, volumes, réseaux privés, groupes de sécurité, clés, quotas. Partagés : images publiques, flavors publics, réseaux marqués partagés ou externes, et l’infrastructure physique (hyperviseurs).*

### Étiquettes et propriétés

```bash
openstack project set acme-dev --domain acme --tag env:dev --tag app:portail
openstack project show acme-dev --domain acme -c tags
openstack project list --domain acme --tags env:dev
```

✅ **Attendu :** les étiquettes sont stockées sur le projet ; le filtre `--tags` retrouve les projets d’un environnement. Usage typique : refacturation, inventaire, automatisation.

---

## 3.2 La hiérarchie de projets

### Scénario : construire l’arbre d’Acme

```bash
openstack project create acme-prod --domain acme --parent acme-si --tag env:prod
openstack project create acme-recette --domain acme --parent acme-si --tag env:recette
openstack project show acme-si --domain acme --children -c name -c subtree
openstack project show acme-prod --domain acme --parents -c name -c parent_id -c parents
openstack project list --domain acme --long
```

✅ **Attendu :**

```text
acme (domaine)
└── acme-si
    ├── acme-dev
    ├── acme-prod
    └── acme-recette
```

`openstack project list --long` affiche la colonne `Parent ID` : `acme-si` a pour parent le **domaine** `acme` (un domaine est la racine de son arbre).

### Les règles de l’arbre

```bash
openstack project create lab-hors-domaine --domain partenaire --parent acme-si
openstack project delete acme-si --domain acme
```

⛔ **Refus attendu** pour les deux :

| Tentative | Règle |
|---|---|
| Parent dans un autre domaine | Un arbre ne traverse pas les domaines |
| Supprimer un parent qui a des enfants | Il faut supprimer (ou déplacer) les enfants d’abord |

❓ **Question :** quelle est la profondeur maximale d’un arbre ? *Réponse : réglage `max_project_tree_depth` de Keystone (5 par défaut). Vérifier : `sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- grep max_project_tree_depth /etc/keystone/keystone.conf`.*

---

## 3.3 Les rôles hérités

### Scénario : David audite tout Acme

David doit **lire** tous les projets d’Acme, y compris ceux qui seront créés plus tard, sans rien pouvoir modifier.

```bash
openstack role add --user david --user-domain acme --project acme-si --project-domain acme --inherited reader
openstack role assignment list --user david --user-domain acme --names
openstack role assignment list --user david --user-domain acme --names --effective
```

✅ **Attendu :**

- liste simple : une affectation `reader` sur `acme-si`, marquée `Inherited = True` ;
- liste effective : `reader` sur `acme-dev`, `acme-prod` et `acme-recette`, **mais pas sur `acme-si` lui-même**. Un rôle hérité s’applique au sous-arbre, pas au projet qui le porte.

```bash
os_as david acme projet acme-dev acme -- network list
os_as david acme projet acme-dev acme -- network create acme-intrus
os_as david acme projet acme-si acme -- network list
```

✅ **Attendu :** la lecture réussit sur `acme-dev`. ⛔ **Refus attendu (403)** sur la création : sur ce lab, la règle de Neutron est `create_network: (rule:admin_only) or (role:member and project_id:%(project_id)s)` (TP 1, § 1.6.2) ; elle exige `member`, et `reader` n’implique pas `member`. ⛔ **Refus attendu (401)** sur `acme-si` : aucun rôle effectif sur ce projet.

### Simulation : un nouveau projet hérite automatiquement

```bash
openstack project create acme-dev-api --domain acme --parent acme-dev
openstack role assignment list --user david --user-domain acme --names --effective | grep acme-dev-api
os_as david acme projet acme-dev-api acme -- server list
```

Le projet est créé par la plateforme : sur ce lab, le gestionnaire de domaine ne peut pas créer de projet (TP 2, § 2.4).

✅ **Attendu :** David a déjà `reader` sur `acme-dev-api`, sans qu’on l’ait ajouté : l’héritage s’évalue à chaque jeton.

### Scénario : Emma, exploitante

Emma travaille en production et doit pouvoir **lire** tout le reste d’Acme.

```bash
openstack role add --user emma --user-domain acme --project acme-prod --project-domain acme member
openstack role add --user emma --user-domain acme --domain acme --inherited reader
openstack role assignment list --user emma --user-domain acme --names --effective
```

✅ **Attendu :** `member` sur `acme-prod`, `reader` sur **tous** les projets du domaine `acme` (y compris `acme-si`, cette fois : l’héritage depuis le **domaine** couvre tous ses projets).

| Affectation | Où s’applique-t-elle |
|---|---|
| `--project P` | P seulement |
| `--project P --inherited` | les descendants de P, pas P |
| `--domain D` | le domaine D lui-même (jeton de domaine), aucun projet |
| `--domain D --inherited` | tous les projets du domaine D |

### Ce que la hiérarchie ne fait pas

❓ **Question :** Chloé a créé `acme-dev-net` dans `acme-dev`. Peut-elle l’utiliser depuis `acme-dev-api`, projet enfant ?

```bash
openstack role add --user chloe --user-domain acme --project acme-dev-api --project-domain acme member
os_as chloe acme projet acme-dev-api acme -- network list
```

✅ **Attendu :** `acme-dev-net` **n’apparaît pas**. La hiérarchie transmet des **rôles**, jamais des **ressources** : un réseau, une VM, un volume restent dans leur projet. Pour partager, il faut un partage explicite (RBAC de Neutron, module 13).

---

## 3.4 Les quotas

### 3.4.1 Lire

```bash
openstack quota show --default
openstack quota show acme-dev --domain acme
openstack quota show --usage acme-dev --domain acme
sudo k8s kubectl -n openstack exec nova-0 -c nova-api -- grep -A3 "^\[quota\]" /etc/nova/nova.conf
openstack registered limit list
```

✅ **Attendu :** les quotas d’`acme-dev` sont ceux par défaut (par exemple 10 instances, 20 vCPU, 50 Go de RAM, 100 réseaux… selon la configuration) ; la consommation montre le réseau et le groupe de sécurité de Chloé. `registered limit list` vide : le lab utilise les **quotas classiques** de chaque service, pas les limites unifiées.

### 3.4.2 Fixer

```bash
openstack quota set acme-dev --domain acme --instances 2 --cores 2 --ram 2048
openstack quota set acme-dev --domain acme --networks 1 --subnets 2 --ports 10 --routers 1 --floating-ips 1 --secgroups 3
openstack quota set acme-dev --domain acme --volumes 2 --gigabytes 20
openstack quota show acme-dev --domain acme
```

✅ **Attendu :** les nouvelles valeurs, service par service (Nova, Neutron, Cinder).

⚠ **Piège :** `openstack quota set` sans `--domain` cherche le projet par nom dans tous les domaines : ambigu dès que deux domaines ont un projet du même nom.

### 3.4.3 Simulation : le quota bloque

**Réseaux :** Chloé a déjà `acme-dev-net`, le quota est à 1.

```bash
os_as chloe acme projet acme-dev acme -- network create acme-dev-net2
```

⛔ **Refus attendu (409) :** `Quota exceeded for resources: ['network']`.

**Instances :** on bloque toute instance en recette, sans rien consommer.

```bash
openstack quota set acme-recette --domain acme --instances 0
openstack role add --user emma --user-domain acme --project acme-recette --project-domain acme member
os_as emma acme projet acme-recette acme -- server create lab-quota --flavor m1.tiny --image ubuntu --nic none
```

⛔ **Refus attendu (403) :** `Quota exceeded for instances: Requested 1, but already used 0 of 0 instances`. Le refus a lieu **avant** toute réservation : rien n’est créé.

```bash
os_as emma acme projet acme-recette acme -- server list
```

**Le gestionnaire ne fixe pas les quotas :**

```bash
os_as bruno acme domaine acme -- quota set acme-dev --domain acme --instances 50
```

⛔ **Refus attendu (403) :** les quotas sont décidés par la plateforme, qui protège la capacité commune. Sur ce lab, Bruno n’est en pratique qu’un lecteur de son domaine (TP 2, § 2.4) : le refus est certain, quelle que soit la règle exacte de chaque service.

### 3.4.4 Les quotas par défaut

❓ **Question :** quel est le réglage le plus important pour protéger la plateforme ? *Réponse : les quotas **par défaut**, car ils s’appliquent à tout nouveau projet tant qu’on ne les a pas fixés explicitement.*

```bash
openstack quota show --default -c instances -c cores -c ram
```

⚠ **Piège :** `openstack quota set --class default …` change le défaut **de tous les projets** du cloud qui n’ont pas de valeur propre. À faire en connaissance de cause ; ne pas le lancer en TP sans revenir ensuite à la valeur initiale.

---

## 3.5 Organiser un SI cloisonné

### Trois modèles

**Modèle A, par environnement** (celui d’Acme) :

```text
acme
└── acme-si
    ├── acme-prod
    ├── acme-recette
    └── acme-dev
```

**Modèle B, par application puis environnement :**

```text
acme
├── acme-portail
│   ├── acme-portail-prod
│   └── acme-portail-dev
└── acme-paie
    ├── acme-paie-prod
    └── acme-paie-dev
```

**Modèle C, par entité (direction, filiale), un domaine chacune :**

```text
acme-rh   (domaine)        acme-finance (domaine)
└── rh-prod, rh-dev        └── fin-prod, fin-dev
```

| Critère | A : environnement | B : application | C : domaine par entité |
|---|---|---|---|
| Séparation production / hors production | Forte | Forte | Forte |
| Séparation entre applications | Faible (toutes dans `acme-prod`) | Forte | Forte |
| Nombre de projets | Faible | Élevé | Élevé |
| Auditeur transversal | Un rôle hérité sur `acme-si` | Un rôle hérité sur le domaine | Un rôle par domaine |
| Délégation | Un gestionnaire pour tout Acme | Un gestionnaire par application possible (rôles sur sous-arbre) | Un gestionnaire par entité |
| Source d’identité distincte | Non | Non | Oui, possible par domaine |

### Exercice de conception

Énoncé à donner aux apprenants : « Acme héberge le portail client (exposé sur Internet, données personnelles) et la paie (données sensibles, accès restreint à 3 personnes). L’équipe de développement est commune. L’auditeur de sécurité doit tout lire. La production ne doit jamais être modifiable par les développeurs. »

1. Choisir un modèle et dessiner l’arbre.
2. Écrire la liste des affectations (qui, quel rôle, où, hérité ou non).
3. Fixer les quotas de chaque projet.
4. L’implémenter sur le lab dans le domaine `acme`, avec des noms préfixés `acme-`.

Corrigé indicatif : modèle B ; `acme-portail` et `acme-paie` sous le domaine ; développeurs `member` sur `acme-portail-dev` et `acme-paie-dev` ; exploitants `member` sur les deux `-prod` ; auditeur `reader` hérité sur le domaine ; quotas faibles en `-dev`, dimensionnés en `-prod`. La paie pourrait justifier un domaine propre (modèle C) si ses comptes viennent d’une autre source d’identité.

---

## État à la fin du TP

| Objet | État |
|---|---|
| Arbre | `acme-si` → `acme-dev` (→ `acme-dev-api`), `acme-prod`, `acme-recette` |
| David | `reader` hérité sur `acme-si` |
| Emma | `member` sur `acme-prod` et `acme-recette`, `reader` hérité sur le domaine `acme` |
| Chloé | `member` sur `acme-dev` et `acme-dev-api` ; réseau `acme-dev-net`, groupe `acme-dev-web` |
| Quotas | `acme-dev` réduits ; `acme-recette` : 0 instance |

Conserve cet état pour le TP 4. Avant de nettoyer, supprime le réseau et le groupe de Chloé :

```bash
os_as chloe acme projet acme-dev acme -- network delete acme-dev-net
os_as chloe acme projet acme-dev acme -- security group delete acme-dev-web
```

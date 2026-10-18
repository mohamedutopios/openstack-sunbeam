# TP 4 : rôles et moindre privilège

> Statut : **pas encore rejoué sur le lab.** Relu et mis en cohérence avec le TP 1, rejoué le 17 octobre 2026 : sur ce lab, Keystone réserve la création de comptes, de projets et d’affectations au rôle `admin` (TP 1, § 1.6.2). Les résultats marqués « à confirmer » en sont déduits. Voir les conventions dans [README.md](README.md).

**Module du support :** 12, rôles et moindre privilège.
**Durée :** 2 h 30.
**Objectifs :** mesurer ce que permet chaque rôle standard, lire et modifier la hiérarchie des rôles impliqués, utiliser les trois portées de Secure RBAC, remplacer un mot de passe par un identifiant d’application limité, et auditer qui a quels droits.

**Prérequis :** état de fin du TP 3. Relevé de Secure RBAC du TP 1 (§ 1.6.1) sous les yeux : la plupart des résultats de ce TP en dépendent. Session préparée :

```bash
source ~/labs-iam/lab-env.sh
```

---

## 4.1 Les rôles standards et leur hiérarchie

```bash
openstack role list
openstack implied role list
```

✅ **Attendu** (observé au TP 1) : une chaîne `admin` → `manager` → `member` → `reader` (le rôle de gauche implique celui de droite), et un rôle `service`.

⚠ Sur ce lab, les règles de Keystone ne donnent **aucun droit propre** à `manager` (TP 1, § 1.6.2 ; TP 2, § 2.4) : il vaut `member` sur un projet, et `reader` sur un domaine. La colonne « Secure RBAC » du tableau ci-dessous décrit le modèle ; la matrice du § 4.2 montre ce que fait réellement le lab.

| Rôle | Pour qui | Ce qu’il permet (Secure RBAC) |
|---|---|---|
| `reader` | auditeur, supervision | lire, sans rien modifier |
| `member` | utilisateur d’un projet | créer et gérer les ressources du projet |
| `manager` | responsable d’un projet ou d’un domaine | `member`, plus gérer certaines propriétés et déléguer `reader`, `member`, `manager` |
| `admin` | équipe plateforme | tout, y compris hors de la cible si les portées ne sont pas appliquées |
| `service` | comptes des services OpenStack | appels entre services, jamais pour une personne |

---

## 4.2 La matrice rôle × opération

### Préparer trois profils sur le même projet

Sur `acme-prod` : David a déjà `reader` (hérité, TP 3), Emma a `member`. On donne `manager` du projet à Bruno.

```bash
openstack role add --user bruno --user-domain acme --project acme-prod --project-domain acme manager
openstack role assignment list --project acme-prod --project-domain acme --names --effective
```

### Lancer la matrice

Le script ci-dessous tente une série d’opérations avec chaque profil, et affiche ✅ (réussi) ou ⛔ (refusé). Il nettoie ce qu’il a créé.

```bash
cat > ~/labs-iam/matrice.sh <<'EOF'
#!/usr/bin/env bash
# Matrice role x operation sur acme-prod. Usage : bash ~/labs-iam/matrice.sh
source ~/labs-iam/lab-env.sh >/dev/null
try() {  # try LIBELLE USER -- commande openstack
    local label=$1 user=$2; shift 3
    if os_as "$user" acme projet acme-prod acme -- "$@" >/dev/null 2>&1; then printf "  %-8s ✅" "$user"
    else printf "  %-8s ⛔" "$user"; fi
}
row() {  # row LIBELLE commande... (exécutée par david, emma, bruno)
    local label=$1; shift
    printf "%-44s" "$label"
    for u in david emma bruno; do try "$label" "$u" -- "$@"; done
    echo
}
cleanup() {  # supprime par identifiant : plusieurs profils creent un objet du meme nom
    openstack network list --name lab-mx-net -f value -c ID | xargs -r -n1 openstack network delete >/dev/null 2>&1
    openstack network list --name lab-mx-shared -f value -c ID | xargs -r -n1 openstack network delete >/dev/null 2>&1
    openstack security group list --all-projects -f value -c ID -c Name | awk '$2=="lab-mx-sg" {print $1}' \
        | xargs -r -n1 openstack security group delete >/dev/null 2>&1
    sleep 10  # un volume en cours de creation ne se supprime pas
    openstack volume list --all-projects --name lab-mx-vol -f value -c ID | xargs -r -n1 openstack volume delete >/dev/null 2>&1
    openstack image list --name lab-mx-img -f value -c ID | xargs -r -n1 openstack image delete >/dev/null 2>&1
}
printf "%-44s  %-11s %-11s %-11s\n" "Operation" "reader" "member" "manager"
row "Lister les instances"                server list
row "Lister les reseaux"                  network list
row "Creer un reseau prive"               network create lab-mx-net
row "Creer un groupe de securite"         security group create lab-mx-sg
row "Creer un volume de 1 Go"             volume create --size 1 lab-mx-vol
row "Creer un reseau partage"             network create --share lab-mx-shared
row "Lister les affectations du projet"   role assignment list --project acme-prod --project-domain acme
row "Donner reader a chloe sur le projet" role add --user chloe --user-domain acme --project acme-prod --project-domain acme reader
openstack role remove --user chloe --user-domain acme --project acme-prod --project-domain acme reader >/dev/null 2>&1
row "Modifier les quotas du projet"       quota set acme-prod --domain acme --instances 9
row "Lister les hyperviseurs"             hypervisor list
row "Creer une image publique"            image create --public --disk-format raw --container-format bare lab-mx-img
echo "Nettoyage des objets de test..."
cleanup
EOF
bash ~/labs-iam/matrice.sh
```

⚖ **dépend des politiques.** Deux attendus : celui du modèle Secure RBAC complet, et celui de **ce lab**, déduit des règles lues au TP 1 (§ 1.6.2). « À relever » : la règle n’a pas été lue ; c’est la matrice qui répond.

| Opération | Secure RBAC : reader | member | manager | **Ce lab** : reader | member | manager |
|---|---|---|---|---|---|---|
| Lister les instances, les réseaux | ✅ | ✅ | ✅ | à relever | ✅ | ✅ |
| Créer un réseau privé | ⛔ | ✅ | ✅ | ⛔ | ✅ | ✅ |
| Créer un groupe de sécurité, un volume | ⛔ | ✅ | ✅ | à relever | ✅ | ✅ |
| Créer un réseau partagé | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| Lister les affectations du projet | ⛔ | ⛔ | ✅ | à relever | à relever | à relever |
| Donner `reader` à un autre compte sur le projet | ⛔ | ⛔ | ✅ | ⛔ | ⛔ | ⛔ |
| Modifier les quotas | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |
| Lister les hyperviseurs, créer une image publique | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ | ⛔ |

Justifications pour ce lab : `create_network` exige `member` du projet ; `create_network:shared` et `os-hypervisors:list` exigent l’administrateur ; `identity:create_grant` exige `admin` (d’où le refus pour `manager`, contrairement au modèle).

❓ **Questions :**

1. Les lignes où **personne** ne réussit : qui peut les faire, et avec quelle portée ? *Réponse : l’équipe plateforme, `admin` (portée système avec Secure RBAC complet).*
2. Si `reader` réussit à créer un réseau sur ton lab, que conclure ? *Réponse : le service concerné applique encore les anciennes règles (`enforce_new_defaults = False`), où tout rôle sur le projet suffit.*

Faire tourner la matrice **avant et après** une modification de politique est le meilleur moyen d’en vérifier l’effet.

---

## 4.3 Les rôles impliqués

### Scénario : un profil « auditeur sécurité »

L’équipe sécurité veut un rôle nommé, qui donne exactement les droits de lecture.

```bash
openstack role create lab-auditeur-securite
openstack implied role create lab-auditeur-securite --implied-role reader
openstack role add --user david --user-domain acme --project acme-dev --project-domain acme lab-auditeur-securite
openstack role assignment list --user david --user-domain acme --project acme-dev --project-domain acme --names --effective
os_as david acme projet acme-dev acme -- token issue -c expires
```

✅ **Attendu :** la liste effective montre `lab-auditeur-securite` **et** `reader` ; dans le jeton, les deux rôles sont présents, et les services n’appliquent que `reader`, le seul qu’ils connaissent.

⚠ **Piège :** un rôle personnalisé **sans** implication ne donne aucun droit ; avec une implication vers `admin`, il en donne trop. Avant de créer un rôle, vérifier qu’un rôle standard ne suffit pas.

```bash
openstack role remove --user david --user-domain acme --project acme-dev --project-domain acme lab-auditeur-securite
openstack implied role delete lab-auditeur-securite --implied-role reader
openstack role delete lab-auditeur-securite
```

❓ **Question :** que se passerait-il si on ajoutait `member` → `admin` comme implication ? *Réponse : tout `member` deviendrait `admin` : l’implication se lit comme une inclusion des droits. C’est la modification la plus dangereuse de Keystone.*

---

## 4.4 Le contrôle d’accès par périmètre (Secure RBAC)

### Les personas

Un **persona** est un rôle donné sur une portée.

| Persona | Affectation | Exemple dans le fil rouge |
|---|---|---|
| Administrateur système | `admin` sur le système | équipe plateforme |
| Lecteur système | `reader` sur le système | supervision de la plateforme |
| Gestionnaire de domaine | `manager` sur un domaine | Bruno sur `acme` |
| Lecteur de domaine | `reader` sur un domaine | — |
| Membre de projet | `member` sur un projet | Chloé sur `acme-dev` |
| Lecteur de projet | `reader` sur un projet | David (hérité) |

### Scénario : un compte de supervision de la plateforme

L’équipe de supervision doit tout **voir** sur la plateforme, sans pouvoir rien changer.

```bash
openstack user create lab-supervision --domain admin_domain --password "$LAB_PW" --description "Supervision plateforme"
openstack role add --user lab-supervision --user-domain admin_domain --system all reader
openstack role assignment list --system all --names
```

Avec un jeton **système** :

```bash
os_as lab-supervision admin_domain systeme -- token issue -c system -c expires
os_as lab-supervision admin_domain systeme -- project list
os_as lab-supervision admin_domain systeme -- user list --domain acme
os_as lab-supervision admin_domain systeme -- hypervisor list
os_as lab-supervision admin_domain systeme -- project create lab-interdit --domain acme
```

✅ **Attendu sur ce lab** (déduit des règles du TP 1 ; à confirmer) :

| Commande | Résultat | Règle |
|---|---|---|
| `token issue` (portée système) | ✅ jeton avec `system` | Le compte a un rôle sur le système : il peut en obtenir la portée |
| `project list`, `user list --domain acme` | ✅ | `list_projects` et `list_users` acceptent `reader` sur le système |
| `hypervisor list` | ⛔ `403` | `os-hypervisors:list` = `rule:context_is_admin` : Nova n’a pas de règle de lecture système sur ce lab |
| `project create lab-interdit` | ⛔ `403` | `create_project` exige `admin` |

Le compte de supervision lit bien l’**identité** de toute la plateforme, mais pas l’infrastructure de Nova : sur ce lab, Keystone connaît le lecteur système, Nova non.

Le même compte n’a **aucun** rôle de projet : il ne peut pas obtenir de jeton de projet.

```bash
os_as lab-supervision admin_domain projet acme-prod acme -- server list
```

⛔ **Refus attendu (401).** ❓ **Question :** pourquoi ne peut-il pas lister les VM d’`acme-prod`, alors qu’il a un rôle sur le système ? *Réponse : lister les instances d’un projet est une opération de **projet**, et il n’a aucun rôle sur ce projet. Pour une vue globale, l’API de Nova propose `server list --all-projects`, prévue en lecture système par Secure RBAC.*

```bash
os_as lab-supervision admin_domain systeme -- server list --all-projects
```

✅ **Attendu :** à relever. Avec Secure RBAC complet, la liste réussit ; sur ce lab, où Nova ne reconnaît pas le lecteur système pour les hyperviseurs, un refus `403` est probable. Faire relier le résultat à la règle : `oslopolicy-policy-generator --namespace nova | grep "servers:index:get_all_tenants"`.

### Simulation : la même opération sous trois portées

`admin` dispose d’un jeton de projet (`admin-openrc`) et d’un rôle sur son domaine, mais **d’aucun rôle système** (TP 1, § 1.4.3).

```bash
openstack hypervisor list
env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME openstack --os-system-scope all hypervisor list
env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME openstack --os-domain-name admin_domain hypervisor list
```

| Portée | Secure RBAC complet (`enforce_scope = True`) | **Ce lab** (déduit ; à confirmer) |
|---|---|---|
| Projet `admin` | ⛔ refusé : la liste des hyperviseurs est une opération système | ✅ réussit : `rule:context_is_admin` ne regarde que le rôle `admin` |
| Système | ✅ | ⛔ `401` : `admin` n’a aucun rôle système, pas de jeton |
| Domaine `admin_domain` | ⛔ | ✅ probable, pour la même raison que la première ligne |

Cette expérience montre aux apprenants **pourquoi** la portée existe : avec Secure RBAC, le même rôle `admin` n’a pas le même effet selon ce qu’il cible ; sur ce lab, il a le même effet partout.

---

## 4.5 Les comptes de service

### 4.5.1 Les comptes des services OpenStack

```bash
openstack user list --domain service_domain
openstack role assignment list --project services --project-domain service_domain --names
```

❓ **Question :** pourquoi chaque service a-t-il son propre compte, plutôt qu’un compte commun ? *Réponse : traçabilité (on sait quel service a agi), révocation ciblée, et moindre privilège possible service par service.*

### 4.5.2 Identifiant d’application : le scénario de la sauvegarde

Emma veut qu’un script de sauvegarde, exécuté chaque nuit, **liste** les instances et les volumes d’`acme-prod`. Le script ne doit ni connaître son mot de passe, ni pouvoir créer ou supprimer quoi que ce soit.

```bash
os_as emma acme projet acme-prod acme -- application credential create lab-sauvegarde \
  --role reader --expiration "$(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%S)" \
  --description "Script de sauvegarde nocturne, lecture seule" -f json > /tmp/appcred.json
python3 -c "import json; d=json.load(open('/tmp/appcred.json')); print(d['id'], d['roles'], d['expires_at'], d['unrestricted'])"
APPCRED_ID=$(python3 -c "import json; print(json.load(open('/tmp/appcred.json'))['id'])")
APPCRED_SECRET=$(python3 -c "import json; print(json.load(open('/tmp/appcred.json'))['secret'])")
```

✅ **Attendu :** un identifiant, le rôle `reader` seul, une date d’expiration, `unrestricted = False`. ⚠ **Piège :** le **secret** n’est affiché qu’une fois, à la création ; Keystone n’en garde qu’une empreinte.

⚠ **Piège : un identifiant d’application ne reçoit que les rôles nommés à sa création.** Les rôles **impliqués** ne suivent pas. Constaté sur le lab le 18 octobre 2026 (`6-securite/04-rotation-secrets.md`, § 4.7) : bien que `member` implique `reader` (`openstack implied role list`), un identifiant créé avec `--role member` seul ne voyait pas les réseaux du projet ; avec `--role member --role reader`, il les voyait. Ici, on nomme `reader` explicitement : le script lit sans problème. Pour un script qui doit aussi créer, nommer **les deux** rôles.

Le script utilise l’identifiant d’application :

```bash
os_appcred "$APPCRED_ID" "$APPCRED_SECRET" -- server list
os_appcred "$APPCRED_ID" "$APPCRED_SECRET" -- volume list
os_appcred "$APPCRED_ID" "$APPCRED_SECRET" -- network create lab-pirate
os_appcred "$APPCRED_ID" "$APPCRED_SECRET" -- application credential create lab-rebond
```

✅ **Attendu :** les deux lectures réussissent. ⛔ **Refus attendu** sur la création de réseau (`reader`, ⚖ **dépend de Secure RBAC**) et sur la création d’un autre identifiant d’application (`unrestricted = False` empêche de rebondir).

### 4.5.3 Restreindre aussi les appels : règles d’accès

```bash
os_as emma acme projet acme-prod acme -- application credential create lab-sauvegarde-strict \
  --role reader --access-rules '[{"service": "compute", "method": "GET", "path": "/v2.1/servers/detail"}]' -f json > /tmp/appcred2.json
ID2=$(python3 -c "import json; print(json.load(open('/tmp/appcred2.json'))['id'])")
SEC2=$(python3 -c "import json; print(json.load(open('/tmp/appcred2.json'))['secret'])")
os_appcred "$ID2" "$SEC2" -- server list
os_appcred "$ID2" "$SEC2" -- volume list
```

✅ **Attendu :** `server list` réussit. ⛔ **Refus attendu** sur `volume list` : l’appel n’est pas dans la liste des règles d’accès.

⚠ **Piège :** les règles d’accès portent sur l’URL exacte appelée par le client. `openstack server list` appelle `GET /v2.1/servers/detail` ; une règle sur `/v2.1/servers` seul ne suffirait pas. Pour trouver le bon chemin : `openstack --debug server list 2>&1 | grep "REQ: curl"`.

### 4.5.4 L’identifiant suit les droits de son propriétaire

```bash
openstack role remove --user emma --user-domain acme --project acme-prod --project-domain acme member
openstack role remove --user emma --user-domain acme --domain acme --inherited reader
os_appcred "$APPCRED_ID" "$APPCRED_SECRET" -- server list
openstack role add --user emma --user-domain acme --project acme-prod --project-domain acme member
openstack role add --user emma --user-domain acme --domain acme --inherited reader
```

⛔ **Refus attendu (401)** pendant qu’Emma n’a plus de rôle sur `acme-prod` : un identifiant d’application ne peut jamais dépasser les droits de son propriétaire, et les perd avec lui.

### 4.5.5 Révoquer

```bash
os_as emma acme projet acme-prod acme -- application credential list
os_as emma acme projet acme-prod acme -- application credential delete lab-sauvegarde
os_as emma acme projet acme-prod acme -- application credential delete lab-sauvegarde-strict
os_appcred "$APPCRED_ID" "$APPCRED_SECRET" -- server list
rm -f /tmp/appcred.json /tmp/appcred2.json
```

⛔ **Refus attendu (401)** après la suppression.

| Besoin | Compte dédié (utilisateur) | Identifiant d’application |
|---|---|---|
| Un script agit pour une personne | Non | ✅ |
| Un service tourne sans personne (ex. supervision) | ✅ | Possible, rattaché au compte d’une personne |
| Droits inférieurs à ceux du propriétaire | Par affectation | ✅ rôles choisis, règles d’accès |
| Expiration | Par la politique de mots de passe | ✅ date d’expiration |
| Révocation sans toucher au mot de passe | Non | ✅ suppression |

---

## 4.6 Auditer les droits

### Qui est administrateur ?

```bash
openstack role assignment list --role admin --names
openstack role assignment list --role admin --names --effective
```

✅ **Attendu :** `admin@admin_domain` (sur le projet `admin` et sur le domaine `admin_domain`), `_charm-keystone-admin@Default` (sur le système et le projet `admin` de `Default`), et les comptes de service sur `services` (TP 1, § 1.2) ; **aucun** compte d’`acme` ou de `partenaire`. ❓ **Question :** pourquoi regarder aussi la liste effective ? *Réponse : un rôle personnalisé qui implique `admin`, ou un groupe qui a `admin`, n’apparaît pas dans la liste simple.*

### Que peut faire un compte ?

```bash
openstack role assignment list --user emma --user-domain acme --names --effective
```

### Qui a accès à un projet ?

```bash
openstack role assignment list --project acme-prod --project-domain acme --names --effective
```

### Identifiants d’application d’un domaine

⚠ **Piège du client :** `openstack application credential list --user <compte>` échoue avec le client 7.5.0 du lab (`'function' object has no attribute 'list'`, constaté le 18 octobre 2026). On interroge donc directement l’API de Keystone, avec le jeton de l’administrateur :

```bash
T=$(openstack token issue -f value -c id)
K=$(openstack endpoint list --service identity --interface public -f value -c URL)
for u in $(openstack user list --domain acme -f value -c ID); do
  name=$(openstack user show "$u" -f value -c name)
  curl -s ${OS_CACERT:+--cacert "$OS_CACERT"} -H "X-Auth-Token: $T" "$K/users/$u/application_credentials" \
    | python3 -c 'import sys,json; [print(sys.argv[1], ":", a["name"], "expire", a["expires_at"]) for a in json.load(sys.stdin)["application_credentials"]]' "$name"
done
openstack token revoke "$T"
```

**La commande, mot par mot :** `T` est un jeton administrateur, `K` l’adresse de Keystone lue dans le catalogue ; pour chaque compte d’`acme`, l’API `GET /v3/users/<id>/application_credentials` liste ses identifiants, dont le programme Python affiche le nom et l’expiration ; `${OS_CACERT:+--cacert …}` n’ajoute la CA que si le cloud est en HTTPS ; le jeton est révoqué à la fin.

✅ **Attendu** (à confirmer) : une ligne par identifiant encore présent, par exemple `emma : lab-sauvegarde expire …` si le § 4.5.5 n’a pas été joué ; aucune ligne s’il l’a été.

### Exercice : la revue des droits

Donne aux apprenants la règle « pas d’`admin` hors plateforme, pas d’affectation directe d’utilisateur sur un projet de production, tout identifiant d’application expire sous 90 jours ». Ils écrivent les commandes qui vérifient chaque point, puis corrigent les écarts trouvés (l’affectation directe d’Emma sur `acme-prod` en est un : la remplacer par un groupe `acme-exploitation`).

```bash
openstack group create acme-exploitation --domain acme
openstack group add user --group-domain acme --user-domain acme acme-exploitation emma
openstack role add --group acme-exploitation --group-domain acme --project acme-prod --project-domain acme member
openstack role remove --user emma --user-domain acme --project acme-prod --project-domain acme member
openstack role assignment list --user emma --user-domain acme --names --effective
```

✅ **Attendu :** Emma garde `member` sur `acme-prod`, désormais **via le groupe**. Un départ ou une arrivée se gère en ajoutant ou retirant le compte du groupe.

---

## Nettoyage du TP

```bash
openstack role remove --user bruno --user-domain acme --project acme-prod --project-domain acme manager
openstack user delete lab-supervision --domain admin_domain
rm -f ~/labs-iam/matrice.sh
```

Le reste (domaines `acme`, `partenaire` et leur contenu) est supprimé par `bash ~/labs-iam/nettoyage.sh`.

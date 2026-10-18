# TP 1 : Keystone en profondeur

> Statut : **rejoué sur le lab les 17 et 18 octobre 2026** (Keystone 2024.1, client `openstack` 7.5.0). Les résultats ✅ ci-dessous sont ceux observés. Voir les conventions dans [README.md](README.md).

**Module du support :** 8, Keystone en profondeur.
**Durée :** 2 h 30.
**Objectifs :** à la fin du TP, l’apprenant sait inventorier les entités de Keystone, lire le catalogue, obtenir, inspecter, valider et révoquer un jeton à la main, identifier la source d’identité d’un compte, et lire une règle de politique pour expliquer un refus.

**Prérequis :** lab démarré et prêt (étape 10 du guide apprenant), session préparée :

```bash
source ~/labs-iam/lab-env.sh
K="sudo k8s kubectl -n openstack"
```

---

## 1.1 Où tourne Keystone sur Sunbeam

```bash
juju status -m openstack keystone
$K get pod keystone-0 -o wide
$K logs keystone-0 -c keystone --tail=5
```

✅ **Observé** (18 octobre 2026) : l’application `keystone` est `active` ; un pod `keystone-0` en `2/2` (deux conteneurs, `charm` et `keystone`) ; des lignes du serveur web Apache. Selon le moment, ce sont surtout des `"GET /healthcheck HTTP/1.1" 200` (le test de santé que Pebble fait toutes les quelques secondes) ou des `"POST /openstack-keystone/v3/auth/tokens HTTP/1.1" 201` (des demandes de jetons). Pour ne voir que les jetons : `$K logs keystone-0 -c keystone --since=10m | grep auth/tokens | tail -3`.

❓ **Question :** pourquoi `keystone-0` n’a-t-il pas de base de données dans le pod ? *Réponse : il utilise MySQL (pod `mysql-0`) via `keystone-mysql-router` ; Keystone est sans état local, hormis ses clés Fernet.*

---

## 1.2 Inventaire des entités

### Scénario : dresser la carte d’un cloud qu’on découvre

Tu arrives sur un cloud inconnu ; tu dois en décrire l’organisation en 10 minutes.

```bash
openstack domain list
openstack project list --long
openstack user list --long
openstack group list
openstack role list
openstack implied role list
openstack region list
openstack service list
```

Puis les **affectations**, qui relient les trois : acteur, rôle, cible.

```bash
openstack role assignment list --names
```

✅ **Observé sur le lab :**

| Entité | Valeurs |
|---|---|
| Domaines | `admin_domain`, `service_domain` (description « Created by Juju »), `users`, `Default` |
| Projets | `admin` (dans `admin_domain`), `services` (dans `service_domain`), `demo` (dans `users`), et un second `admin` dans `Default` : « Bootstrap project for initializing the cloud » |
| Utilisateurs | `admin` ; dans `service_domain`, un compte par service avec un suffixe aléatoire (`svc_nova-F8OChi`, `svc_neutron-Jbci5R`, `svc_glance-…`, `svc_placement-…`, `svc_cinder-…`), plus `horizon`, `cinder-volume`, `openstack-hypervisor` et un `svc_remote_…` ; `demo` dans `users` ; `_charm-keystone-admin` dans `Default` |
| Rôles | `admin`, `manager`, `member`, `reader`, `service` |
| Rôles impliqués | `admin` → `manager` → `member` → `reader` |
| Affectations | `admin@admin_domain` : `admin` et `member` sur le projet `admin`, `admin` sur le **domaine** `admin_domain` ; chaque compte de service : `admin` **et** `service` sur `services` ; `demo` : `member` sur `demo` ; `_charm-keystone-admin@Default` : `admin` sur le **système** (le seul) |

`_charm-keystone-admin` est le compte que le charm Keystone utilise lui-même pour administrer Keystone : c’est le seul à avoir un rôle sur le système.

### Exercice

Les apprenants complètent ce tableau à partir de leurs sorties :

| Question | Commande | Leur réponse |
|---|---|---|
| Combien de domaines, et lequel héberge les comptes des services ? | `openstack domain list` | |
| Quels utilisateurs ont un rôle sur le projet `services` ? | `openstack role assignment list --project services --project-domain service_domain --names` | |
| Quel rôle `admin` implique-t-il ? | `openstack implied role list` | |
| L’utilisateur `demo` peut-il voir le projet `admin` ? | `openstack role assignment list --user demo --user-domain users --names` | |

⚠ **Piège :** `openstack user list` sans `--domain` ne liste **pas** tous les comptes. Sur le lab, avec le jeton de l’administrateur (projet `admin` du domaine `admin_domain`), il ne renvoie qu’**un** compte, `admin` : ceux du domaine du jeton. Pour inventorier un cloud, interroger chaque domaine : `for d in $(openstack domain list -f value -c Name); do openstack user list --domain "$d"; done`. Deux utilisateurs peuvent aussi porter le même nom dans deux domaines différents (TP 2) : un nom seul ne désigne pas un compte.

### Lire une affectation effective

```bash
openstack role assignment list --user demo --user-domain users --names --effective
```

`--effective` développe les **rôles impliqués** et les **groupes** : il répond à la question « que peut réellement faire ce compte ? », alors que la liste simple ne montre que les affectations directes.

❓ **Question :** pourquoi la liste effective de `demo` contient-elle `reader` alors qu’on ne lui a affecté que `member` ? *Réponse : `member` implique `reader` (`openstack implied role list`).*

---

## 1.3 Le catalogue de services

### Scénario : comment le client trouve Neutron

```bash
openstack catalog list
openstack catalog show network
openstack endpoint list --service network
openstack endpoint list --interface internal
```

✅ **Observé :** chaque service a **trois** points d’accès dans `RegionOne` : **public** (`http://cloud.lab.test:80/openstack-<service>`), **internal** (`http://internal.lab.test:80/openstack-<service>`) et **admin** (une adresse interne à Kubernetes, par exemple `http://10.152.183.106:9696` pour Neutron). Exceptions : Keystone, dont l’interface `admin` est aussi `internal.lab.test` ; le stockage objet (`swift`, `s3`), servi par `rgw.lab.test`.

| Interface | Adresse | Qui l’utilise |
|---|---|---|
| `public` | `cloud.lab.test` → `192.168.56.201` (Traefik public) | Clients : CLI, Horizon, apprenants |
| `internal` | `internal.lab.test` → `192.168.56.202` (Traefik interne) | Services entre eux : nova-compute → Neutron, Glance… ; c’est aussi l’adresse par laquelle les services **valident les jetons** auprès de Keystone |
| `admin` | `10.152.183.x` (adresse de service Kubernetes) | Joignable seulement depuis le cluster |

⚠ Les services résolvent `internal.lab.test` grâce au DNS de **Lab-Services**. Si Lab-Services est arrêté, Nova, Neutron et Cinder ne peuvent plus valider les jetons et répondent `503 … The Keystone service is temporarily unavailable` (constaté sur le lab).

Voir le catalogue dans le jeton lui-même, et l’appel qui suit :

```bash
openstack --debug network list 2>&1 | grep -E "^REQ: curl|^RESP: \[" | head -6
```

✅ **Observé** (18 octobre 2026) :

```text
REQ: curl -g -i -X GET http://cloud.lab.test/openstack-keystone/v3 …
RESP: [200] …
REQ: curl -g -i -X GET http://cloud.lab.test:80/openstack-neutron/v2.0/networks …
RESP: [200] …
```

**Comment le lire :** le client interroge d’abord Keystone (`GET …/v3` : découverte de la version de l’API), puis obtient un jeton (`POST …/auth/tokens`, que le client n’affiche pas sous la forme `REQ: curl`, car la requête contient le mot de passe), puis appelle Neutron. L’adresse de Neutron, `http://cloud.lab.test:80/openstack-neutron`, il l’a lue **dans le catalogue renvoyé avec le jeton** ; il ne la connaissait pas.

### Simulation : demander une autre interface

```bash
openstack --os-interface internal --debug network list 2>&1 | grep -E "^REQ: curl" | sed -n 2p
```

✅ **Observé :** l’appel part cette fois vers `http://internal.lab.test:80/openstack-neutron/v2.0/networks`. Depuis Lab-Sunbeam, les deux fonctionnent ; depuis Windows, seul `public` est prévu.

❓ **Question :** un apprenant obtient `EndpointNotFound` en appelant `openstack volume list`. Quelles sont les trois causes possibles ? *Réponse : service absent du catalogue, mauvaise région (`OS_REGION_NAME`), interface demandée inexistante pour ce service.*

---

## 1.4 Les jetons

### 1.4.1 Obtenir un jeton avec le client

```bash
openstack token issue
```

✅ **Observé :** les champs `expires` (une heure plus tard), `id` (le jeton, environ 180 caractères, commençant par `gAAAAA`), `project_id`, `user_id`.

### 1.4.2 Obtenir un jeton à la main, avec l’API

Le corps de la requête décrit **qui** (méthode `password`) et **pour quoi** (portée `scope`).

```bash
DEMO_PW=$(sed -n 's/^export OS_PASSWORD=//p' ~/demo-openrc | tr -d "\"'")
cat > /tmp/auth-projet.json <<EOF
{"auth": {
  "identity": {"methods": ["password"],
    "password": {"user": {"name": "demo", "domain": {"name": "users"}, "password": "$DEMO_PW"}}},
  "scope": {"project": {"name": "demo", "domain": {"name": "users"}}}
}}
EOF
curl -s -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-projet.json \
  -o /tmp/token-body.json -D /tmp/token-headers.txt
grep -i "^X-Subject-Token" /tmp/token-headers.txt | cut -c1-60
python3 -m json.tool /tmp/token-body.json | head -40
```

`-o` écrit le **corps** de la réponse dans un fichier, `-D` les **en-têtes** dans un autre. ⚠ Ne pas ajouter `-i` : il mêlerait les en-têtes au corps, et le fichier ne serait plus du JSON lisible (constaté : `Expecting value: line 1 column 1`).

✅ **Observé :**

- le jeton est dans l’**en-tête** `X-Subject-Token` (`gAAAAA…`), pas dans le corps ;
- le corps contient `methods: ["password"]`, `user`, `project`, `roles`, `expires_at`, `audit_ids`, et `catalog` (la liste des services).

```bash
python3 -c "import json; t=json.load(open('/tmp/token-body.json'))['token']; print('roles :', [r['name'] for r in t['roles']]); print('services au catalogue :', len(t['catalog'])); print('expire :', t['expires_at'])"
```

```text
roles : ['member', 'reader']
services au catalogue : 9
expire : 2026-10-17T21:47:25.000000Z
```

L’ordre des rôles peut varier d’un jeton à l’autre (`['reader', 'member']` le 18 octobre) : Keystone ne les trie pas.

### 1.4.3 Les portées : sans portée, projet, domaine, système

**Jeton sans portée** : il prouve l’identité, sans autoriser d’action. Première tentative, en retirant simplement le `scope` de la demande :

```bash
python3 - <<'EOF'
import json
d = json.load(open('/tmp/auth-projet.json'))
del d['auth']['scope']
json.dump(d, open('/tmp/auth-defaut.json', 'w'))
d['auth']['scope'] = 'unscoped'
json.dump(d, open('/tmp/auth-sans-portee.json', 'w'))
EOF
curl -s -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-defaut.json \
  | python3 -c "import sys,json; t=json.load(sys.stdin)['token']; print('roles' in t, 'catalog' in t, t.get('project',{}).get('name'))"
```

✅ **Observé :** `True True demo`. Surprise : le jeton **a** une portée, le projet `demo`. Le compte `demo` a un **projet par défaut** (`openstack user show demo --domain users -c default_project_id`) : sans portée demandée, Keystone lui applique celui-là.

Pour un vrai jeton sans portée, il faut le **demander explicitement** avec `"scope": "unscoped"` (second fichier préparé ci-dessus) :

```bash
curl -s -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-sans-portee.json \
  | python3 -c "import sys,json; t=json.load(sys.stdin)['token']; print('roles' in t, 'catalog' in t, list(t))"
```

✅ **Observé :** `False False ['methods', 'user', 'audit_ids', 'expires_at', 'issued_at']` : ni rôles, ni catalogue, ni projet. Un jeton sans portée sert seulement à découvrir ce à quoi on a droit :

```bash
UNSCOPED=$(curl -s -i -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-sans-portee.json | grep -i "^X-Subject-Token" | cut -d' ' -f2 | tr -d '\r')
curl -s "$OS_AUTH_URL/auth/projects" -H "X-Auth-Token: $UNSCOPED" | python3 -m json.tool | grep -E '"name"'
```

✅ **Observé :** `"name": "demo"` (le seul projet où `demo` a un rôle).

**Jeton de domaine** et **jeton système**, avec l’administrateur. D’abord, sur quoi `admin` a-t-il un rôle ?

```bash
openstack role assignment list --user admin --user-domain admin_domain --names
openstack role assignment list --system all --names
```

✅ **Observé :** `admin` a `admin` et `member` sur le projet `admin`, et `admin` sur le **domaine** `admin_domain` ; sur le **système**, seul `_charm-keystone-admin@Default` a un rôle. `admin` n’en a **aucun**.

```bash
env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME openstack --os-domain-name admin_domain token issue -c domain_id -c expires
env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME openstack --os-system-scope all token issue -c system -c expires
```

✅ **Observé :** un jeton de domaine (`domain_id` de `admin_domain`) ; puis ⛔ `The request you have made requires authentication. (HTTP 401)` pour le jeton système, puisqu’`admin` n’a aucun rôle sur le système : **on ne peut pas obtenir une portée sur laquelle on n’a rien**. Sur un cloud Sunbeam, l’administrateur travaille donc avec des jetons de **projet** ou de **domaine**.

⚠ **Piège :** le client `openstack` refuse de mélanger les portées. Si `OS_PROJECT_NAME` est défini, `--os-domain-name` est ignoré ou provoque une erreur : d’où les `env -u …`. La fonction `os_as` du fichier `lab-env.sh` s’en charge.

| Portée | Ce qu’elle cible | Exemples d’usage |
|---|---|---|
| sans portée | rien | découvrir ses projets (`/v3/auth/projects`) |
| projet | les ressources d’un projet | créer une VM, un réseau |
| domaine | les identités d’un domaine | créer des utilisateurs et des projets du domaine |
| système | la plateforme entière | lister les hyperviseurs, gérer les services et le catalogue |

### 1.4.4 Valider un jeton (ce que font les services)

Chaque service qui reçoit un jeton le fait vérifier par Keystone (on le voit dans le traceur : `GET /v3/auth/tokens (validation du jeton)`). On reproduit l’appel :

```bash
TOKEN=$(grep -i "^X-Subject-Token" /tmp/token-headers.txt | cut -d' ' -f2 | tr -d '\r')
ADMIN_TOKEN=$(openstack token issue -f value -c id)
curl -s -o /dev/null -w "validation : %{http_code}\n" -I "$OS_AUTH_URL/auth/tokens" \
  -H "X-Auth-Token: $ADMIN_TOKEN" -H "X-Subject-Token: $TOKEN"
```

✅ **Observé :** `validation : 200`. Le jeton de `demo` est vérifié avec le jeton de quelqu’un qui a le droit de vérifier (ici `admin`, en pratique le compte de service de Nova, de Neutron…).

### 1.4.5 Un jeton est un « porteur »

Quiconque possède le jeton agit comme son propriétaire, sans mot de passe :

```bash
curl -s "http://cloud.lab.test/openstack-neutron/v2.0/networks" -H "X-Auth-Token: $TOKEN" \
  | python3 -c "import sys,json; print([n['name'] for n in json.load(sys.stdin)['networks']])"
```

✅ **Observé :** `['demo-network', 'external-network']`, les réseaux visibles par `demo`, obtenus sans aucun mot de passe.

❓ **Question :** quelles conséquences pour la sécurité ? *Réponse : un jeton intercepté vaut un accès complet jusqu’à son expiration ; d’où HTTPS, une durée courte, et la révocation.*

### 1.4.6 Révoquer un jeton

```bash
openstack token revoke "$TOKEN"
curl -s -o /dev/null -w "validation après révocation : %{http_code}\n" -I "$OS_AUTH_URL/auth/tokens" \
  -H "X-Auth-Token: $ADMIN_TOKEN" -H "X-Subject-Token: $TOKEN"
curl -s -o /dev/null -w "appel Neutron après révocation : %{http_code}\n" \
  "http://cloud.lab.test/openstack-neutron/v2.0/networks" -H "X-Auth-Token: $TOKEN"
```

✅ **Observé :** `validation après révocation : 404` : Keystone refuse aussitôt le jeton. Pour Neutron, ça dépend du moment : le 17 octobre, `appel Neutron après révocation : 401` ; le 18 octobre, `200`.

**Pourquoi :** un service garde en **cache** les jetons qu’il vient de faire valider (bibliothèque `keystonemiddleware`), pour ne pas interroger Keystone à chaque appel. Un jeton que Neutron venait d’accepter (§ 1.4.5) est donc encore accepté juste après sa révocation. Mesuré le 18 octobre :

| Moment | Appel Neutron avec le jeton révoqué |
|---|---|
| Avant révocation | `200` |
| Juste après | `200` (cache) |
| 30 secondes après, et ensuite | `401` |

⚠ **Piège :** révoquer un jeton volé ne le coupe pas **instantanément** partout : il reste utilisable quelques secondes, le temps que les caches des services expirent.

### 1.4.7 Durée de vie et clés Fernet

```bash
python3 -c "import json; print(json.load(open('/tmp/token-body.json'))['token']['expires_at'])"
date -u +%Y-%m-%dT%H:%M:%S
$K exec keystone-0 -c keystone -- sh -c 'awk "/^\[/{p=(\$0==\"[token]\" || \$0==\"[fernet_tokens]\")} p && NF" /etc/keystone/keystone.conf; ls -l /etc/keystone/fernet-keys/'
```

**La commande, mot par mot :** `awk` lit le fichier ligne par ligne ; à chaque titre de section (`[…]`), il note si c’est `[token]` ou `[fernet_tokens]` ; il n’affiche que les lignes non vides (`NF`) de ces deux sections. `ls -l` liste les clés. (Un simple `grep -A6 "^\[token\]"` afficherait aussi le début des sections voisines.)

✅ **Observé** (17 octobre ; le 18, après une rotation, les clés étaient `0`, `8`, `9`, `10` : les numéros changent chaque jour, voir `7-securite/04-rotation-secrets.md`, § 4.3) :

```text
[token]
expiration = 3600
allow_expired_window = 169200

[fernet_tokens]
max_active_keys = 4

-rw------- 1 keystone keystone    44 Oct 17 17:10 0
-rw------- 1 keystone keystone    44 Oct 15 14:10 7
-rw------- 1 keystone keystone    44 Oct 15 14:10 8
-rw------- 1 keystone keystone    44 Oct 16 12:45 9
drwx------ 2 root     root     16384 Oct  9 10:35 lost+found
```

| Réglage | Signification |
|---|---|
| `expiration = 3600` | Un jeton vit **une heure** |
| `allow_expired_window = 169200` | Pendant 47 heures après expiration, un service peut encore faire accepter un jeton expiré pour **terminer** une opération longue déjà commencée |
| `max_active_keys = 4` | Quatre clés Fernet au plus |
| Fichiers `0`, `7`, `8`, `9` | Les clés : `0` est la clé **en attente** (la prochaine primaire) ; la plus grande, `9`, est la clé **primaire**, qui chiffre les nouveaux jetons ; les autres servent encore à **déchiffrer** les jetons émis avant la dernière rotation |

`provider = fernet` n’est pas écrit : c’est la valeur par défaut. Le charm Keystone ne propose pas d’option Juju pour la durée des jetons ni pour la rotation des clés (`juju config -m openstack keystone` n’en montre aucune) : ces réglages sont fixés par le charm.

❓ **Question :** un jeton Fernet ne contient pas le catalogue, alors que la réponse de `POST /auth/tokens` le contient. Pourquoi ? *Réponse : le jeton Fernet est un petit message chiffré (utilisateur, portée, méthodes, expiration, identifiant d’audit) ; Keystone reconstruit le reste (rôles, catalogue) à chaque validation. Il n’est stocké nulle part : c’est pourquoi la révocation passe par une liste d’événements de révocation.*

---

## 1.5 Les sources d’identité

### 1.5.1 Identités locales (SQL)

```bash
openstack user show demo --domain users
$K exec keystone-0 -c keystone -- sh -c 'sed -n "/^\[identity\]/,/^\[/p" /etc/keystone/keystone.conf | grep -v "^\["'
```

La seconde commande n’affiche **que** la section `[identity]`. ⚠ Ne pas utiliser `grep -A8 "^\[identity\]"` : sur le lab, les lignes suivantes sont celles de la section `[database]`, dont la chaîne de connexion contient **le mot de passe MySQL de Keystone** (constaté).

✅ **Observé :**

```text
driver = sql
default_domain_id = default
domain_specific_drivers_enabled = True
domain_config_dir = /etc/keystone/domains
```

**Comment le lire :** les comptes sont en base SQL (`driver = sql`) ; mais `domain_specific_drivers_enabled = True` : Keystone accepte qu’**un domaine** ait sa **propre** source d’identité (un annuaire LDAP, par exemple), décrite par un fichier dans `/etc/keystone/domains`. C’est ce qui permet de brancher l’annuaire d’un client sur son seul domaine (module 11).

### 1.5.2 Comptes de service

```bash
openstack user list --domain service_domain
openstack role assignment list --project services --project-domain service_domain --names
```

✅ **Observé :** un compte par service, nommé `svc_<service>-<suffixe aléatoire>` (`svc_nova-F8OChi`, `svc_neutron-Jbci5R`…), plus `horizon`, `cinder-volume`, `openstack-hypervisor` et `svc_remote_…`, chacun avec **deux** rôles sur le projet `services` : `admin` et `service`. Ce sont eux qui valident les jetons des utilisateurs (§ 1.4.4) et qui s’appellent entre eux (nova-compute → Neutron).

### 1.5.3 Identités externes (fédération)

```bash
openstack identity provider list
openstack mapping list
```

✅ **Observé :** deux listes vides, tant que la fédération Keycloak n’est pas configurée (module 11). Keycloak tourne pourtant dans Lab-Services : **présence d’un fournisseur d’identité ne veut pas dire fédération active**.

### 1.5.4 Politique de mots de passe

```bash
$K exec keystone-0 -c keystone -- sh -c 'grep -E "^\[security_compliance\]" -A15 /etc/keystone/keystone.conf'
```

✅ **Observé :** aucune ligne, et `command terminated with exit code 1` (`grep` n’a rien trouvé) : la section est **absente**, donc réglages par défaut : pas de verrouillage après échecs, pas d’expiration des mots de passe. À comparer avec les recommandations de l’ANSSI vues au module 11.

### Simulation : un utilisateur change son mot de passe

```bash
openstack user create lab-pwtest --domain users --password "$LAB_PW"
openstack role add --user lab-pwtest --user-domain users --project demo --project-domain users member
os_as lab-pwtest users projet demo users -- user password set --original-password "$LAB_PW" --password "${LAB_PW}-2"
```

⚠ **Observé :** avec le client `openstack` 7.5.0 du lab, cette commande **plante** : `Proxy.update_user() missing 1 required positional argument: 'user'`. C’est un défaut du client, pas de Keystone : le mot de passe n’est pas changé. On passe donc par l’**API**, ce que fait de toute façon le client :

```bash
USER_ID=$(openstack user show lab-pwtest --domain users -f value -c id)
[ -n "$USER_ID" ] || echo "ATTENTION : identifiant vide, relancer la commande precedente"
T=$(os_as lab-pwtest users projet demo users -- token issue -f value -c id)
curl -s -o /dev/null -w "changement : %{http_code}\n" -X POST "$OS_AUTH_URL/users/$USER_ID/password" \
  -H "X-Auth-Token: $T" -H "Content-Type: application/json" \
  -d "{\"user\": {\"original_password\": \"$LAB_PW\", \"password\": \"${LAB_PW}-2\"}}"
LAB_PW="${LAB_PW}-2" os_as lab-pwtest users projet demo users -- token issue -c expires
openstack user delete lab-pwtest --domain users
```

✅ **Observé** (18 octobre 2026) : `changement : 204`, puis un jeton obtenu avec le nouveau mot de passe. Un utilisateur change son mot de passe avec **son propre** jeton, en prouvant qu’il connaît l’ancien.

⚠ **Pièges :**

- si `USER_ID` est vide (commande précédente en échec), la requête part vers `…/v3/users/password` et Keystone répond `500` (constaté dans son journal) ;
- `os_as` lit `LAB_PW` : d’où le `LAB_PW=… os_as` sur une seule ligne, pour ne pas modifier la variable de la session.

---

## 1.6 Les politiques d’accès

### 1.6.1 Où en est Secure RBAC sur ton lab (à faire en premier)

```bash
for pc in keystone-0/keystone:/etc/keystone/keystone.conf nova-0/nova-api:/etc/nova/nova.conf \
          neutron-0/neutron-server:/etc/neutron/neutron.conf glance-0/glance-api:/etc/glance/glance-api.conf \
          placement-0/placement-api:/etc/placement/placement.conf; do
  pod=${pc%%/*}; rest=${pc#*/}; c=${rest%%:*}; f=${rest#*:}
  echo "== $c"; $K exec "$pod" -c "$c" -- sh -c "grep -A6 '^\[oslo_policy\]' $f 2>/dev/null | grep -E 'enforce_scope|enforce_new_defaults|policy_file' || echo '  (valeurs par defaut)'"
done
```

✅ **Observé :** les cinq services répondent `(valeurs par defaut)` : aucun ne fixe ces options dans son fichier.

« Valeurs par défaut » : pour les lire, génère la configuration de référence de la bibliothèque de politiques :

```bash
$K exec keystone-0 -c keystone -- oslo-config-generator --namespace oslo.policy 2>/dev/null | grep -B2 -A2 -E "^#enforce_(scope|new_defaults)"
```

✅ **Observé :** `#enforce_scope = false` et `#enforce_new_defaults = false`.

⚠ Ce sont les défauts de la **bibliothèque** ; un service peut les surcharger dans son code. La preuve se lit dans les règles (§ 1.6.2) et dans les essais des TP 2 à 4. Note le résultat :

| Service | `enforce_scope` | `enforce_new_defaults` |
|---|---|---|
| Keystone | | |
| Nova | | |
| Neutron | | |
| Glance | | |
| Placement | | |

| Si… | Alors |
|---|---|
| `enforce_scope = True` et `enforce_new_defaults = True` | Secure RBAC complet : `reader`, `member`, `manager`, `admin` et les portées s’appliquent comme dans le support |
| `enforce_new_defaults = True` seul | Les nouveaux rôles s’appliquent, mais une portée inadaptée n’est pas refusée |
| les deux à `False` | Anciennes règles : `admin` sur n’importe quel projet = administrateur de tout le cloud (« admin quelque part, admin partout ») |

### 1.6.2 Lire une règle

```bash
$K exec keystone-0 -c keystone -- oslopolicy-policy-generator --namespace keystone 2>/dev/null \
  | grep -E '"identity:(list_users|create_user|list_projects|create_project|create_grant)"'
```

✅ **Observé sur le lab :**

```text
"identity:create_grant": "(rule:admin_required) or ((role:admin and domain_id:%(target.user.domain_id)s and domain_id:%(target.project.domain_id)s) or …"
"identity:list_projects": "(rule:admin_required) or (role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s)"
"identity:create_project": "rule:admin_required"
"identity:list_users": "(rule:admin_required) or (role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s)"
"identity:create_user": "rule:admin_required"
```

**Comment le lire :**

| Règle | Qui peut |
|---|---|
| `list_users`, `list_projects` | Un `admin` (`rule:admin_required`), **ou** un `reader` sur le système, **ou** un `reader` sur le domaine visé |
| `create_user`, `create_project` | **Seulement** un `admin` |
| `create_grant` (donner un rôle) | Un `admin`, éventuellement limité au domaine de l’utilisateur et de la cible |

Deux conclusions, essentielles pour la suite :

1. **`rule:admin_required` est en tête de chaque règle**, et il ne regarde pas la portée : un rôle `admin`, **n’importe où**, suffit. C’est la marque des anciennes règles (`enforce_new_defaults = false`), et l’origine du « admin quelque part, admin partout » (TP 2, § 2.3).
2. **Aucune règle ne mentionne `manager`** : sur ce Keystone, un gestionnaire de domaine ne peut ni créer de comptes ni créer de projets. Le TP 2 (§ 2.4) le vérifie.

Même exercice pour Nova et Neutron :

```bash
$K exec nova-0 -c nova-api -- oslopolicy-policy-generator --namespace nova 2>/dev/null | grep -E '"os_compute_api:(servers:create|os-hypervisors:list)"'
$K exec neutron-0 -c neutron-server -- oslopolicy-policy-generator --namespace neutron 2>/dev/null | grep -E '"(create_network|create_network:shared)"'
```

✅ **Observé :**

```text
"os_compute_api:os-hypervisors:list": "rule:context_is_admin"
"os_compute_api:servers:create": "rule:project_member_or_admin"
"create_network": "(rule:admin_only) or (role:member and project_id:%(project_id)s)"
"create_network:shared": "rule:admin_only"
```

Lister les hyperviseurs : réservé à l’administrateur ; créer une VM : membre du projet ou administrateur ; créer un réseau : administrateur, ou `member` **du projet** ; le rendre partagé : administrateur seulement.

❓ **Question :** pourquoi `create_network:shared` a-t-il une règle distincte de `create_network` ? *Réponse : un réseau partagé est visible par tous les projets ; c’est une action d’administration, alors que créer un réseau privé est un droit de membre.*

### 1.6.3 Expliquer un refus

```bash
source ~/demo-openrc
openstack user list
openstack hypervisor list
openstack network create lab-partage --share
source ~/admin-openrc
```

⛔ **Refus observés** pour les trois, avec un code `403` :

```text
ForbiddenException: 403: … You are not authorized to perform the requested action: identity:list_users.
ForbiddenException: 403: … Policy doesn't allow os_compute_api:os-hypervisors:list-detail to be performed.
Error while executing command: ForbiddenException: 403, (rule:create_network and rule:create_network:shared) is disallowed by policy
```

Chaque message **nomme la règle** refusée : l’apprenant la relie à celle lue en 1.6.2.

| Code | Signification |
|---|---|
| `401 Unauthorized` | Keystone ne sait pas qui tu es (jeton absent, expiré, révoqué, mauvais mot de passe) |
| `403 Forbidden` | Keystone sait qui tu es, mais la **politique** du service refuse cette action à ton rôle sur cette portée |

---

## 1.7 Exploitation : relier un refus aux journaux

```bash
source ~/demo-openrc
openstack --debug user list 2>&1 | grep -i -E "X-Openstack-Request-Id|RESP: \[403\]" | head -3
source ~/admin-openrc
$K logs keystone-0 -c keystone --since=2m | grep -E " 403 |Forbidden|not authorized" | tail -3
```

✅ **Observé :** le client reçoit `RESP: [403] … X-Openstack-Request-Id: req-40a5efb0-…` ; le journal de Keystone contient la même requête :

```text
… ERROR keystone.server.flask.application [None req-40a5efb0-a28e-4ac4-a530-625bd7acda2a …] keystone.exception.ForbiddenAction: You are not authorized to perform the requested action: identity:list_users.
… "GET /openstack-keystone/v3/users HTTP/1.1" 403 132 …
```

L’identifiant de requête relie le refus vu par l’utilisateur à la ligne du serveur.

---

## Nettoyage

```bash
rm -f /tmp/auth-projet.json /tmp/auth-defaut.json /tmp/auth-sans-portee.json /tmp/token-body.json /tmp/token-headers.txt
openstack user delete lab-pwtest --domain users 2>/dev/null
```

## Pour aller plus loin

- Lancer le traceur (`2-trace/PAS-A-PAS.md`) pendant le § 1.4 : chaque `POST /auth/tokens` et chaque validation `GET /auth/tokens` apparaît en direct.
- Mesurer la taille d’un jeton : `openstack token issue -f value -c id | wc -c`.

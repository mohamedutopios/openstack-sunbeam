# TP 1 : Keystone en profondeur

> Statut : rédigé, **pas encore rejoué sur le lab**. Voir les conventions dans [README.md](README.md).

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

✅ **Attendu :** l’application `keystone` est `active` ; un pod `keystone-0` avec deux conteneurs (`charm`, `keystone`) ; des lignes Apache `"POST /openstack-keystone/v3/auth/tokens HTTP/1.1" 201`.

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

✅ **Attendu (cloud Sunbeam du lab) :**

| Entité | Valeurs typiques |
|---|---|
| Domaines | `admin_domain`, `service_domain`, `users`, `Default` |
| Projets | `admin` (dans `admin_domain`), `services` (dans `service_domain`), `demo` (dans `users`) |
| Utilisateurs | `admin` ; un compte par service (`nova`, `neutron`, `glance`, `placement`, `cinder`…) dans `service_domain` ; `demo` dans `users` |
| Rôles | au moins `admin`, `member`, `reader`, souvent `manager` et `service` |
| Affectations | `admin` → `admin` sur le projet `admin` ; comptes de service → `admin` ou `service` sur `services` ; `demo` → `member` sur `demo` |

### Exercice

Les apprenants complètent ce tableau à partir de leurs sorties :

| Question | Commande | Leur réponse |
|---|---|---|
| Combien de domaines, et lequel héberge les comptes des services ? | `openstack domain list` | |
| Quels utilisateurs ont un rôle sur le projet `services` ? | `openstack role assignment list --project services --project-domain service_domain --names` | |
| Quel rôle `admin` implique-t-il ? | `openstack implied role list` | |
| L’utilisateur `demo` peut-il voir le projet `admin` ? | `openstack role assignment list --user demo --user-domain users --names` | |

⚠ **Piège :** `openstack user list` sans `--domain` liste les utilisateurs **de tous les domaines** quand on est administrateur global. Deux utilisateurs peuvent porter le même nom dans deux domaines différents (on le démontre au TP 2) : un nom seul ne désigne pas un compte.

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

✅ **Attendu :** chaque service a un point d’accès **public** (`http://cloud.lab.test/openstack-<service>`) et **internal** (`http://internal.lab.test/openstack-<service>`), dans la région `RegionOne`.

| Interface | Adresse | Qui l’utilise |
|---|---|---|
| `public` | `cloud.lab.test` → `192.168.56.201` (Traefik public) | Clients : CLI, Horizon, apprenants |
| `internal` | `internal.lab.test` → `192.168.56.202` (Traefik interne) | Services entre eux : nova-compute → Neutron, Glance… |

Voir le catalogue dans le jeton lui-même, et l’appel qui suit :

```bash
openstack --debug network list 2>&1 | grep -E "^REQ: curl|^RESP: \[" | head -6
```

✅ **Attendu :** une requête vers `…/openstack-keystone/v3/auth/tokens`, puis une vers `http://cloud.lab.test:80/openstack-neutron/v2.0/networks`. Le client a lu l’adresse de Neutron **dans le catalogue renvoyé avec le jeton** ; il ne la connaissait pas.

### Simulation : demander une autre interface

```bash
openstack --os-interface internal --debug network list 2>&1 | grep -E "^REQ: curl" | sed -n 2p
```

✅ **Attendu :** l’appel part cette fois vers `internal.lab.test`. Depuis Lab-Sunbeam, les deux fonctionnent ; depuis Windows, seul `public` est prévu.

❓ **Question :** un apprenant obtient `EndpointNotFound` en appelant `openstack volume list`. Quelles sont les trois causes possibles ? *Réponse : service absent du catalogue, mauvaise région (`OS_REGION_NAME`), interface demandée inexistante pour ce service.*

---

## 1.4 Les jetons

### 1.4.1 Obtenir un jeton avec le client

```bash
openstack token issue
```

✅ **Attendu :** les champs `expires`, `id` (le jeton, environ 180 à 250 caractères), `project_id`, `user_id`.

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
curl -s -i -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-projet.json \
  -o /tmp/token-body.json -D /tmp/token-headers.txt
grep -i "^X-Subject-Token" /tmp/token-headers.txt | cut -c1-60
python3 -m json.tool /tmp/token-body.json | head -40
```

✅ **Attendu :**

- le jeton est dans l’**en-tête** `X-Subject-Token`, pas dans le corps ;
- le corps contient `methods: ["password"]`, `user`, `project`, `roles` (`member`, `reader`), `expires_at`, `audit_ids`, et `catalog` (la liste des services).

```bash
python3 -c "import json; t=json.load(open('/tmp/token-body.json'))['token']; print('roles :', [r['name'] for r in t['roles']]); print('services au catalogue :', len(t['catalog'])); print('expire :', t['expires_at'])"
```

### 1.4.3 Les portées : sans portée, projet, domaine, système

**Jeton sans portée** (aucun `scope`) : il prouve l’identité, sans autoriser d’action.

```bash
python3 - <<'EOF'
import json
d = json.load(open('/tmp/auth-projet.json'))
del d['auth']['scope']
json.dump(d, open('/tmp/auth-sans-portee.json', 'w'))
EOF
curl -s -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-sans-portee.json \
  | python3 -c "import sys,json; t=json.load(sys.stdin)['token']; print('roles' in t, 'catalog' in t, list(t))"
```

✅ **Attendu :** `False False`, et la liste des clés sans `roles` ni `catalog`. Un jeton sans portée sert seulement à découvrir ce à quoi on a droit :

```bash
UNSCOPED=$(curl -s -i -X POST "$OS_AUTH_URL/auth/tokens" -H "Content-Type: application/json" -d @/tmp/auth-sans-portee.json | grep -i "^X-Subject-Token" | cut -d' ' -f2 | tr -d '\r')
curl -s "$OS_AUTH_URL/auth/projects" -H "X-Auth-Token: $UNSCOPED" | python3 -m json.tool | grep -E '"name"'
```

✅ **Attendu :** `demo` (le seul projet où `demo` a un rôle).

**Jeton de domaine** et **jeton système**, avec l’administrateur :

```bash
openstack role assignment list --user admin --user-domain admin_domain --names
openstack role assignment list --system all --names
```

Si `admin` a un rôle sur son domaine, ou sur le système :

```bash
env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME openstack --os-domain-name admin_domain token issue -c domain_id -c expires
env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME openstack --os-system-scope all token issue -c system -c expires
```

✅ **Attendu :** un jeton avec `domain_id`, et un jeton avec `system: {'all': True}`. ⛔ **Refus attendu** (`401`) si le compte n’a **aucun rôle** sur la cible demandée : on ne peut pas obtenir une portée sur laquelle on n’a rien.

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

✅ **Attendu :** `200`. Le jeton de `demo` est vérifié avec le jeton de quelqu’un qui a le droit de vérifier (ici `admin`, en pratique le compte de service de Nova, de Neutron…).

### 1.4.5 Un jeton est un « porteur »

Quiconque possède le jeton agit comme son propriétaire, sans mot de passe :

```bash
curl -s "http://cloud.lab.test/openstack-neutron/v2.0/networks" -H "X-Auth-Token: $TOKEN" \
  | python3 -c "import sys,json; print([n['name'] for n in json.load(sys.stdin)['networks']])"
```

✅ **Attendu :** les réseaux visibles par `demo`.

❓ **Question :** quelles conséquences pour la sécurité ? *Réponse : un jeton intercepté vaut un accès complet jusqu’à son expiration ; d’où HTTPS, une durée courte, et la révocation.*

### 1.4.6 Révoquer un jeton

```bash
openstack token revoke "$TOKEN"
curl -s -o /dev/null -w "validation après révocation : %{http_code}\n" -I "$OS_AUTH_URL/auth/tokens" \
  -H "X-Auth-Token: $ADMIN_TOKEN" -H "X-Subject-Token: $TOKEN"
curl -s -o /dev/null -w "appel Neutron après révocation : %{http_code}\n" \
  "http://cloud.lab.test/openstack-neutron/v2.0/networks" -H "X-Auth-Token: $TOKEN"
```

✅ **Attendu :** `404` à la validation, `401` chez Neutron. ⚠ **Piège :** les services gardent les validations en cache quelques minutes (`keystonemiddleware`, `token_cache_time`) : un jeton révoqué peut encore être accepté par un service peu de temps après la révocation.

### 1.4.7 Durée de vie et clés Fernet

```bash
python3 -c "import json; print(json.load(open('/tmp/token-body.json'))['token']['expires_at'])"
date -u +%Y-%m-%dT%H:%M:%S
$K exec keystone-0 -c keystone -- sh -c 'grep -A6 "^\[token\]" /etc/keystone/keystone.conf; grep -A4 "^\[fernet_tokens\]" /etc/keystone/keystone.conf; ls -l /etc/keystone/fernet-keys/'
juju config -m openstack keystone | grep -B1 -A4 -i -E "token-expiration|fernet"
```

✅ **Attendu :** `provider = fernet` ; une durée de vie (`expiration`, en secondes, souvent 3600) ; un répertoire `fernet-keys` contenant plusieurs clés numérotées (`0` = clé en attente, la plus grande = clé primaire).

❓ **Question :** un jeton Fernet ne contient pas le catalogue, alors que la réponse de `POST /auth/tokens` le contient. Pourquoi ? *Réponse : le jeton Fernet est un petit message chiffré (utilisateur, portée, méthodes, expiration, identifiant d’audit) ; Keystone reconstruit le reste (rôles, catalogue) à chaque validation. Il n’est stocké nulle part : c’est pourquoi la révocation passe par une liste d’événements de révocation.*

---

## 1.5 Les sources d’identité

### 1.5.1 Identités locales (SQL)

```bash
openstack user show demo --domain users
$K exec keystone-0 -c keystone -- sh -c 'grep -E "^\[identity\]" -A8 /etc/keystone/keystone.conf'
```

✅ **Attendu :** `driver = sql` (ou valeur par défaut) ; `domain_specific_drivers_enabled` absent ou `false` : tous les domaines utilisent la base SQL de Keystone.

### 1.5.2 Comptes de service

```bash
openstack user list --domain service_domain
openstack role assignment list --project services --project-domain service_domain --names
```

✅ **Attendu :** un compte par service (`nova`, `neutron`, `glance`, `placement`, `cinder`…), avec un rôle sur le projet `services`. Ce sont eux qui valident les jetons des utilisateurs (§ 1.4.4) et qui s’appellent entre eux (nova-compute → Neutron).

### 1.5.3 Identités externes (fédération)

```bash
openstack identity provider list
openstack mapping list
```

✅ **Attendu sur ce lab :** listes vides tant que la fédération Keycloak n’est pas configurée (module 11). Keycloak tourne pourtant dans Lab-Services : **présence d’un fournisseur d’identité ne veut pas dire fédération active**.

### 1.5.4 Politique de mots de passe

```bash
$K exec keystone-0 -c keystone -- sh -c 'grep -E "^\[security_compliance\]" -A15 /etc/keystone/keystone.conf'
```

✅ **Attendu :** section absente ou réglages par défaut (pas de verrouillage, pas d’expiration). À comparer avec les recommandations de l’ANSSI vues au module 11.

### Simulation : un utilisateur change son mot de passe

```bash
openstack user create lab-pwtest --domain users --password "$LAB_PW"
openstack role add --user lab-pwtest --user-domain users --project demo --project-domain users member
os_as lab-pwtest users projet demo users -- user password set --original-password "$LAB_PW" --password "${LAB_PW}-2"
LAB_PW="${LAB_PW}-2" os_as lab-pwtest users projet demo users -- token issue -c expires
openstack user delete lab-pwtest --domain users
```

✅ **Attendu :** le changement réussit, le nouveau mot de passe fonctionne. ⚠ **Piège :** `os_as` lit `LAB_PW` ; d’où le `LAB_PW=… os_as` sur une seule ligne pour ne pas modifier la variable de la session.

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

Note le résultat dans ce tableau, il conditionne les TP 2 à 4 :

| Service | `enforce_scope` | `enforce_new_defaults` |
|---|---|---|
| Keystone | | |
| Nova | | |
| Neutron | | |
| Glance | | |
| Placement | | |

« Valeurs par défaut » : la valeur dépend de la version du service. Pour la lire, génère la configuration effective :

```bash
$K exec keystone-0 -c keystone -- oslo-config-generator --namespace oslo.policy 2>/dev/null | grep -B2 -A2 -E "^#enforce_(scope|new_defaults)"
```

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

✅ **Attendu :** des règles du type `"identity:list_users": "(role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s)"` (le texte exact dépend de la version).

Lecture : **lister les utilisateurs** est permis à qui a `reader` **sur le système**, ou `reader` **sur le domaine** des utilisateurs listés.

Même exercice pour Nova et Neutron :

```bash
$K exec nova-0 -c nova-api -- oslopolicy-policy-generator --namespace nova 2>/dev/null | grep -E '"os_compute_api:(servers:create|os-hypervisors:list)"'
$K exec neutron-0 -c neutron-server -- oslopolicy-policy-generator --namespace neutron 2>/dev/null | grep -E '"(create_network|create_network:shared)"'
```

❓ **Question :** pourquoi `create_network:shared` a-t-il une règle distincte de `create_network` ? *Réponse : un réseau partagé est visible par tous les projets ; c’est une action d’administration, alors que créer un réseau privé est un droit de membre.*

### 1.6.3 Expliquer un refus

```bash
source ~/demo-openrc
openstack user list
openstack hypervisor list
openstack network create lab-partage --share
source ~/admin-openrc
```

⛔ **Refus attendu** pour les trois, avec un code `403` (« You are not authorized… » ou « Policy doesn’t allow… »). L’apprenant relie chaque refus à la règle lue en 1.6.2.

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

✅ **Attendu :** l’identifiant de requête renvoyé au client se retrouve dans les journaux de Keystone, avec le refus.

---

## Nettoyage

```bash
rm -f /tmp/auth-projet.json /tmp/auth-sans-portee.json /tmp/token-body.json /tmp/token-headers.txt
openstack user delete lab-pwtest --domain users 2>/dev/null
```

## Pour aller plus loin

- Lancer le traceur (`trace/PAS-A-PAS.md`) pendant le § 1.4 : chaque `POST /auth/tokens` et chaque validation `GET /auth/tokens` apparaît en direct.
- Mesurer la taille d’un jeton : `openstack token issue -f value -c id | wc -c`.

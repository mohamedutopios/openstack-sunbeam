# Démo 3 : rotation des secrets et des clés de jetons

> **Statut : rédigé, pas encore rejoué sur le lab.** Les observations citées (secrets Juju, clés Fernet) viennent du lab, les 16 et 17 octobre 2026. Support : modules 11 et 12.

## Objectif

Savoir **quels** secrets protègent le cloud, **lesquels tournent tout seuls** et à quel rythme, le **vérifier**, et savoir quoi faire pour ceux qui ne tournent pas. C’est la suite logique de la démo 2 : les comptes de service ne relèvent pas d’une politique de mots de passe, mais d’une **gestion de secrets**.

## 3.1 Comprendre : quels secrets, pourquoi les faire tourner

| Secret | Ce qu’il protège | Si on le vole | Rotation |
|---|---|---|---|
| **Clés Fernet** de Keystone | Le chiffrement des jetons | On fabrique des jetons valides pour n’importe qui | Automatique, par le charm |
| **Clés de credentials** de Keystone | Le chiffrement des justificatifs stockés (secrets TOTP, clés EC2) | On déchiffre ces justificatifs | Automatique, par le charm |
| **Mots de passe des comptes de service** (`svc_nova-…`) | L’accès des services à Keystone | On agit comme un service, avec le rôle `admin` sur `services` | Automatique, par le charm |
| **Mots de passe des bases** (MySQL) et de RabbitMQ | Les données et la messagerie du cloud | Accès direct aux données de tous les projets | Pas de rotation automatique observée |
| **Mot de passe d’`admin`** | L’administration du cloud | Tout | Manuelle, avec précaution (3.6) |
| **Identifiants d’application** | Les scripts des utilisateurs | Les droits de leur propriétaire sur un projet | Par l’utilisateur, avec une expiration (3.7) |

**Pourquoi tourner :** un secret qui tourne limite la **durée** pendant laquelle un vol reste exploitable, sans qu’on ait besoin de détecter le vol. C’est aussi ce que demande la recommandation R26 de l’ANSSI en cas de compromission : renouveler sans délai.

## 3.2 Inventaire : les secrets gérés par Juju

**À quoi ça sert :** sur Sunbeam, les secrets des services ne sont pas dans des fichiers épars : ils sont rangés et distribués par **Juju** (guide `4-juju-cli/`, démo 7). On en dresse la liste, avec leur politique de rotation.

```bash
sunbeam utils juju-login
juju secrets -m openstack | head -3
juju secrets -m openstack --format json | python3 -c "
import sys, json, collections
d = json.load(sys.stdin)
c = collections.Counter((v.get('owner'), v.get('rotation', 'never')) for v in d.values())
for (owner, rot), n in sorted(c.items()): print(f'{owner:32} {rot:8} {n}')
"
```

**La commande, mot par mot :** `juju secrets -m openstack` liste les secrets du modèle ; `--format json` les donne sous une forme lisible par un script ; le petit programme Python les **compte** par propriétaire et par politique de rotation.

✅ **Attendu** (d’après le lab) : environ 55 secrets. La plupart en `never` : ceux des routeurs MySQL, de `mysql`, de `rabbitmq`, des certificats. Ceux du charm `keystone` en `daily` ou `monthly`.

**Comment le lire :**

| Colonne `Rotation` | Signification |
|---|---|
| `never` | Le charm ne renouvelle pas ce secret de lui-même |
| `daily`, `monthly` | Le charm le renouvelle à cette fréquence ; Juju le réveille à l’échéance |

Le détail d’un secret de Keystone, **sans** son contenu :

```bash
juju secrets -m openstack --format yaml | grep -B2 -A10 "owner: keystone" | grep -E "^[a-z0-9]{20}:|label|rotation|rotates|revision|created|updated"
```

✅ **Attendu** (observé le 17 octobre pour un ancien compte de Heat) :

```text
  label: credentials_svc_heat
  rotation: monthly
  rotates: 2026-11-16T14:36:14Z
  created: 2026-10-16T14:36:14Z
```

**Comment le lire :** ce secret contient le mot de passe d’un compte de service ; il a été créé le 16 octobre et sera **renouvelé le 16 novembre**, un mois plus tard (`rotates`). Chaque renouvellement augmente sa **révision** (`revision`).

## 3.3 Les clés de jetons (Fernet)

### Observer les clés

```bash
sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- sh -c '
  grep -A2 "^\[fernet_tokens\]" /etc/keystone/keystone.conf
  ls -l --time-style=+%F_%H:%M /etc/keystone/fernet-keys/ /etc/keystone/credential-keys/ 2>/dev/null'
```

✅ **Attendu** (relevé le 17 octobre, répertoire `fernet-keys`) :

```text
[fernet_tokens]
max_active_keys = 4
-rw------- 1 keystone keystone 44 2026-10-17_17:10 0
-rw------- 1 keystone keystone 44 2026-10-15_14:10 7
-rw------- 1 keystone keystone 44 2026-10-15_14:10 8
-rw------- 1 keystone keystone 44 2026-10-16_12:45 9
```

⚠ **à vérifier :** l’existence et le contenu de `credential-keys/` sur le lab.

**Comment le lire :** Keystone tient **4** clés au plus (`max_active_keys`), qui jouent trois rôles :

| Clé | Rôle |
|---|---|
| `0` | **En attente** (*staged*) : la prochaine clé primaire. Elle est déjà connue de tous, pour qu’aucun jeton ne soit refusé au moment de la bascule |
| La plus grande, `9` | **Primaire** : elle chiffre les **nouveaux** jetons |
| Les autres, `7` et `8` | **Secondaires** : elles ne chiffrent plus, mais **déchiffrent** encore les jetons émis avant la dernière rotation |

**Une rotation :** la clé `0` devient primaire sous un nouveau numéro (`10`), une nouvelle clé `0` est créée, et la plus ancienne secondaire (`7`) est détruite. Les dates des fichiers montrent que les rotations ont lieu à quelques heures ou jours d’intervalle.

### Pourquoi une rotation ne casse pas les jetons en cours

Un jeton est valable **une heure** (`expiration = 3600`, TP 1 des `labs-iam/`, § 1.4.7). Avec 4 clés et une rotation par jour, une clé reste utilisable environ **trois jours** après avoir été primaire : bien plus que la durée de vie d’un jeton.

⚠ **Piège :** `allow_expired_window = 169200` (47 heures, même paragraphe du TP 1) permet aux services de faire accepter un jeton **expiré** pour terminer une opération longue. Si les clés tournaient trop vite, ces jetons deviendraient indéchiffrables. Le réglage du nombre de clés et de la fréquence se fait **ensemble**.

### Vérifier qu’un jeton survit à une rotation

Le charm décide du moment de la rotation. On ne peut donc pas la déclencher à la demande, sauf action prévue par le charm :

```bash
juju actions -m openstack keystone
```

⚠ **à vérifier :** si une action de rotation existe (par exemple `rotate-fernet-keys`), elle permet de faire la démonstration en direct. Sinon, on la fait **sur deux jours** :

1. Le premier jour, garder la **liste** des clés (un jeton ne dure qu’une heure : on ne peut pas en garder un jusqu’au lendemain) :

   ```bash
   sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- ls /etc/keystone/fernet-keys/ > ~/lab-sec-cles-j1.txt
   ```

2. Le lendemain, comparer :

   ```bash
   sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- ls /etc/keystone/fernet-keys/ | diff ~/lab-sec-cles-j1.txt -
   ```

✅ **Attendu :** des numéros ont changé : le plus ancien a disparu, un nouveau est apparu. La rotation a eu lieu sans intervention, et le cloud a continué de fonctionner.

## 3.4 Les mots de passe des comptes de service

**À quoi ça sert :** montrer que ces mots de passe ne sont **jamais** saisis par une personne, et qu’ils tournent.

```bash
juju secrets -m openstack --format yaml | grep -B2 -A10 "label: credentials_svc" | grep -E "label|rotation|rotates|revision"
source ~/admin-openrc
openstack user list --domain service_domain
```

✅ **Attendu :** un secret `credentials_svc_<service>` par compte de service, en `monthly` ; et les comptes `svc_nova-…`, `svc_neutron-…` correspondants dans Keystone.

**Comment le lire :** chaque mois, le charm Keystone génère un nouveau mot de passe, le met dans Keystone, publie la nouvelle révision du secret, et Juju prévient le service concerné, qui l’utilise aussitôt. Personne ne voit ce mot de passe. C’est la « gestion de secrets » de la démo 2 : génération, stockage, rotation.

**Suivre une rotation :** dans les journaux du charm, le jour indiqué par `rotates` :

```bash
juju debug-log -m openstack --replay --no-tail --include keystone/0 | grep -i -E "rotat|secret" | tail -5
juju show-status-log -m openstack keystone/0 | grep -i secret | tail -3
```

⚠ **à vérifier :** le texte exact des lignes de journal ; on cherche une étape `secret-rotate` du charm, et l’augmentation de la révision du secret.

## 3.5 Les secrets qui ne tournent pas

Les secrets en `never` (bases MySQL, RabbitMQ, certificats internes) ne sont pas renouvelés automatiquement.

| Question | Réponse |
|---|---|
| Est-ce grave ? | Ils ne sortent pas du cluster : ils sont échangés entre charms par les relations Juju, chiffrés au repos par Juju. Le risque principal est l’accès à la machine ou au contrôleur Juju |
| Comment les protéger ? | Limiter l’accès à Lab-Sunbeam et au contrôleur, protéger les sauvegardes, auditer les lectures (`juju show-secret --reveal` est tracé par Juju) |
| Et en cas de compromission (R26) ? | Le renouvellement passe par les charms (recréer les relations concernées) : c’est une **opération de maintenance lourde**, à préparer avec la documentation de Canonical, pas à improviser |

## 3.6 Le mot de passe d’`admin`

⚠ **Ne pas le changer avec `openstack user set admin --password …`.** Sunbeam fournit ce mot de passe à la demande (`sunbeam openrc`, qui lit le compte via l’action `get-admin-account` du charm, guide `4-juju-cli/`, démo 6). Un changement fait à la main risquerait de laisser au charm l’**ancien** mot de passe : `sunbeam openrc` distribuerait alors des accès invalides (⚠ **à vérifier**, comportement non testé).

À faire à la place :

```bash
juju actions -m openstack keystone
```

⚠ **à vérifier :** une action du charm qui régénère le mot de passe administrateur. À défaut, traiter `admin` comme un **compte de secours** (démo 2) : rarement utilisé, tracé, et protégé par l’accès restreint à Lab-Sunbeam plutôt que par une rotation.

## 3.7 Les identifiants d’application : la rotation côté utilisateur

**À quoi ça sert :** les scripts des utilisateurs utilisent des identifiants d’application (démo 1, étape 1.10). Leur rotation est simple, et sans interruption : on crée le nouveau **avant** de supprimer l’ancien.

```bash
source ~/demo-openrc
openstack application credential create lab-sec-script-v1 --role member \
  --expiration "$(date -u -d '+90 days' +%Y-%m-%dT%H:%M:%S)" -f value -c id
```

Trois mois plus tard (ou en cas de doute sur sa confidentialité) :

```bash
openstack application credential create lab-sec-script-v2 --role member \
  --expiration "$(date -u -d '+90 days' +%Y-%m-%dT%H:%M:%S)" -f json > ~/lab-sec-ac-v2.json
chmod 600 ~/lab-sec-ac-v2.json
```

1. Mettre le nouvel identifiant et son secret dans le script (ou dans son `clouds.yaml`), vérifier que le script fonctionne ;
2. Supprimer l’ancien :

   ```bash
   openstack application credential delete lab-sec-script-v1
   openstack application credential list -c Name -c "Expires At"
   ```

✅ **Attendu :** seul `lab-sec-script-v2` reste, avec sa date d’expiration.

| Règle | Pourquoi |
|---|---|
| Toujours une **expiration** | Un identifiant oublié finit par mourir de lui-même |
| Le **moindre rôle** nécessaire (`reader` si le script ne fait que lire) | Un vol donne moins de droits |
| Créer le nouveau **avant** de supprimer l’ancien | Pas d’interruption du script |
| Le **secret** n’est affiché qu’une fois | Keystone n’en garde qu’une empreinte : le ranger aussitôt dans un coffre |

## Nettoyage

```bash
source ~/demo-openrc
openstack application credential delete lab-sec-script-v2 2>/dev/null
openstack application credential delete lab-sec-script-v1 2>/dev/null
rm -f ~/lab-sec-ac-v2.json ~/lab-sec-cles-j1.txt
```

## À retenir

- Sur Sunbeam, les secrets des services sont **générés, rangés et distribués par Juju** : on les inventorie avec `juju secrets`, sans jamais les révéler.
- Les **clés Fernet** et les **mots de passe des comptes de service** tournent **automatiquement** (`daily`, `monthly`), sans interruption, grâce aux clés en attente et secondaires.
- Les secrets en `never` se protègent par l’**accès** au nœud et au contrôleur ; leur renouvellement est une opération de maintenance planifiée.
- Côté utilisateurs : des **identifiants d’application** à durée limitée, renouvelés en recouvrement.

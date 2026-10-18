# Démo 1 : fédération d’identité, une autorité d’authentification externe

> **Statut : rédigé, pas encore rejoué sur le lab.** Les étapes 1.4 à 1.6 reprennent les sections 12 et 13 du guide `Guide-Sunbeam-VirtualBox-Keycloak.md`, en les expliquant. Support : module 11.

## Objectif

À la fin de la démo, un utilisateur de l’entreprise, `alice`, se connecte à Horizon **avec son compte Keycloak**, sans aucun mot de passe stocké dans OpenStack, et obtient exactement les droits prévus. On montre ensuite pourquoi c’est un gain de sécurité : la politique de mots de passe, le verrouillage et la désactivation des comptes se décident **à un seul endroit**.

## 1.1 Comprendre avant de faire

### Le principe

Sans fédération, Keystone est à la fois l’**annuaire** (il stocke les comptes et les mots de passe) et le **contrôle d’accès** (il donne les rôles). Avec la fédération, on sépare les deux :

| Rôle | Qui le tient | Ce qu’il fait |
|---|---|---|
| **Fournisseur d’identité** (*IdP*, *Identity Provider*) | Keycloak, dans Lab-Services | Vérifie l’identité : mot de passe, verrouillage, second facteur |
| **Fournisseur de service** (*SP*, *Service Provider*) | Keystone | Fait confiance à l’IdP, puis attribue les rôles OpenStack |

### Le trajet d’une connexion (SAML 2)

```text
 Navigateur            Horizon / Keystone (SP)                  Keycloak (IdP)
     │  1. « Se connecter avec Keycloak - Lab »  │                      │
     │──────────────────────────────────────────►│                      │
     │  2. redirection vers Keycloak, avec une demande SAML signée      │
     │◄──────────────────────────────────────────│                      │
     │  3. alice saisit son mot de passe ──────────────────────────────►│  politique de mots de passe,
     │                                                                   │  verrouillage, MFA : ICI
     │  4. Keycloak renvoie une assertion SAML signée (« c’est alice ») │
     │◄──────────────────────────────────────────────────────────────── │
     │  5. le navigateur la remet à Keystone ───►│                      │
     │                                           │ 6. Keystone vérifie la signature, applique le
     │                                           │    MAPPING : alice → groupe lab_members → rôle
     │  7. jeton Keystone, Horizon ouvert        │    member sur le projet formation
     │◄──────────────────────────────────────────│                      │
```

**Ce qu’il faut retenir :** le mot de passe d’`alice` ne passe **jamais** par OpenStack. Keystone ne reçoit qu’une assertion signée par Keycloak, dont il vérifie la signature avec le certificat de Keycloak.

### Les objets à créer

| Objet | Où | Rôle |
|---|---|---|
| Utilisateur `alice` | Keycloak, realm `lab` | La personne |
| Paire de clés SAML du SP | Keystone (via `sunbeam identity set-saml-x509`) | Keystone signe ses demandes ; Keycloak vérifie cette signature |
| Fournisseur d’identité `keycloak` | Keystone | « Je fais confiance aux assertions de ce Keycloak » |
| Client SAML | Keycloak | « Je réponds aux demandes de ce Keystone » |
| Domaine `keycloak` | Keystone | L’espace où sont rangés les comptes venus de Keycloak |
| Groupe `lab_members`, projet `formation` | Keystone | La cible des droits |
| **Mapping** | Keystone | La règle qui traduit « qui est l’utilisateur » en « dans quel groupe il entre » |
| Protocole `saml2` | Keystone | Relie le fournisseur d’identité au mapping |

## 1.2 Prérequis

| Prérequis | Comment le vérifier | Si non |
|---|---|---|
| L’adresse publique du cloud est en **HTTPS** | `sunbeam dashboard url` commence par `https://` | Démo 4, étape 4.3 |
| Keycloak tourne | Dans Lab-Services : `cd ~/lab/keycloak && docker compose ps` : `keycloak` et `postgres` `Up` | `docker compose up -d` |
| Les pods résolvent `sso.lab.test` | Guide Keycloak, section 10 (pod `lab-dns-check`) | Ajouter le bloc `lab.test` dans CoreDNS (même section) |
| Lab-Sunbeam fait confiance à la CA du lab | `curl --fail --cacert ~/ca.crt https://sso.lab.test:8443/realms/lab -o /dev/null && echo OK` | Guide Keycloak, section 11, premier bloc |

## 1.3 Créer l’utilisateur dans Keycloak

**À quoi ça sert :** dans un vrai déploiement, Keycloak serait relié à l’annuaire de l’entreprise (Active Directory). Ici, on crée la personne directement dans le realm `lab`.

1. Sur **Lab-Services**, lire le mot de passe de l’administrateur de Keycloak (sans l’afficher à l’écran projeté) :

   ```bash
   grep KEYCLOAK_ADMIN_PASSWORD ~/lab/keycloak/.env
   ```

2. Sur Windows, ouvrir **https://sso.lab.test:8443/admin/** et se connecter avec `kcadmin` et ce mot de passe. Pas d’alerte de certificat : la CA du lab est approuvée sur Windows.
3. En haut à gauche, choisir le realm **lab** (et non `master`, réservé à l’administration de Keycloak).
4. **Users → Add user** : *Username* `alice`, *Email* `alice@lab.test`, *First name* `Alice`, *Email verified* activé → **Create**.
5. Onglet **Credentials → Set password** : un mot de passe d’au moins 12 caractères, *Temporary* **désactivé** → **Save**.

✅ **Attendu :** `alice` apparaît dans **Users**, `Enabled`.

**Comment le lire :** *Temporary* activé obligerait `alice` à changer son mot de passe à la première connexion : c’est la bonne pratique en production (recommandation R30 du guide de l’ANSSI, démo 2), désactivée ici pour que la démo reste courte.

## 1.4 Déclarer Keycloak à Keystone

Sur **Lab-Sunbeam**. **À quoi ça sert :** Keystone doit pouvoir **signer** ses demandes SAML (paire de clés du SP), et savoir **où** se trouve Keycloak et **quelle CA** valide son certificat HTTPS.

```bash
umask 077
mkdir -p ~/saml
openssl req -x509 -newkey rsa:3072 -nodes -days 365 \
  -keyout ~/saml/sp.key -out ~/saml/sp.crt -subj '/CN=Keystone SAML Lab'
sunbeam identity set-saml-x509 ~/saml/sp.crt ~/saml/sp.key
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `umask 077` | Les fichiers créés ne seront lisibles que par toi : `sp.key` est une clé privée |
| `openssl req -x509 -newkey rsa:3072 -nodes -days 365` | Crée une clé RSA de 3 072 bits et un certificat auto-signé valable un an |
| `-subj '/CN=Keystone SAML Lab'` | Le nom inscrit dans le certificat |
| `sunbeam identity set-saml-x509` | Confie cette paire au charm Keystone. C’est l’option `saml-x509-keypair` de `juju config keystone`, de type `secret`, vue dans le guide `4-juju-cli/` |

⚠ Ce certificat sert à **signer les messages SAML**. Il est distinct du certificat HTTPS du cloud (démo 4).

Télécharger les **métadonnées** de Keycloak, qui serviront à l’étape 1.6 :

```bash
curl --fail --cacert ~/ca.crt https://sso.lab.test:8443/realms/lab/protocol/saml/descriptor -o ~/keycloak-idp.xml
grep -o 'entityID="[^"]*"' ~/keycloak-idp.xml
```

✅ **Attendu :** `entityID="https://sso.lab.test:8443/realms/lab"` (⚠ à vérifier) : l’identifiant de Keycloak, que Keystone exigera dans les assertions. `curl` doit réussir **sans** `-k` : la vérification du certificat de Keycloak est faite avec la CA du lab.

Puis déclarer le fournisseur d’identité :

```bash
cat > ~/keycloak-saml.yaml <<YAML
metadata-url: https://sso.lab.test:8443/realms/lab/protocol/saml/descriptor
ca-chain: $(base64 -w0 ~/ca.crt)
label: 'Keycloak - Lab'
YAML
sunbeam identity provider add generic saml2 keycloak --config ~/keycloak-saml.yaml
sunbeam identity provider list
```

| Morceau | Rôle |
|---|---|
| `metadata-url` | L’adresse des **métadonnées** SAML du realm : Keycloak y publie son identifiant, ses adresses et son certificat de signature |
| `ca-chain` | La CA du lab, encodée en base64, pour que Keystone accepte le certificat HTTPS de Keycloak |
| `label` | Le texte du bouton de connexion dans Horizon |
| `identity provider add generic saml2 keycloak` | Un fournisseur générique, en SAML 2, nommé `keycloak` |

✅ **Attendu :** `sunbeam identity provider list` affiche une ligne `keycloak` en plus de `Keystone Credentials` (seule ligne observée sur le lab avant la démo).

## 1.5 Déclarer Keystone à Keycloak

**À quoi ça sert :** la confiance doit être **réciproque**. Keycloak doit connaître le Keystone qui lui envoie des demandes, et l’adresse exacte où renvoyer l’assertion.

Télécharger les métadonnées du SP Keystone :

```bash
source ~/admin-openrc
curl --fail --cacert "$OS_CACERT" \
  'https://cloud.lab.test/openstack-keystone/v3/OS-FEDERATION/identity_providers/keycloak/protocols/saml2/auth/mellon/metadata' \
  -o ~/keystone-sp.xml
python3 ~/lab/scripts/extract-sp.py ~/keystone-sp.xml
```

`extract-sp.py` affiche l’**Entity ID** (l’identifiant de Keystone) et l’**ACS** (*Assertion Consumer Service*, l’adresse où Keycloak doit poster l’assertion). Ne pas les deviner : les recopier.

⚠ **à vérifier :** `OS_CACERT` doit être défini par l’openrc **après** l’activation du TLS (`sunbeam openrc > ~/admin-openrc`, démo 4). Le script `extract-sp.py` est dans `~/lab/scripts/` sur la VM où le dépôt a été copié ; sinon, prendre `lab-sunbeam/scripts/extract-sp.py` du dépôt.

Dans Keycloak, realm **lab** : **Clients → Import client**, choisir `keystone-sp.xml` (copié sur Windows), **Save**. Puis contrôler les réglages du client, tableau de la section 12 du guide Keycloak : *Name ID format* `persistent`, *Force POST binding*, signatures des documents et des assertions activées, `RSA_SHA256`, et la clé publique du client égale à `~/saml/sp.crt`.

## 1.6 Donner des droits aux comptes fédérés

**À quoi ça sert :** la fédération dit **qui** est l’utilisateur ; elle ne donne **aucun** droit. On décide maintenant **où** il entre, toujours selon le moindre privilège : un groupe, un projet, le rôle `member`, jamais `admin`.

```bash
source ~/admin-openrc
openstack domain create keycloak --description "Comptes federes depuis Keycloak"
openstack group create --domain keycloak lab_members
openstack project create --domain keycloak formation
openstack role add --group lab_members --group-domain keycloak --project formation --project-domain keycloak member
```

Le **mapping** traduit l’assertion en groupe. Celui du lab (`lab-sunbeam/mapping.json`) :

```json
[
  {
    "local": [
      { "user":  { "name": "{0}" },
        "group": { "domain": { "name": "keycloak" }, "name": "lab_members" } }
    ],
    "remote": [ { "type": "REMOTE_USER" } ]
  }
]
```

**Comment le lire :** pour toute assertion qui contient un `REMOTE_USER` (l’identifiant transmis par Keycloak), créer un utilisateur local nommé comme cet identifiant (`{0}` = la première valeur de `remote`), et le placer dans le groupe `lab_members` du domaine `keycloak`. **Tout** utilisateur authentifié par ce Keycloak entre donc dans ce groupe : simple pour un lab, trop large en entreprise (voir 1.9).

```bash
REMOTE_ID=$(python3 -c 'import xml.etree.ElementTree as E; print(E.parse("/home/vagrant/keycloak-idp.xml").getroot().attrib["entityID"])')
openstack identity provider create --remote-id "$REMOTE_ID" --domain keycloak keycloak
openstack mapping create --rules ~/lab/mapping.json keycloak_saml
openstack federation protocol create --identity-provider keycloak --mapping keycloak_saml saml2
openstack federation protocol list --identity-provider keycloak
openstack role assignment list --group lab_members --group-domain keycloak --names
```

| Commande | Rôle |
|---|---|
| `REMOTE_ID=…` | Lit l’identifiant exact de Keycloak (`entityID`) dans ses métadonnées (`~/keycloak-idp.xml`, téléchargé à l’étape 1.4) |
| `identity provider create --remote-id … --domain keycloak keycloak` | Les assertions dont l’émetteur est ce Keycloak sont acceptées, et leurs utilisateurs rangés dans le domaine `keycloak` |
| `mapping create` | Enregistre la règle de traduction |
| `federation protocol create … saml2` | Pour ce fournisseur, en SAML 2, appliquer ce mapping |

⚠ **à vérifier :** `sunbeam identity provider add` (1.4) peut avoir déjà créé l’objet `keycloak` dans Keystone. Si `identity provider create` répond `Conflict`, afficher l’existant (`openstack identity provider show keycloak`), vérifier que son `remote_ids` et son domaine conviennent, et passer à la commande suivante.

✅ **Attendu :** un protocole `saml2` relié au mapping `keycloak_saml`, et `lab_members@keycloak` → `member` sur `formation@keycloak`.

## 1.7 Tester la connexion

1. Sur Windows, ouvrir une **fenêtre de navigation privée** (pour ne pas réutiliser une session `admin`), à l’adresse donnée par `sunbeam dashboard url`.
2. Méthode d’authentification : **Keycloak - Lab** → **Se connecter**.
3. La page de Keycloak s’affiche : saisir `alice` et son mot de passe.
4. Retour dans Horizon : projet **formation**.

✅ **Attendu :** Horizon ouvert sur `formation`, avec les menus d’un `member` (pas de menu Admin).

Côté OpenStack, ce qui a été créé :

```bash
openstack user list --domain keycloak
openstack role assignment list --group lab_members --group-domain keycloak --names --effective
```

✅ **Attendu :** un utilisateur dans le domaine `keycloak`, dont le nom est l’identifiant transmis par Keycloak. Avec le format `persistent`, ce n’est pas forcément le texte `alice`, mais un identifiant stable : c’est normal. C’est un **utilisateur fantôme** (*shadow user*) : Keystone garde sa trace pour les affectations et l’audit, mais **aucun mot de passe**.

## 1.8 Ce que la fédération change pour la sécurité

**Désactiver une personne, à un seul endroit.** Dans Keycloak : **Users → alice → Enabled : Off → Save**. Puis retenter la connexion (étapes 1 à 3 de 1.7).

✅ **Attendu :** Keycloak refuse la connexion ; Horizon n’est pas atteint.

⚠ **Piège :** un jeton Keystone déjà délivré reste valable jusqu’à son expiration (une heure sur le lab, TP 1 des `labs-iam/`, § 1.4.7). Pour un départ urgent : désactiver dans Keycloak **et** révoquer les jetons en cours (`openstack token revoke`, ou désactiver l’utilisateur fantôme dans Keystone le temps de l’expiration).

Réactiver `alice` (*Enabled : On*) pour la suite.

| Sans fédération | Avec fédération |
|---|---|
| Un mot de passe de plus par personne, stocké dans Keystone | Aucun mot de passe dans OpenStack |
| Politique de mots de passe : options de Keystone, **non exposées par Sunbeam** (démo 2) | Politique de Keycloak : longueur, historique, verrouillage, second facteur (démo 2) |
| Départ d’un salarié : penser à chaque application | Un seul compte à désactiver |
| Second facteur : à construire dans Keystone | Natif dans Keycloak (TOTP, WebAuthn) |

## 1.9 En entreprise : affiner le mapping

Le mapping du lab donne `member` à **toute** personne authentifiée par Keycloak. En entreprise, on filtre sur un **groupe** de l’annuaire, transmis dans l’assertion. Exemple de règle (⚠ **à vérifier** : le nom exact de l’attribut dépend du *mapper* de groupes configuré dans le client Keycloak) :

```json
[
  {
    "local": [
      { "user":  { "name": "{0}" },
        "group": { "domain": { "name": "keycloak" }, "name": "lab_members" } }
    ],
    "remote": [
      { "type": "REMOTE_USER" },
      { "type": "MELLON_groups", "any_one_of": ["openstack-formation"] }
    ]
  }
]
```

**Comment le lire :** seuls les utilisateurs membres du groupe Keycloak `openstack-formation` entrent dans `lab_members`. Les autres sont authentifiés par Keycloak, mais Keystone ne leur donne **rien**.

## 1.10 La ligne de commande pour un utilisateur fédéré

La connexion par le navigateur ne donne pas de mot de passe utilisable par la commande `openstack`. Pour un script, l’utilisateur fédéré crée, depuis Horizon (**Identité → Identifiants d’application**), un **identifiant d’application** limité au projet, avec une date d’expiration (`labs-iam/04-roles-moindre-privilege.md`, § 4.5.2). C’est aussi ce qui évite de partager un mot de passe.

## Retour arrière

Dans l’ordre inverse de la création :

```bash
source ~/admin-openrc
openstack federation protocol delete --identity-provider keycloak saml2
openstack mapping delete keycloak_saml
openstack identity provider delete keycloak
openstack role remove --group lab_members --group-domain keycloak --project formation --project-domain keycloak member
openstack project delete formation --domain keycloak
openstack group delete lab_members --domain keycloak
openstack domain set --disable keycloak
openstack domain delete keycloak
```

Puis, côté Sunbeam (⚠ **à vérifier** : nom exact de la sous-commande avec `sunbeam identity provider --help`) :

```bash
sunbeam identity provider remove keycloak
sunbeam identity provider list
```

Et dans Keycloak : supprimer le client SAML de Keystone et l’utilisateur `alice`. Si un instantané VirtualBox a été pris avant la démo, le restaurer est plus sûr.

## À retenir

- La fédération **sépare** la vérification de l’identité (Keycloak) des droits (Keystone).
- Le mot de passe des personnes ne passe plus par OpenStack ; la politique et la désactivation des comptes se gèrent **à un seul endroit**.
- La fédération ne donne aucun droit : c’est le **mapping**, les **groupes** et les **affectations** qui décident, selon le moindre privilège.
- Le compte `admin` local de Keystone reste indispensable pour réparer une fédération en panne : c’est un compte de **secours**, à protéger particulièrement (démo 2).

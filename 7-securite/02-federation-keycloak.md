# Démo 2 : fédération d’identité, une autorité d’authentification externe

> **Statut : rejouée sur le lab le 18 octobre 2026** (Sunbeam 2024.1, snap rev 1101 ; charms `keystone-k8s` rev 440, `keystone-saml-k8s` rev 134). Les sorties citées sont celles de ce jour. Les manipulations dans la console de Keycloak ont été faites par son API, avec les mêmes réglages. Retour arrière joué le même jour. Non observé : l’affichage dans le navigateur (2.7). Support : module 11.

## Objectif

À la fin de la démo, un utilisateur de l’entreprise, `alice`, obtient un jeton OpenStack **avec son compte Keycloak**, sans aucun mot de passe stocké dans OpenStack, et exactement les droits prévus. (La connexion par Horizon bute sur une limite du charm Keystone de ce lab : étape 2.7.) On montre ensuite pourquoi c’est un gain de sécurité : la politique de mots de passe, le verrouillage et la désactivation des comptes se décident **à un seul endroit**.

## 2.1 Comprendre avant de faire

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
| Application `keystone-idp-saml2-keycloak` (charm `keystone-saml-k8s`) | Modèle Juju `openstack` | Donne à Keystone les métadonnées et la CA de Keycloak, et configure le module SAML d’Apache |
| Fournisseur d’identité `keycloak` | Keystone | « Je fais confiance aux assertions de ce Keycloak » |
| Client SAML | Keycloak | « Je réponds aux demandes de ce Keystone » |
| Domaine `keycloak` | Keystone | L’espace où sont rangés les comptes venus de Keycloak |
| Groupe `lab_members`, projet `formation` | Keystone | La cible des droits |
| **Mapping** | Keystone | La règle qui traduit « qui est l’utilisateur » en « dans quel groupe il entre » |
| Protocole `saml2` | Keystone | Relie le fournisseur d’identité au mapping |

## 2.2 Prérequis

| Prérequis | Comment le vérifier | Si non |
|---|---|---|
| L’adresse publique du cloud est en **HTTPS** | `sunbeam dashboard url` commence par `https://` | Démo 1, étape 1.3 |
| Keycloak tourne | Dans Lab-Services : `cd ~/lab/keycloak && docker compose ps` : `keycloak` et `postgres` `Up` | `docker compose up -d` |
| Le pod Keystone résout `sso.lab.test` | `sudo k8s kubectl -n openstack exec keystone-0 -c keystone -- getent hosts sso.lab.test` affiche `192.168.56.20   sso.lab.test` | Guide Keycloak, section 10 : le DNS des pods (CoreDNS) doit interroger Lab-Services |
| La CA du lab est sur Lab-Sunbeam, en `~/ca.crt` | `openssl x509 -in ~/ca.crt -noout -subject` affiche `CN = Sunbeam Lab Root CA` | Depuis Windows, dans le dossier du dépôt : `scp .\ca.crt vagrant@192.168.56.10:~/` (constaté le 18 octobre : le fichier n’y était pas) |
| Lab-Sunbeam joint Keycloak en vérifiant son certificat | `curl -s --fail --cacert ~/ca.crt https://sso.lab.test:8443/realms/lab -o /dev/null && echo OK` affiche `OK` | Vérifier que Keycloak tourne et que `sso.lab.test` est résolu (`getent hosts sso.lab.test`) |

**Rappel (démo 1, étape 1.2) :** deux CA coexistent. La CA du **lab** (`~/ca.crt`) signe le certificat de Keycloak ; la CA de **Sunbeam** (`$OS_CACERT`) signe celui du cloud. Chaque commande `curl` de cette démo utilise la bonne.

## 2.3 Créer l’utilisateur dans Keycloak

**À quoi ça sert :** dans un vrai déploiement, Keycloak serait relié à l’annuaire de l’entreprise (Active Directory). Ici, on crée la personne directement dans le realm `lab`.

1. Sur **Lab-Services**, lire le mot de passe de l’administrateur de Keycloak (sans l’afficher à l’écran projeté) :

   ```bash
   grep KEYCLOAK_ADMIN_PASSWORD ~/lab/keycloak/.env
   ```

2. Sur Windows, ouvrir **https://sso.lab.test:8443/admin/** et se connecter avec `kcadmin` et ce mot de passe. Pas d’alerte de certificat : la CA du lab est approuvée sur Windows.
3. En haut à gauche, choisir le realm **lab** (et non `master`, réservé à l’administration de Keycloak).
4. **Users → Add user** : *Username* `alice`, *Email* `alice@lab.test`, *First name* `Alice`, ***Last name*** `Martin`, *Email verified* activé → **Create**.
5. Onglet **Credentials → Set password** : un mot de passe d’au moins 12 caractères, *Temporary* **désactivé** → **Save**.

✅ **Attendu :** `alice` apparaît dans **Users**, `Enabled`.

⚠ **Ne pas oublier le nom de famille** (constaté le 18 octobre) : Keycloak 26 l’exige. Sans lui, après le mot de passe, Keycloak affiche *Update Account Information* et demande de compléter le profil au lieu de renvoyer `alice` vers OpenStack.

**Comment le lire :** *Temporary* activé obligerait `alice` à changer son mot de passe à la première connexion : c’est la bonne pratique en production (recommandation R30 du guide de l’ANSSI, démo 3), désactivée ici pour que la démo reste courte.

## 2.4 Déclarer Keycloak à Keystone

Sur **Lab-Sunbeam**. **À quoi ça sert :** Keystone doit pouvoir **signer** ses demandes SAML (paire de clés du SP), et savoir **où** se trouve Keycloak et **quelle CA** valide son certificat HTTPS.

### La paire de clés SAML de Keystone

```bash
umask 077
mkdir -p ~/saml
openssl req -x509 -newkey rsa:3072 -nodes -days 365 \
  -keyout ~/saml/sp.key -out ~/saml/sp.crt -subj '/CN=Keystone SAML Lab'
time sunbeam identity set-saml-x509 ~/saml/sp.crt ~/saml/sp.key
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `umask 077` | Les fichiers créés ne seront lisibles que par toi : `sp.key` est une clé privée |
| `openssl req -x509 -newkey rsa:3072 -nodes -days 365` | Crée une clé RSA de 3 072 bits et un certificat auto-signé valable un an |
| `-subj '/CN=Keystone SAML Lab'` | Le nom inscrit dans le certificat |
| `sunbeam identity set-saml-x509` | Confie cette paire au charm Keystone. C’est l’option `saml-x509-keypair` de `juju config keystone`, de type `secret`, vue dans le guide `4-juju-cli/` |

✅ **Attendu** (18 octobre) : la commande rend la main sans message, en 45 secondes environ ; `juju config -m openstack keystone saml-x509-keypair` affiche alors une référence `secret:…`.

⚠ Ce certificat sert à **signer les messages SAML**. Il est distinct du certificat HTTPS du cloud (démo 1).

### Les métadonnées de Keycloak

```bash
curl -s --fail --cacert ~/ca.crt https://sso.lab.test:8443/realms/lab/protocol/saml/descriptor -o ~/keycloak-idp.xml
grep -o 'entityID="[^"]*"' ~/keycloak-idp.xml
```

✅ **Attendu** (18 octobre) : `entityID="https://sso.lab.test:8443/realms/lab"`, l’identifiant de Keycloak, que Keystone exigera dans les assertions. `curl` réussit **sans** `-k` : la vérification du certificat de Keycloak est faite avec la CA du lab.

### Brancher Keycloak : avec Juju, pas avec Sunbeam

La commande prévue par Sunbeam est `sunbeam identity provider add generic saml2 keycloak --config <fichier>`. **Sur ce lab (Sunbeam 2024.1, snap rev 1101), elle échoue**, essayée le 18 octobre :

| Essai | Résultat |
|---|---|
| `ca-chain:` la CA du lab en base64 | `Error: Failed to validate IDP a bytes-like object is required, not 'str'` |
| `ca-chain: ''` (vide) | `Error: invalid state for provider step` |
| `ca-chain: !!binary …` | `Error: Object of type bytes is not JSON serializable` |

**Pourquoi :** dans le code de Sunbeam (`sunbeam/steps/sso.py`), la validation écrit la valeur de `ca-chain` (du texte) dans un fichier ouvert en mode binaire, qui n’accepte que des octets : toute CA fait échouer la validation. Et une valeur vide est refusée plus loin, car toutes les options doivent être remplies. Sans `ca-chain`, Keystone ne pourrait de toute façon pas vérifier le certificat de Keycloak, signé par la CA du lab. Avec un Keycloak signé par une autorité publique, le problème ne se poserait pas.

⚠ Après un échec, Sunbeam garde les réponses saisies (élément `FeatureSSOExternalIDPConfig-saml2-keycloak` de sa base) et les repropose comme valeurs par défaut aux essais suivants : une clé absente du fichier n’est donc **pas** vide.

**Ce que ferait Sunbeam**, lu dans son plan Terraform (`/snap/openstack/current/etc/deploy-openstack/main.tf`, ressource `sso-saml2-providers`) : déployer le charm `keystone-saml-k8s` sous le nom `keystone-idp-saml2-<nom>`, avec la configuration du fichier, et l’intégrer à `keystone`. On le fait donc directement avec Juju :

```bash
tr -d '\r' < ~/ca.crt > ~/ca-lf.crt
juju deploy -m openstack keystone-saml-k8s keystone-idp-saml2-keycloak --channel 2024.1/stable \
  --config name=keycloak \
  --config label='Keycloak - Lab' \
  --config metadata-url=https://sso.lab.test:8443/realms/lab/protocol/saml/descriptor \
  --config ca-chain="$(base64 -w0 ~/ca-lf.crt)
$(cat ~/ca-lf.crt)"
juju integrate -m openstack keystone-idp-saml2-keycloak:keystone-saml keystone:keystone-saml
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `tr -d '\r'` | Retire les fins de ligne Windows du fichier `ca.crt` copié depuis le dépôt |
| `keystone-saml-k8s` | Le charm qui déclare un fournisseur SAML à Keystone (constaté : révision 134 en `2024.1/stable`) |
| `name=keycloak` | L’identifiant du fournisseur dans Keystone : il apparaît dans toutes les adresses de fédération |
| `label` | Le texte proposé dans la liste de connexion de Horizon |
| `metadata-url` | L’adresse des **métadonnées** SAML du realm : Keycloak y publie son identifiant, ses adresses et son certificat de signature |
| `ca-chain` | La CA du lab, pour que le charm accepte le certificat HTTPS de Keycloak (voir ci-dessous) |
| `juju integrate … keystone-saml` | Relie le fournisseur à Keystone, qui configure alors le module SAML d’Apache (`mod_auth_mellon`) |

**Pourquoi ce `ca-chain` étrange, base64 puis texte ?** Le charm a, lui aussi, un défaut en révision 134 : la description de l’option et le code qui télécharge les métadonnées attendent la CA **en base64** ; mais le contrôle de validité cherche un bloc `-----BEGIN CERTIFICATE-----` **en clair** dans la même valeur. Avec le base64 seul, le charm se bloque sur `Invalid ca-chain in config` (constaté). En mettant les deux, le contrôle trouve le bloc en clair, et le décodage base64 s’arrête au `==` final : il rend exactement la CA. Lu dans le code du charm (`src/charm.py`, `src/certs.py`, dans le conteneur `charm` du pod `keystone-idp-saml2-keycloak-0`).

✅ **Attendu** (18 octobre, environ 10 minutes) :

```bash
juju status -m openstack keystone-idp-saml2-keycloak keystone horizon
```

```text
horizon/0*                      active    idle   10.1.0.208
keystone-idp-saml2-keycloak/0*  active    idle   10.1.0.194         Provider is ready
keystone/0*                     active    idle   10.1.0.79
```

Les états intermédiaires, dans l’ordre : `waiting … installing agent`, `waiting … Waiting for keystone to set SP URLs`, puis `Provider is ready`.

| Message bloquant | Cause |
|---|---|
| `Invalid ca-chain in config` | Valeur en base64 seul : reprendre la commande `juju config … ca-chain=` ci-dessus |
| `Failed to get IDP metadata` | Keycloak injoignable depuis le pod, ou CA incorrecte |

**Comment le lire :** `sunbeam identity provider list` continue de n’afficher que `Keystone Credentials` : Sunbeam ne connaît pas ce fournisseur, ajouté sans lui. Le retour arrière se fera donc aussi avec Juju.

## 2.5 Déclarer Keystone à Keycloak

**À quoi ça sert :** la confiance doit être **réciproque**. Keycloak doit connaître le Keystone qui lui envoie des demandes, et l’adresse exacte où renvoyer l’assertion.

Télécharger les métadonnées du SP Keystone :

```bash
source ~/admin-openrc
curl -s --fail --cacert "$OS_CACERT" \
  'https://cloud.lab.test/openstack-keystone/v3/OS-FEDERATION/identity_providers/keycloak/protocols/saml2/auth/mellon/metadata' \
  -o ~/keystone-sp.xml
python3 ~/lab/scripts/extract-sp.py ~/keystone-sp.xml
```

`OS_CACERT` est défini par l’openrc régénéré après l’activation du TLS (démo 1, étape 1.3). Si le script `~/lab/scripts/extract-sp.py` est absent de Lab-Sunbeam, le copier depuis le dépôt : `scp .\lab-sunbeam\scripts\extract-sp.py vagrant@192.168.56.10:~/` (depuis Windows), puis `python3 ~/extract-sp.py ~/keystone-sp.xml`.

✅ **Attendu** (18 octobre) :

```text
Client ID / Entity ID: urn:saml2:keycloak
ACS urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST https://cloud.lab.test/openstack-keystone/v3/OS-FEDERATION/identity_providers/keycloak/protocols/saml2/auth/mellon/postResponse
SLO urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect https://cloud.lab.test/openstack-keystone/v3/OS-FEDERATION/identity_providers/keycloak/protocols/saml2/auth/mellon/logout
SP X509 certificate (base64): MIIEGTCC…
```

| Ligne | Ce que c’est |
|---|---|
| `Entity ID` | L’identifiant de Keystone auprès de Keycloak : `urn:saml2:` suivi du nom du fournisseur |
| `ACS` (*Assertion Consumer Service*) | L’adresse où Keycloak fait poster l’assertion par le navigateur |
| `SLO` (*Single Logout*) | L’adresse de déconnexion |
| `SP X509 certificate` | Le certificat `~/saml/sp.crt` de l’étape 2.4 : Keycloak s’en servira pour vérifier les demandes de Keystone |

Copier `keystone-sp.xml` sur Windows (PowerShell, dans le dossier du dépôt) :

```powershell
scp vagrant@192.168.56.10:~/keystone-sp.xml .
```

Dans Keycloak, realm **lab** : **Clients → Import client**, choisir `keystone-sp.xml`, **Save**. Puis contrôler les réglages du client :

| Réglage Keycloak | Valeur | Pourquoi |
|---|---|---|
| *Client ID* | `urn:saml2:keycloak`, l’`Entity ID` affiché par `extract-sp.py` | C’est par lui que Keycloak reconnaît Keystone |
| *Valid redirect URIs* | L’adresse **ACS** exacte, sans joker | Keycloak ne renvoie l’assertion qu’à cette adresse : pas de redirection vers un site tiers |
| *Master SAML Processing URL* | La même adresse ACS | Adresse par défaut des réponses |
| *Name ID format* | `persistent`, avec *Force Name ID format* activé | Un identifiant stable de l’utilisateur |
| *Force POST binding* | activé | L’assertion est postée par le navigateur |
| *Sign documents*, *Sign assertions* | activés | Keystone vérifie que l’assertion vient bien de Keycloak et n’a pas été modifiée |
| *Signature algorithm* | `RSA_SHA256` | Algorithme de signature actuel |
| *Client signature required* | activé, avec le certificat de `~/saml/sp.crt` dans l’onglet *Keys* | Keycloak vérifie que la demande vient bien de Keystone |
| *Encrypt assertions* | **à désactiver** | L’import l’active (constaté : `saml.encrypt = true` après conversion du fichier) ; le lab ne chiffre pas les assertions, le transport est déjà en HTTPS |

⚠ La clé publique du client doit être celle de `~/saml/sp.crt` (signature SAML), **pas** une CA TLS.

## 2.6 Donner des droits aux comptes fédérés

**À quoi ça sert :** la fédération dit **qui** est l’utilisateur ; elle ne donne **aucun** droit. On décide maintenant **où** il entre, toujours selon le moindre privilège : un groupe, un projet, le rôle `member`, jamais `admin`.

```bash
source ~/admin-openrc
openstack domain create keycloak --description "Comptes federes depuis Keycloak"
openstack group create --domain keycloak lab_members
openstack project create --domain keycloak formation
openstack role add --group lab_members --group-domain keycloak --project formation --project-domain keycloak member
```

Le **mapping** traduit l’assertion en groupe. Celui du lab, `lab-sunbeam/mapping.json` dans le dépôt, copié par Vagrant dans `~/lab/mapping.json` sur Lab-Sunbeam (sinon : `scp .\lab-sunbeam\mapping.json vagrant@192.168.56.10:~/lab/` depuis Windows) :

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

**Comment le lire :** pour toute assertion qui contient un `REMOTE_USER` (l’identifiant transmis par Keycloak), créer un utilisateur local nommé comme cet identifiant (`{0}` = la première valeur de `remote`), et le placer dans le groupe `lab_members` du domaine `keycloak`. **Tout** utilisateur authentifié par ce Keycloak entre donc dans ce groupe : simple pour un lab, trop large en entreprise (voir 2.9).

```bash
REMOTE_ID=$(python3 -c 'import xml.etree.ElementTree as E; print(E.parse("/home/vagrant/keycloak-idp.xml").getroot().attrib["entityID"])')
echo "$REMOTE_ID"
openstack identity provider create --remote-id "$REMOTE_ID" --domain keycloak keycloak
openstack mapping create --rules ~/lab/mapping.json keycloak_saml
openstack federation protocol create --identity-provider keycloak --mapping keycloak_saml saml2
openstack federation protocol list --identity-provider keycloak
openstack role assignment list --group lab_members --group-domain keycloak --names
```

| Commande | Rôle |
|---|---|
| `REMOTE_ID=…` | Lit l’identifiant exact de Keycloak (`entityID`) dans ses métadonnées (`~/keycloak-idp.xml`, téléchargé à l’étape 2.4) |
| `identity provider create --remote-id … --domain keycloak keycloak` | Les assertions dont l’émetteur est ce Keycloak sont acceptées, et leurs utilisateurs rangés dans le domaine `keycloak`. Le nom `keycloak` doit être celui de l’option `name` du charm (2.4) |
| `mapping create` | Enregistre la règle de traduction |
| `federation protocol create … saml2` | Pour ce fournisseur, en SAML 2, appliquer ce mapping |

✅ **Attendu** (18 octobre) : `https://sso.lab.test:8443/realms/lab`, un fournisseur `keycloak` avec ce `remote_ids`, puis :

```text
+-------+---------------+
| id    | mapping       |
+-------+---------------+
| saml2 | keycloak_saml |
+-------+---------------+
+--------+------+----------------------+--------------------+--------+--------+-----------+
| Role   | User | Group                | Project            | Domain | System | Inherited |
+--------+------+----------------------+--------------------+--------+--------+-----------+
| member |      | lab_members@keycloak | formation@keycloak |        |        | False     |
+--------+------+----------------------+--------------------+--------+--------+-----------+
```

Le charm ne crée pas l’objet `keycloak` dans Keystone (`openstack identity provider list` était vide avant ces commandes) : pas de conflit.

## 2.7 Tester la connexion

### La limite de Horizon sur ce lab

Horizon propose bien le nouveau choix (constaté : `curl -s --cacert "$OS_CACERT" https://cloud.lab.test/openstack-horizon/auth/login/ | grep -o "<option[^>]*>[^<]*</option>"` affiche `Keystone Credentials` et `Keycloak - Lab`). Mais la connexion par ce choix **échoue** : après le mot de passe, Keystone répond `401`, et son journal indique :

```text
ERROR keystone.api.auth […] https://cloud.lab.test/openstack-horizon/auth/websso/ is not a trusted dashboard host
```

**Pourquoi :** Keystone ne renvoie un jeton qu’aux adresses de Horizon inscrites dans `[federation] trusted_dashboard`. Horizon envoie bien son adresse à Keystone (relation `trusted-dashboard`). Mais le modèle du charm Keystone (révision 440, fichier `src/templates/section-federation`) n’écrit cette section que s’il existe un fournisseur **OpenID Connect** ; avec des fournisseurs SAML seuls, elle reste absente de `keystone.conf`. Constaté et lu dans le code le 18 octobre. Ce n’est pas réparable à la main : le charm réécrit `keystone.conf`.

On teste donc la fédération **directement sur Keystone**, ce qui prouve toute la chaîne de confiance.

### Dans le navigateur

Prérequis : la CA de Sunbeam importée sur Windows (démo 1, étape 1.3).

1. Ouvrir une **fenêtre de navigation privée**, à l’adresse :
   `https://cloud.lab.test/openstack-keystone/v3/OS-FEDERATION/identity_providers/keycloak/protocols/saml2/auth`
2. La page de Keycloak s’affiche : saisir `alice` et son mot de passe.
3. Retour sur Keystone : la page affiche le **jeton** au format JSON.

✅ **Attendu :** dans le JSON, `"methods": ["saml2"]`, le domaine `keycloak`, et `"OS-FEDERATION"` avec le fournisseur `keycloak` et le protocole `saml2`. (Trajet joué par le script ci-dessous le 18 octobre ; l’affichage dans le navigateur n’a pas été observé.)

### En ligne de commande, sans navigateur

Le script `7-securite/outils/test-sso-saml.py` du dépôt joue le même trajet qu’un navigateur. Le copier sur Lab-Sunbeam (`scp .\7-securite\outils\test-sso-saml.py vagrant@192.168.56.10:~/` depuis Windows), puis :

```bash
source ~/admin-openrc
python3 ~/test-sso-saml.py alice
```

✅ **Attendu** (18 octobre) :

```text
Mot de passe Keycloak de alice :
1. Redirigé vers : https://sso.lab.test:8443/realms/lab/protocol/saml
2. Assertion SAML reçue de Keycloak, postée vers : OS-FEDERATION/identity_providers/keycloak/protocols/saml2/auth/mellon/postResponse
3. Réponse de Keystone : 201
4. Jeton non limité à un projet : gAAAAABq…, expire 2026-10-18T11:03:41.000000Z
   Utilisateur : G-50829266-f3a1-4e25-95ec-a0a551b573c0 | domaine : keycloak
   Méthode : ['saml2'] | fournisseur : keycloak | groupes (id) : ['2e13f063…']
```

**Comment le lire :** les étapes 1 à 3 sont celles du schéma de 2.1. Le mot de passe a été vérifié par Keycloak ; Keystone n’a reçu qu’une assertion signée. Le jeton est rangé dans `~/lab-sec-fed-token` (droits `600`).

**Ce que ce jeton permet :** l’échanger contre un jeton limité au projet `formation`, puis agir en `member`.

```bash
T=$(cat ~/lab-sec-fed-token)
F() { env -u OS_USERNAME -u OS_PASSWORD -u OS_USER_DOMAIN_NAME -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME \
      openstack --os-auth-type token --os-token "$T" --os-project-name formation --os-project-domain-name keycloak "$@"; }
F token issue -c project_id -c expires
F network list -c Name
F user list
```

✅ **Attendu** (18 octobre) : un jeton sur le projet `formation`, la liste des réseaux (`external-network`), puis un refus : `ForbiddenException: 403 … identity:list_users`. `alice` a les droits d’un `member`, rien de plus.

**La fonction `F`, mot par mot :** `env -u …` retire les variables de l’administrateur chargées par `admin-openrc` ; `--os-auth-type token --os-token "$T"` s’authentifie avec le jeton fédéré au lieu d’un mot de passe ; `--os-project-name formation --os-project-domain-name keycloak` demande un jeton limité à ce projet.

Côté administrateur, ce qui a été créé :

```bash
openstack user list --domain keycloak
```

✅ **Attendu** (18 octobre) : un utilisateur `G-50829266-f3a1-4e25-95ec-a0a551b573c0`. C’est l’identifiant **persistant** transmis par Keycloak (format `persistent` de 2.5), pas le texte `alice` : stable, et sans rapport visible avec la personne. C’est un **utilisateur fantôme** (*shadow user*) : Keystone garde sa trace pour les affectations et l’audit, mais **aucun mot de passe**.

⚠ Ne pas chercher ses droits avec `openstack role assignment list --group lab_members --group-domain keycloak --effective` : Keystone refuse (`400 … Combining effective and group filter will always result in an empty list`). Son appartenance au groupe n’est pas enregistrée : elle est recalculée par le mapping **à chaque connexion**.

## 2.8 Ce que la fédération change pour la sécurité

**Désactiver une personne, à un seul endroit.** Dans Keycloak : **Users → alice → Enabled : Off → Save**. Puis retenter la connexion (2.7).

✅ **Attendu** (18 octobre) : Keycloak refuse : `Account is disabled, contact your administrator.` Keystone n’est jamais atteint.

⚠ **Piège 1 : un jeton déjà délivré reste valable**, jusqu’à son expiration (une heure sur le lab, TP 1 des `labs-iam/`, § 1.4.7). Constaté : le jeton obtenu avant la désactivation fonctionne encore (`F token issue` réussit). Pour un départ urgent, le révoquer côté Keystone :

```bash
openstack token revoke "$(cat ~/lab-sec-fed-token)"
F token issue
```

✅ **Attendu :** `Failed to validate token (HTTP 404)`.

⚠ **Piège 2 : désactiver l’utilisateur fantôme dans Keystone ne suffit pas.** Essayé le 18 octobre :

```bash
FED=$(openstack user list --domain keycloak -f value -c ID | head -1)
openstack user set --disable "$FED"
```

| Effet | Constaté |
|---|---|
| Jetons déjà délivrés | **Coupés** (`Failed to validate token (HTTP 404)`) |
| Nouvelle connexion fédérée | **Acceptée** (`201`) : Keystone délivre un jeton, et `F network list` répond, alors que `openstack user show "$FED" -c enabled` affiche `False` |

**Comment le lire :** pour un compte fédéré, Keystone ne consulte pas l’état du fantôme au moment de la connexion. La seule désactivation qui bloque la personne est celle de **l’autorité d’authentification**. Procédure de départ : désactiver dans Keycloak, **puis** couper les jetons en cours (révocation, ou désactivation du fantôme).

Réactiver : `openstack user set --enable "$FED"`, et dans Keycloak *Enabled : On*.

| Sans fédération | Avec fédération |
|---|---|
| Un mot de passe de plus par personne, stocké dans Keystone | Aucun mot de passe dans OpenStack |
| Politique de mots de passe : options de Keystone, **non exposées par Sunbeam** (démo 3) | Politique de Keycloak : longueur, historique, verrouillage, second facteur (démo 3) |
| Départ d’un salarié : penser à chaque application | Un seul compte à désactiver, puis les jetons en cours à couper |
| Second facteur : à construire dans Keystone | Natif dans Keycloak (TOTP, WebAuthn) |

## 2.9 En entreprise : affiner le mapping

Le mapping du lab donne `member` à **toute** personne authentifiée par Keycloak. En entreprise, on filtre sur un **groupe** de l’annuaire, transmis dans l’assertion.

**1. Dans Keycloak, transmettre les groupes.** Realm **lab** :

- **Groups → Create group** : `openstack-formation`.
- **Clients → urn:saml2:keycloak → Client scopes → urn:saml2:keycloak-dedicated → Configure a new mapper → Group list** : *Name* `groups`, *Group attribute name* `groups`, *SAML Attribute NameFormat* `Basic`, *Single Group Attribute* désactivé, *Full group path* **désactivé** → **Save**.

*Full group path* désactivé : l’assertion contient `openstack-formation`, et non `/openstack-formation`.

**2. Dans Keystone, la règle filtrée.** Le module SAML d’Apache (`mod_auth_mellon`) transmet chaque attribut de l’assertion sous le nom `MELLON_<attribut>` : l’attribut `groups` devient `MELLON_groups`.

```bash
cat > ~/lab-sec-mapping-groupe.json <<'JSON'
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
JSON
openstack mapping set --rules ~/lab-sec-mapping-groupe.json keycloak_saml
python3 ~/test-sso-saml.py alice
```

✅ **Attendu** (18 octobre) : `3. Réponse de Keystone : 401`. `alice` est authentifiée par Keycloak, mais n’est pas dans le groupe : Keystone ne lui donne **rien**. Son journal (`sudo k8s kubectl -n openstack logs keystone-0 -c keystone --since=5m | grep "Could not map"`) indique `Could not map any federated user properties to identity values`.

**3. Ajouter `alice` au groupe** : **Users → alice → Groups → Join Group → openstack-formation**. Puis `python3 ~/test-sso-saml.py alice` de nouveau.

✅ **Attendu** (18 octobre) : `3. Réponse de Keystone : 201`.

Remettre le mapping du lab pour la suite :

```bash
openstack mapping set --rules ~/lab/mapping.json keycloak_saml
```

## 2.10 La ligne de commande pour un utilisateur fédéré

La connexion fédérée ne donne pas de mot de passe utilisable par la commande `openstack`. Pour un script, l’utilisateur fédéré crée un **identifiant d’application** limité au projet, avec une date d’expiration (`labs-iam/04-roles-moindre-privilege.md`, § 4.5.2). C’est aussi ce qui évite de partager un mot de passe.

```bash
F application credential create lab-sec-alice-script --role member \
  --expiration "$(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%S)"
```

✅ **Attendu** (18 octobre) : un **refus**.

```text
BadRequestException: 400: … Invalid application credential: Could not find role assignment with role: ee240b17…, user or group: c1c6c722…, project, domain, or system: 5b7e3738…
```

**Comment le lire :** le rôle d’`alice` vient du groupe du mapping, recalculé à chaque connexion et non enregistré. Un identifiant d’application doit survivre à la session : Keystone exige donc un rôle **enregistré** pour l’utilisateur lui-même.

**La parade :** la plateforme donne le rôle directement à l’utilisateur fantôme, pour les seules personnes qui ont besoin de scripts.

```bash
FED=$(openstack user list --domain keycloak -f value -c ID | head -1)
openstack role add --user "$FED" --project formation --project-domain keycloak member
umask 077
F application credential create lab-sec-alice-script --role member \
  --expiration "$(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%S)" -f json > ~/lab-sec-alice-ac.json
ACID=$(python3 -c 'import json; print(json.load(open("/home/vagrant/lab-sec-alice-ac.json"))["id"])')
ACS=$(python3 -c 'import json; print(json.load(open("/home/vagrant/lab-sec-alice-ac.json"))["secret"])')
env -u OS_USERNAME -u OS_PASSWORD -u OS_USER_DOMAIN_NAME -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME \
  openstack --os-auth-type v3applicationcredential \
  --os-application-credential-id "$ACID" --os-application-credential-secret "$ACS" network list -c Name
```

✅ **Attendu** (18 octobre) : l’identifiant est créé, et la dernière commande liste `external-network` : le script agit sur `formation`, sans aucun mot de passe de personne.

⚠ Ce rôle enregistré ne dépend plus de Keycloak : si `alice` quitte l’entreprise, il faut **aussi** retirer ce rôle et supprimer ses identifiants d’application. C’est le prix de l’automatisation.

## Retour arrière

Après la démo 3, qui utilise `alice`. Joué le 18 octobre : il faut plus que l’ordre inverse de la création, à cause de deux défauts des charms (étape 3 ci-dessous).

**1. Les objets de Keystone**, dans l’ordre inverse de la création :

```bash
source ~/admin-openrc
FED=$(openstack user list --domain keycloak -f value -c ID | head -1)
openstack federation protocol delete --identity-provider keycloak saml2
openstack mapping delete keycloak_saml
openstack identity provider delete keycloak
openstack user delete "$FED"
openstack project delete formation --domain keycloak
openstack group delete lab_members --domain keycloak
openstack domain set --disable keycloak
openstack domain delete keycloak
rm -f ~/lab-sec-fed-token ~/lab-sec-alice-ac.json ~/lab-sec-mapping-groupe.json ~/lab-sec-bundle.pem
```

Supprimer l’utilisateur fantôme supprime aussi ses identifiants d’application (2.10). Ne pas passer par `openstack application credential … --user` : le client 7.5.0 y échoue (`'function' object has no attribute 'list'`, constaté).

**2. Le fournisseur côté Juju :**

```bash
juju remove-application -m openstack keystone-idp-saml2-keycloak --no-prompt
```

Sans `--no-prompt`, Juju demande une confirmation (`Continue [y/N]?`) et, dans un script, abandonne (`ERROR application removal: aborted`). L’application disparaît de `juju status` en une minute environ ; Keystone répond `503` quelques instants, le temps de reconfigurer Apache.

**3. Horizon garde le choix « Keycloak - Lab ».** Constaté : plus de 8 minutes après, la page de connexion le propose toujours. Lu dans le code du charm Keystone (`_handle_update_trusted_dashboard`) : quand il ne reste aucun fournisseur fédéré, il s’arrête (`No OAuth relations found, skipping update`) **sans effacer** la liste déjà envoyée à Horizon. Et Horizon ne réécrit pas sa configuration tout seul. Il faut donc recréer la relation, puis faire régénérer la configuration de Horizon :

```bash
juju remove-relation -m openstack horizon:trusted-dashboard keystone:trusted-dashboard
juju status -m openstack --relations | grep trusted-dashboard      # attendre qu'elle ait disparu (20 s environ)
juju integrate -m openstack horizon:trusted-dashboard keystone:trusted-dashboard
juju exec -m openstack --unit horizon/0 -- "JUJU_DISPATCH_PATH=hooks/config-changed ./dispatch"
curl -s --cacert "$OS_CACERT" https://cloud.lab.test/openstack-horizon/auth/login/ | grep -c "Keycloak - Lab"
```

✅ **Attendu** (18 octobre) : `0`. La page de connexion de Horizon redevient le simple formulaire d’origine, sans liste de méthodes.

`JUJU_DISPATCH_PATH=hooks/config-changed ./dispatch`, lancé par `juju exec` dans l’unité, rejoue le traitement « configuration modifiée » du charm : il réécrit `local_settings.py` à partir de l’état actuel des relations.

**4. La paire de clés SAML** (2.4), confiée au charm Keystone :

```bash
juju config -m openstack keystone saml-x509-keypair          # affiche secret:<ID>
juju config -m openstack keystone --reset saml-x509-keypair
juju remove-secret -m openstack <ID>                          # 🔁 l'identifiant affiché
rm -rf ~/saml ~/keycloak-idp.xml ~/keycloak-saml*.yaml ~/keystone-sp.xml ~/ca-lf.crt ~/test-sso-saml.py
```

✅ **Attendu :** `juju config … saml-x509-keypair` n’affiche plus rien, et Keystone revient `active idle` en moins d’une minute.

**5. Dans Keycloak :** supprimer le client `urn:saml2:keycloak`, le groupe `openstack-formation` et l’utilisateur `alice`.

⚠ Ce qui reste : la tentative du début de 2.4 a laissé les réponses de `sunbeam identity provider add` dans la base de Sunbeam (élément `FeatureSSOExternalIDPConfig-saml2-keycloak`). Sans effet sur le cloud ; elles seraient reproposées à un nouvel essai. Si un instantané VirtualBox a été pris avant la démo, le restaurer est le retour arrière le plus complet.

## À retenir

- La fédération **sépare** la vérification de l’identité (Keycloak) des droits (Keystone).
- Le mot de passe des personnes ne passe plus par OpenStack ; la politique et la désactivation des comptes se gèrent **à un seul endroit**, puis on coupe les jetons en cours.
- La fédération ne donne aucun droit : c’est le **mapping**, les **groupes** et les **affectations** qui décident, selon le moindre privilège.
- Sur ce lab (Sunbeam 2024.1), la mise en place bute sur des défauts des outils : `sunbeam identity provider add` avec une CA privée, la validation de `ca-chain` du charm `keystone-saml-k8s`, l’absence de `trusted_dashboard` pour SAML dans le charm Keystone. Chaque contournement est expliqué : lire le code et les journaux, plutôt que deviner.
- Le compte `admin` local de Keystone reste indispensable pour réparer une fédération en panne : c’est un compte de **secours**, à protéger particulièrement (démo 3).

# Démo 4 : chiffrement des flux (TLS)

> **Statut : rédigé, pas encore rejoué sur le lab.** Le constat de départ (adresses en `http://`) a été relevé le 17 octobre 2026 (TP 1 des `labs-iam/`, § 1.3). Les commandes `sunbeam enable tls …` viennent de l’aide de la CLI et du guide Keycloak (section 11) : leur syntaxe exacte est à confirmer avec `--help`.

## Objectif

**Constater** que les API du cloud circulent en clair, **montrer** ce que cela expose (un jeton lisible sur le réseau), **activer** TLS sur les adresses publiques et internes, **gérer** les certificats, puis **durcir**.

⚠ **Avant de commencer : un instantané VirtualBox de Lab-Sunbeam.** L’activation de TLS change toutes les adresses du cloud (`http://` → `https://`) : les fichiers d’accès (`~/admin-openrc`, `~/demo-openrc`, `clouds.yaml`) et les guides qui utilisent `http://` ne fonctionneront plus tels quels. Le retour arrière est décrit en fin de démo, mais l’instantané est la sécurité.

## 4.1 Le constat : ce qui circule en clair

### Les adresses du catalogue

```bash
source ~/admin-openrc
openstack endpoint list -c "Service Name" -c Interface -c URL | grep -E "keystone|nova|neutron"
```

✅ **Attendu** (observé le 17 octobre) : des adresses `http://cloud.lab.test:80/…` (interface `public`), `http://internal.lab.test:80/…` (`internal`), `http://10.152.183.x:…` (`admin`). **Aucune** en `https://`.

### Voir un jeton passer en clair

**À quoi ça sert :** rendre le risque concret. Chaque appel à une API transporte un **jeton** dans l’en-tête `X-Auth-Token` ; ce jeton vaut un mot de passe pendant une heure.

Dans une **première** session SSH sur Lab-Sunbeam :

```bash
sudo tcpdump -i any -A -s 0 'host 192.168.56.201 and tcp port 80' 2>/dev/null | grep -m 2 -o "X-Auth-Token: .\{20\}"
```

Dans une **seconde** session :

```bash
source ~/demo-openrc
openstack network list
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `tcpdump -i any` | Écoute le trafic de toutes les cartes réseau de la VM |
| `-A -s 0` | Affiche le **contenu** des paquets en texte, en entier |
| `'host 192.168.56.201 and tcp port 80'` | Seulement le trafic vers l’adresse publique du cloud, en HTTP |
| `grep -m 2 -o "X-Auth-Token: .\{20\}"` | Les 2 premières lignes de jeton, coupées à 20 caractères (pour ne pas projeter un jeton entier) |

✅ **Attendu :** deux lignes `X-Auth-Token: gAAAAAB…`. **Quiconque écoute le réseau** de gestion lit les jetons, et peut s’en servir (TP 1 des `labs-iam/`, § 1.4.5 : un jeton est un « porteur »).

Révoquer aussitôt les jetons de la démonstration :

```bash
openstack token revoke "$(openstack token issue -f value -c id)"
```

(Ceci révoque un nouveau jeton ; ceux capturés expirent dans l’heure. En formation, c’est l’occasion de rappeler pourquoi la durée de vie des jetons est courte.)

### Les flux internes

Les services s’appellent entre eux par `internal.lab.test` (`192.168.56.202`) : nova-compute vers Neutron, validation des jetons auprès de Keystone. Même expérience, sur l’adresse interne :

```bash
sudo timeout 60 tcpdump -i any -A -s 0 'host 192.168.56.202 and tcp port 80' 2>/dev/null | grep -m 2 -o -E "X-(Auth|Subject)-Token: .{20}"
```

✅ **Attendu :** des jetons, cette fois ceux des **comptes de service** (qui ont le rôle `admin` sur `services`), ou ceux des utilisateurs en cours de validation (`X-Subject-Token`).

### Ce qui est déjà protégé

TLS n’est pas absent partout. ⚠ **à vérifier** sur le lab, pour compléter le tableau :

| Flux | Constat attendu | Vérification |
|---|---|---|
| Navigateur → Keycloak | **HTTPS** (`sslRequired: all` dans `lab-sunbeam/keycloak/realm.json`, port 8443) | `curl -sI https://sso.lab.test:8443` |
| Bases OVN (réseau virtuel) | Certificats émis par l’application `certificate-authority` du modèle `openstack` (charm `self-signed-certificates`) | `juju status -m openstack certificate-authority --relations` |
| API publiques et internes | **HTTP** | ci-dessus |
| Contrôleur Juju | Chiffré (port 17070, TLS) | `juju show-controller sunbeam-controller \| grep ca-cert` |

**Sécurité par défaut :** Sunbeam chiffre ses flux de **contrôle** (Juju, OVN) mais laisse les **API** en HTTP : c’est à l’exploitant d’activer TLS, avec ses propres certificats.

## 4.2 Choisir le mode de TLS

Sunbeam propose trois fonctionnalités (`sunbeam list-features`) :

| Fonctionnalité | Qui émet les certificats | Pour qui |
|---|---|---|
| `tls.self-signed` | Une CA créée par Sunbeam, propre au cloud | Lab, démonstration : le plus simple |
| `tls.ca` | **Ta** CA d’entreprise : Sunbeam produit des demandes de certificat (CSR), tu les signes | Production avec une PKI existante |
| `tls.vault` | Vault, dont le certificat d’autorité intermédiaire est signé par ta CA | Production : renouvellement automatique (guide Keycloak, section 11) |

Et trois cibles (option `--endpoint`, répétable) : `public` (`cloud.lab.test`), `internal` (`internal.lab.test`), `rgw` (stockage objet).

Pour la démo : **`tls.self-signed`** sur `public` et `internal`. Le guide Keycloak (section 11) décrit la variante `tls.vault` avec la CA du lab, plus proche de la production.

## 4.3 Activer TLS

D’abord, l’aide de la commande :

```bash
sunbeam enable tls self-signed --help
```

⚠ **à vérifier :** le nom exact de l’option (`--endpoint`) et ses valeurs. Puis :

```bash
sunbeam enable tls self-signed --endpoint public --endpoint internal
```

**Ce qui se passe** (plusieurs minutes, à suivre dans une seconde session avec `watch -n 15 "juju status -m openstack --format short | grep -v workload:active"`) : Sunbeam crée la CA, Juju distribue un certificat aux répartiteurs Traefik (`traefik-public`, `traefik`), qui servent alors en HTTPS, et les charms mettent à jour les adresses du **catalogue**.

✅ **Attendu :** la commande se termine sans erreur ; `sunbeam list-features | grep tls` montre `tls.self-signed` coché.

### Mettre à jour les accès

```bash
sunbeam openrc > ~/admin-openrc
grep -E "OS_AUTH_URL|OS_CACERT" ~/admin-openrc
source ~/admin-openrc
openstack token issue -c expires
openstack endpoint list -c "Service Name" -c Interface -c URL | grep -E "keystone|nova"
```

✅ **Attendu :** `OS_AUTH_URL=https://cloud.lab.test/…`, une variable `OS_CACERT` qui pointe vers le certificat de la CA, un jeton obtenu, et des adresses `public` et `internal` en `https://`.

**Comment le lire :** `OS_CACERT` dit au client **quelle CA** croire. Sans elle, le client refuserait le certificat (`SSL: CERTIFICATE_VERIFY_FAILED`) : c’est voulu.

Régénérer aussi `~/demo-openrc` : `sunbeam cloud-config` (démo 2 du guide `3-sunbeam-cli/`) ou une copie de l’openrc adaptée aux comptes `demo`.

### Faire confiance à la CA sur Windows

Le navigateur doit connaître la nouvelle CA pour ouvrir Horizon sans alerte.

1. Sur Lab-Sunbeam, afficher le certificat de la CA (fichier public, sans clé privée) :

   ```bash
   CA=$(grep -oP "OS_CACERT=\K.*" ~/admin-openrc); echo "$CA"; openssl x509 -in "$CA" -noout -subject -enddate
   cat "$CA"
   ```

2. Copier le bloc `-----BEGIN CERTIFICATE-----` … `-----END CERTIFICATE-----` dans un fichier `sunbeam-ca.crt` sur Windows.
3. Double-cliquer sur `sunbeam-ca.crt` → **Installer le certificat** → **Ordinateur local** → **Placer tous les certificats dans le magasin suivant** → **Autorités de certification racines de confiance** → **Terminer**.
4. Ouvrir l’adresse donnée par `sunbeam dashboard url`.

✅ **Attendu :** Horizon en `https://cloud.lab.test/…`, cadenas fermé, sans avertissement.

## 4.4 Vérifier le chiffrement

Rejouer la capture de 4.1, sur le port **443** cette fois :

```bash
sudo timeout 30 tcpdump -i any -A -s 0 'host 192.168.56.201 and tcp port 443' 2>/dev/null | grep -c "X-Auth-Token"
```

Dans une seconde session : `openstack network list`.

✅ **Attendu :** `0`. Le trafic passe, mais son contenu est chiffré : plus aucun jeton lisible.

Lire le certificat servi :

```bash
echo | openssl s_client -connect cloud.lab.test:443 -servername cloud.lab.test -CAfile "$CA" 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

| Champ | Ce qu’on vérifie |
|---|---|
| `subject`, `subjectAltName` | Le nom `cloud.lab.test` y figure : sinon, alerte du navigateur |
| `issuer` | La CA créée par Sunbeam |
| `notBefore`, `notAfter` | La période de validité : la date d’expiration est à **surveiller** (4.5) |

Et la chaîne complète : `echo | openssl s_client -connect cloud.lab.test:443 -CAfile "$CA" 2>/dev/null | grep "Verify return code"` doit répondre `0 (ok)`.

## 4.5 Gérer les certificats

| Certificat | Émis par | Durée | Renouvellement |
|---|---|---|---|
| API publiques et internes (`tls.self-signed`) | CA de Sunbeam | ⚠ à relever (`notAfter`, 4.4) | ⚠ à vérifier : renouvellement automatique par le charm de la CA ? |
| API (`tls.ca`) | CA de l’entreprise | Selon la PKI | **Manuel** : nouvelles CSR, signature, réinjection |
| API (`tls.vault`) | Vault | Courte | Automatique, tant que le certificat intermédiaire de Vault est valide (1 an, `sign-vault.sh`) |
| Keycloak | CA du lab (`init-keycloak.sh`) | **365 jours** | Manuel : nouveau certificat signé par la CA du lab, redémarrage du conteneur |
| CA du lab | Elle-même | Jusqu’en **2031** | Prévoir le remplacement bien avant |

**Surveiller l’expiration :** un certificat expiré coupe tout le cloud d’un coup. Une vérification simple, à planifier chaque semaine :

```bash
for h in cloud.lab.test:443 internal.lab.test:443 sso.lab.test:8443; do
  printf "%-24s " "$h"
  echo | openssl s_client -connect "$h" -servername "${h%:*}" 2>/dev/null | openssl x509 -noout -enddate
done
```

En production, la même vérification alimente la supervision (alerte à 30 jours de l’échéance).

## 4.6 Durcir

| Mesure | Pourquoi | Vérification |
|---|---|---|
| Ne jamais utiliser `--insecure` ni `curl -k` comme solution | Ces options acceptent **n’importe quel** certificat : TLS ne protège plus de l’usurpation | `grep -r "insecure" ~/*.sh ~/.config/openstack/ 2>/dev/null` |
| Chiffrer aussi l’interface **interne** | Les jetons des comptes de service y circulent (4.1) | `--endpoint internal` (4.3) |
| Refuser les versions anciennes de TLS | TLS 1.0 et 1.1 sont obsolètes | `echo \| openssl s_client -connect cloud.lab.test:443 -tls1_1 2>&1 \| grep -i -E "alert\|error"` : attendu, un refus (⚠ à vérifier sur Traefik) |
| Protéger la clé de la CA | Avec elle, on fabrique des certificats acceptés partout | Clé de la CA du lab : `~/lab/keycloak/pki/ca.key` sur Lab-Services, droits `600`, hors des sauvegardes non chiffrées |
| Une CA d’entreprise plutôt qu’auto-signée en production | Les postes font déjà confiance à la CA de l’entreprise ; révocation possible | `tls.ca` ou `tls.vault` |

## Retour arrière

```bash
sunbeam disable tls self-signed --help
sunbeam disable tls self-signed
sunbeam openrc > ~/admin-openrc
source ~/admin-openrc
openstack endpoint list -c "Service Name" -c Interface -c URL | grep keystone
```

⚠ **à vérifier :** la syntaxe exacte de `disable`, et que les adresses reviennent bien en `http://`. Sur Windows, retirer la CA : `certlm.msc` → **Autorités de certification racines de confiance** → **Certificats** → supprimer la CA de Sunbeam. En cas de doute, restaurer l’instantané VirtualBox.

## À retenir

- Par défaut, Sunbeam chiffre ses flux de **contrôle** (Juju, OVN), mais laisse les **API** en HTTP : jetons et mots de passe circulent en clair, y compris ceux des comptes de service sur l’interface interne.
- TLS s’active par une **fonctionnalité** de Sunbeam (`tls.self-signed`, `tls.ca`, `tls.vault`), sur les adresses `public`, `internal` et `rgw`.
- Activer TLS, c’est aussi **gérer** : distribuer la CA aux clients, surveiller les dates d’expiration, renouveler.
- Durcir : jamais de `--insecure`, l’interne aussi, une CA d’entreprise en production.

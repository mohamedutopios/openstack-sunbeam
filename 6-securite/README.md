# Sécuriser un cloud Sunbeam : démos de la journée

Ce dossier accompagne la journée **sécurité** de la formation. Les supports théoriques sont `support/11-openstack-authentification-externe-mots-de-passe-v1.0.html` et `support/13-openstack-micro-segmentation-audit-v1.0.html` ; ici, on **fait**, pas à pas, sur le lab.

> **Statut : rédigé, pas encore rejoué sur le lab.** Les commandes s’appuient sur ce que le lab a déjà montré (TP 1 des `labs-iam/`, guides `3-sunbeam-cli/`, `4-juju-cli/`, `5-openstack-cli/`, guide Keycloak). Les points non vérifiés sont signalés **⚠ à vérifier**, avec la commande qui permet de le faire. Rejouer chaque démo avant de l’utiliser en formation.

## Objectifs de la journée

| Objectif | Démos |
|---|---|
| Fédération d’identité, durcissement selon le référentiel ANSSI | 1, 2, 3, 4 |
| Centralisation des journaux | 6 |
| Isolation réseau entre tenants | 5 |

## Le parcours

| # | Fichier | Thème du programme | Durée indicative |
|---|---|---|---|
| 1 | [01-federation-keycloak.md](01-federation-keycloak.md) | Fédération d’identité : une autorité d’authentification externe (Keycloak) | 1 h 30 |
| 2 | [02-mots-de-passe-anssi.md](02-mots-de-passe-anssi.md) | Politique de mots de passe ANSSI : complexité, expiration, verrouillage, impact sur les comptes de service | 1 h |
| 3 | [03-rotation-secrets.md](03-rotation-secrets.md) | Rotation des mots de passe de service et des clés de jetons | 45 min |
| 4 | [04-chiffrement-tls.md](04-chiffrement-tls.md) | Chiffrement des flux : constat, activation, certificats, sécurité par défaut et durcissement | 1 h 15 |
| 5 | [05-reseau-micro-segmentation.md](05-reseau-micro-segmentation.md) | Réseaux partagés et privés, isolation entre tenants, groupes de sécurité, micro-segmentation | 1 h 30 |
| 6 | [06-audit-tracabilite.md](06-audit-tracabilite.md) | Journalisation des services, corrélation, centralisation | 1 h 15 |

**Dépendance entre démos :** la fédération SAML (démo 1) exige que l’adresse publique du cloud soit en **HTTPS**. L’activation du TLS public est décrite en démo 4 (étape 4.3). Si l’on suit l’ordre du programme, faire cette étape **avant** la démo 1, ou l’avoir faite la veille.

## L’environnement

| Élément | Valeur sur le lab |
|---|---|
| Cloud | Lab-Sunbeam, `192.168.56.10` ; API publique `cloud.lab.test` (`192.168.56.201`), interne `internal.lab.test` (`.202`) |
| Autorité d’authentification | Keycloak 26.8.0 dans Lab-Services (`192.168.56.20`), `https://sso.lab.test:8443`, realm `lab` |
| Autorité de certification du lab | `ca.crt` à la racine du dépôt : « Sunbeam Lab Root CA », valable jusqu’en 2031, approuvée sur Windows par `Configurer-Lab-Sunbeam.ps1` |
| Réseau externe (IP flottantes) | `external-network`, `192.168.57.0/24` ; le PC Windows y a l’adresse `192.168.57.1` |
| Comptes OpenStack | `admin` (domaine `admin_domain`), `demo` (domaine `users`) |

## Avant la séance : un lab stable

Les démos créent des VM et modifient le cloud : elles demandent un lab **en bon état**. Constaté le 17 octobre : après une veille du PC, ou sans Lab-Services, le cloud répond de façon aléatoire (`503`, « Keystone service is temporarily unavailable »).

1. **Désactiver la veille du PC** pendant la journée (PowerShell, PC branché) :

   ```powershell
   powercfg /change standby-timeout-ac 0
   ```

2. **Démarrer Lab-Services, puis Lab-Sunbeam**, dans cet ordre : Lab-Services fournit le DNS du lab et Keycloak.
3. **Vérifier le cloud** dans Lab-Sunbeam (guide `3-sunbeam-cli/DEMOS-ENTREPRISE.md`, démo 1) :

   ```bash
   sunbeam utils juju-login
   sunbeam cluster list
   juju status -m openstack --format short | grep -v "workload:active"
   uptime
   ```

   Attendu : tous les rôles `active`, aucune ligne pour la deuxième commande, et une charge (`load average`) redescendue sous 10 environ.
4. **Prendre un instantané** de Lab-Sunbeam et de Lab-Services dans VirtualBox (Machine → Prendre un instantané) : les démos 1, 3 et 4 modifient durablement le cloud, l’instantané garantit le retour arrière.

## Conventions

| Repère | Signification |
|---|---|
| **À quoi ça sert** | Le but de l’étape, et ce qu’on en attend en entreprise |
| **La commande, mot par mot** | Le rôle de chaque morceau |
| ✅ **Attendu** | Le résultat prévu (non encore observé sur le lab) |
| **Comment le lire** | L’interprétation |
| **Si tu vois autre chose** | Les écarts probables et la conduite à tenir |
| ⚠ **à vérifier** | Syntaxe ou comportement non confirmé : commande de vérification fournie |
| 🔁 **À adapter** | Valeur propre au lab à remplacer chez toi |

Les objets créés par les démos commencent par `lab-sec` : un nom facile à retrouver pour le nettoyage.

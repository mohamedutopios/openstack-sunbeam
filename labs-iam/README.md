# Travaux pratiques : identité, cloisonnement et moindre privilège

Ces travaux pratiques accompagnent les modules 8 à 12 du support (`support/8-…` à `support/12-…`). Les supports expliquent ; ici, on **fait** : chaque TP enchaîne des scénarios reproductibles, avec les commandes, le résultat attendu, les questions à poser aux apprenants et les pièges.

> **Statut : rédigé, pas encore rejoué sur le lab.** Les commandes suivent la syntaxe du client `openstack` 7.x et de Keystone 2024.1 (Caracal), déployé par Sunbeam. Certains comportements dépendent de l’activation de Secure RBAC sur ton lab : le TP 1 (§ 1.6) le fait vérifier en premier, et chaque résultat concerné est signalé par **⚖ dépend de Secure RBAC**. Rejoue chaque TP avant de l’utiliser en formation et note les écarts.

## Parcours

| Fichier | Thème | Module du support | Durée indicative |
|---|---|---|---|
| [01-keystone-en-profondeur.md](01-keystone-en-profondeur.md) | Entités, catalogue, jetons, sources d’identité, politiques | 8 | 2 h 30 |
| [02-domaines.md](02-domaines.md) | Espaces d’identité indépendants, administration globale ou déléguée | 9 | 2 h |
| [03-projets-hierarchie-quotas.md](03-projets-hierarchie-quotas.md) | Projets, hiérarchie, rôles hérités, quotas, modèles d’organisation | 10 | 2 h 30 |
| [04-roles-moindre-privilege.md](04-roles-moindre-privilege.md) | Rôles standards, Secure RBAC, rôles impliqués, comptes de service | 12 | 2 h 30 |
| [05-scenario-integre.md](05-scenario-integre.md) | Simulation complète : construire et auditer un SI cloisonné | 8 à 12 | 3 h |

Les TP se suivent : chacun réutilise ce que le précédent a créé. Le TP 5 peut aussi se jouer seul, comme évaluation.

## Le fil rouge : l’entreprise Acme

Une entreprise, **Acme**, confie son SI à la plateforme OpenStack. Elle travaille avec un **partenaire** externe, qui doit rester totalement séparé.

```text
Plateforme (équipe cloud)        → domaine admin_domain, projet admin   (déjà créés par Sunbeam)
│
├── Domaine acme                 → administré par Bruno (gestionnaire de domaine)
│   └── Projet acme-si           → racine de l'arbre d'Acme (ne contient pas de ressources)
│       ├── acme-prod            → production : Emma (exploitation)
│       ├── acme-recette         → recette
│       └── acme-dev             → développement : Chloé (développeuse)
│
└── Domaine partenaire           → un autre Bruno, sans aucun lien avec celui d'Acme
    └── Projet partenaire-app
```

| Personne | Compte | Domaine | Rôle visé | TP |
|---|---|---|---|---|
| Équipe plateforme | `admin` (existant) | `admin_domain` | admin | tous |
| Bruno | `bruno` | `acme` | manager du domaine `acme` | 2, 4 |
| Chloé | `chloe` | `acme` | member de `acme-dev` | 3, 4 |
| Emma | `emma` | `acme` | member de `acme-prod`, reader sur tout Acme | 3, 4 |
| David | `david` | `acme` | reader (auditeur) sur tout Acme | 4 |
| Bruno du partenaire | `bruno` | `partenaire` | manager du domaine `partenaire` | 2 |
| Sauvegarde | identifiant d’application de `emma` | — | reader limité | 4 |

## Où taper les commandes

Toutes les commandes se lancent **dans Lab-Sunbeam** :

```powershell
ssh vagrant@192.168.56.10
```

Copie d’abord ce dossier depuis Windows (PowerShell, dans le dossier du dépôt) :

```powershell
scp -r .\labs-iam vagrant@192.168.56.10:~/
```

## Préparer la session : `lab-env.sh`

Au début de **chaque** session SSH :

```bash
source ~/labs-iam/lab-env.sh
```

Ce fichier :

- charge les identifiants administrateur (`~/admin-openrc`) ;
- demande **une fois** le mot de passe commun des comptes de TP (`LAB_PW`), sans l’afficher ;
- définit la fonction **`os_as`**, qui exécute une commande `openstack` **en tant qu’un autre utilisateur**, avec la portée voulue, sans toucher à ta session administrateur.

```bash
os_as <utilisateur> <domaine-de-l-utilisateur> projet <projet> <domaine-du-projet> -- <commande openstack>
os_as <utilisateur> <domaine-de-l-utilisateur> domaine <domaine> -- <commande openstack>
os_as <utilisateur> <domaine-de-l-utilisateur> systeme -- <commande openstack>
```

Exemples :

```bash
os_as chloe acme projet acme-dev acme -- server list
os_as bruno acme domaine acme -- user list --domain acme
```

Sans `os_as`, les commandes s’exécutent en tant qu’`admin` sur le projet `admin`.

## Conventions

| Convention | Signification |
|---|---|
| ✅ **Attendu** | Résultat attendu si tout est conforme |
| ⛔ **Refus attendu** | La commande **doit** échouer : c’est la démonstration |
| ⚖ **dépend de Secure RBAC** | Le résultat change selon `enforce_scope` et `enforce_new_defaults` (TP 1, § 1.6) |
| ❓ **Question** | À poser aux apprenants avant de lancer la commande |
| ⚠ **Piège** | Erreur fréquente ou comportement contre-intuitif |

Les noms d’objets créés commencent par `acme` ou `partenaire`, ou par `lab-` : c’est ce qui permet au nettoyage de tout retrouver.

## Nettoyer

```bash
bash ~/labs-iam/nettoyage.sh
```

Le script supprime, dans l’ordre, les identifiants d’application, les projets, les utilisateurs, les groupes, les rôles et les domaines créés par les TP. Il affiche ce qu’il va supprimer et demande confirmation. Les objets de Sunbeam (`admin_domain`, `service_domain`, `users`, `Default`, projets `admin`, `demo`, `services`) ne sont jamais touchés.

**Avant un export des OVA**, lance le nettoyage puis supprime le dossier : `rm -rf ~/labs-iam`.

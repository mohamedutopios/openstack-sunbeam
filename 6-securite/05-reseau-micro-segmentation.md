# Démo 5 : réseau, isolation entre tenants et micro-segmentation

> **Statut : rédigé, pas encore rejoué sur le lab.** Les commandes `openstack` sont celles des guides `5-openstack-cli/` et du support 13 ; les adresses sont celles du lab. Support : module 13.

## Objectif

Construire deux **tenants** (projets) étanches, avec chacun son réseau privé, puis découper l’un d’eux en **zones** (web, base de données) dont les échanges sont limités au strict nécessaire, et **prouver** chaque cloisonnement par un essai de connexion.

```text
                           external-network 192.168.57.0/24 (IP flottantes)
                     ┌───────────────┴─────────────────────┐
                 lab-sec-a-router                      lab-sec-b-router
                     │                                     │
 Tenant A  lab-sec-a-net 10.60.1.0/24          Tenant B  lab-sec-b-net 10.60.2.0/24
   ├── lab-sec-a-web  [sg-web]  + IP flottante     └── lab-sec-b-app [sg-b-app] + IP flottante
   └── lab-sec-a-db   [sg-db]   sans IP flottante

 Autorisé :  PC (192.168.57.1) → a-web (ping, SSH)     a-web → a-db (ping)     PC → b-app (ping)
 Interdit :  PC → a-db     b-app → a-web, a-db     a-web → b-app (privé ou flottant)
```

## 5.1 Comprendre : les quatre niveaux d’isolation

| Niveau | Mécanisme | Ce qu’il isole |
|---|---|---|
| **Visibilité** | Chaque ressource appartient à un projet | Un tenant ne **voit** pas les réseaux, VM, groupes de sécurité des autres |
| **Réseau privé** | Un réseau par projet, réalisé par OVN en réseau virtuel (*overlay*) distinct | Deux projets peuvent même utiliser les **mêmes** adresses : leurs paquets ne se mélangent pas |
| **Routage** | Un routeur par projet ; pas de route entre deux réseaux privés | Seule sortie : le réseau externe, par traduction d’adresse (SNAT) ou IP flottante |
| **Groupes de sécurité** | Pare-feu à **état**, appliqué à chaque **port** de VM | Qui peut parler à **chaque** VM, sur quel protocole : c’est la micro-segmentation |

### Réseau partagé ou réseau privé

```bash
source ~/admin-openrc
openstack network list --long -c Name -c Project -c Shared -c "Router Type" -c Subnets
```

✅ **Attendu :** `external-network`, marqué **externe** (`Router Type: External`), et `demo-network`, privé au projet `demo`.

| Type | Créé par | Visible par | Usage |
|---|---|---|---|
| **Externe** | Plateforme | Tous (pour y brancher un routeur ou prendre une IP flottante) | Sortie vers l’extérieur |
| **Partagé** (`--share`) | Plateforme seulement (`create_network:shared` = `rule:admin_only`, TP 1 des `labs-iam/`, § 1.6.2) | **Tous** les projets | Rare : un réseau commun à tout le cloud. À éviter, il casse l’isolation |
| **Partagé à certains projets** (RBAC réseau) | Propriétaire ou plateforme | Les seuls projets désignés | Partage contrôlé (5.7) |
| **Privé** | Membre du projet | Le seul projet | Le cas normal |

## 5.2 Préparer les deux tenants (plateforme)

**À quoi ça sert :** un domaine dédié, deux projets, un compte par tenant, et des quotas bas. Tout porte le préfixe `lab-sec` pour le nettoyage.

```bash
source ~/admin-openrc
read -rsp "Mot de passe des comptes de demo : " LAB_PW; echo
openstack domain create lab-sec --description "Demo micro-segmentation"
for t in a b; do
  openstack project create lab-sec-$t --domain lab-sec
  openstack user create lab-sec-$t-user --domain lab-sec --password "$LAB_PW"
  openstack role add --user lab-sec-$t-user --user-domain lab-sec --project lab-sec-$t --project-domain lab-sec member
  openstack quota set "$(openstack project show lab-sec-$t --domain lab-sec -f value -c id)" \
    --instances 2 --cores 2 --ram 1024 --floating-ips 1 --networks 1 --routers 1
done
openstack role assignment list --domain lab-sec --names
openstack role assignment list --project lab-sec-a --project-domain lab-sec --names
```

Deux fonctions pour agir **en tant que** chaque tenant, sans toucher à la session administrateur :

```bash
A() { env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME -u OS_USERNAME -u OS_USER_DOMAIN_NAME -u OS_PASSWORD \
      openstack --os-username lab-sec-a-user --os-user-domain-name lab-sec --os-password "$LAB_PW" \
                --os-project-name lab-sec-a --os-project-domain-name lab-sec "$@"; }
B() { env -u OS_PROJECT_NAME -u OS_PROJECT_DOMAIN_NAME -u OS_USERNAME -u OS_USER_DOMAIN_NAME -u OS_PASSWORD \
      openstack --os-username lab-sec-b-user --os-user-domain-name lab-sec --os-password "$LAB_PW" \
                --os-project-name lab-sec-b --os-project-domain-name lab-sec "$@"; }
A token issue -c project_id -c expires
B token issue -c project_id -c expires
```

✅ **Attendu :** deux jetons, avec deux `project_id` différents.

⚠ Si le TLS a été activé (démo 4), `OS_AUTH_URL` et `OS_CACERT` viennent de `~/admin-openrc` et restent valables dans ces fonctions.

## 5.3 Le réseau de chaque tenant

**Tenant A :**

```bash
A network create lab-sec-a-net
A subnet create lab-sec-a-subnet --network lab-sec-a-net --subnet-range 10.60.1.0/24 --dns-nameserver 192.168.56.20
A router create lab-sec-a-router
A router set lab-sec-a-router --external-gateway external-network
A router add subnet lab-sec-a-router lab-sec-a-subnet
```

**Tenant B :**

```bash
B network create lab-sec-b-net
B subnet create lab-sec-b-subnet --network lab-sec-b-net --subnet-range 10.60.2.0/24 --dns-nameserver 192.168.56.20
B router create lab-sec-b-router
B router set lab-sec-b-router --external-gateway external-network
B router add subnet lab-sec-b-router lab-sec-b-subnet
```

| Commande | Rôle |
|---|---|
| `network create` | Un réseau virtuel privé au projet |
| `subnet create … --subnet-range` | La plage d’adresses ; `--dns-nameserver` : le DNS du lab, pour que les VM résolvent les noms |
| `router create` puis `router set --external-gateway external-network` | Un routeur, avec une patte sur le réseau externe : sortie vers l’extérieur |
| `router add subnet` | La seconde patte, sur le réseau privé : passerelle des VM |

**Vérifier la visibilité :**

```bash
A network list -c Name
B network list -c Name
```

✅ **Attendu :** A voit `lab-sec-a-net` et `external-network`, **pas** `lab-sec-b-net` ; et réciproquement. Ni l’un ni l’autre ne voit `demo-network`.

## 5.4 Les groupes de sécurité : micro-segmentation du tenant A

### Le piège du groupe `default`

```bash
A security group rule list default -c Direction -c "IP Protocol" -c "Remote IP Prefix" -c "Remote Security Group"
```

✅ **Attendu :** des règles **entrantes** dont la source est le groupe `default` lui-même (`Remote Security Group` = l’identifiant de `default`), et des règles **sortantes** vers tout (`0.0.0.0/0`, `::/0`).

**Comment le lire :** toutes les VM placées dans `default` peuvent se joindre librement **entre elles**, sur tous les ports. C’est pratique, mais c’est l’inverse de la micro-segmentation : on n’utilise pas `default` pour les VM de la démo.

### Le groupe des serveurs web

```bash
A security group create sg-web --description "Serveurs web : SSH et ping depuis le poste d'administration"
A security group rule create sg-web --ingress --protocol icmp --remote-ip 192.168.57.1/32
A security group rule create sg-web --ingress --protocol tcp --dst-port 22 --remote-ip 192.168.57.1/32
```

**Comment le lire :** entrées autorisées : le **ping** et le **SSH**, depuis la **seule** adresse `192.168.57.1`, le PC Windows sur le réseau externe. Tout le reste entrant est refusé : un groupe de sécurité **refuse par défaut**.

### Le groupe des bases de données

```bash
A security group create sg-db --description "Base : joignable seulement depuis sg-web"
A security group rule create sg-db --ingress --protocol icmp --remote-group sg-web
A security group rule create sg-db --ingress --protocol tcp --dst-port 5432 --remote-group sg-web
A security group rule list sg-db -c Direction -c "IP Protocol" -c "Port Range" -c "Remote Security Group"
```

**La commande, mot par mot :** `--remote-group sg-web` = la source autorisée n’est pas une plage d’adresses, mais **toute VM membre du groupe `sg-web`**, quelle que soit son adresse. Le port `5432` est celui de PostgreSQL : l’essai de connexion se fera au ping, mais la règle montre ce qu’on ouvrirait en vrai.

**Pourquoi c’est de la micro-segmentation :** la règle suit le **rôle** de la VM, pas son adresse. Une nouvelle VM web hérite du droit en entrant dans `sg-web` ; une VM retirée du groupe le perd aussitôt.

### Le groupe du tenant B

```bash
B security group create sg-b-app --description "Appli B : ping depuis le poste d'administration"
B security group rule create sg-b-app --ingress --protocol icmp --remote-ip 192.168.57.1/32
```

## 5.5 Les VM et les IP flottantes

Une clé SSH par tenant, gardée sur Lab-Sunbeam :

```bash
A keypair create lab-sec-a-key --private-key ~/lab-sec-a.pem
chmod 600 ~/lab-sec-a.pem
```

Les VM, chacune **seulement** dans son groupe (pas dans `default`) :

```bash
A server create lab-sec-a-web --image ubuntu --flavor m1.tiny --network lab-sec-a-net --security-group sg-web --key-name lab-sec-a-key --wait -c status -f value
A server create lab-sec-a-db  --image ubuntu --flavor m1.tiny --network lab-sec-a-net --security-group sg-db  --key-name lab-sec-a-key --wait -c status -f value
B server create lab-sec-b-app --image ubuntu --flavor m1.tiny --network lab-sec-b-net --security-group sg-b-app --wait -c status -f value
```

Une IP flottante pour `a-web` et pour `b-app`, **pas** pour `a-db` :

```bash
FIP_A=$(A floating ip create external-network -f value -c floating_ip_address)
A server add floating ip lab-sec-a-web "$FIP_A"
FIP_B=$(B floating ip create external-network -f value -c floating_ip_address)
B server add floating ip lab-sec-b-app "$FIP_B"
DB_IP=$(A server show lab-sec-a-db -f json -c addresses | python3 -c "import sys,json; print(list(json.load(sys.stdin)['addresses'].values())[0][0])")
B_IP=$(B server show lab-sec-b-app -f json -c addresses | python3 -c "import sys,json; print(list(json.load(sys.stdin)['addresses'].values())[0][0])")
echo "a-web=$FIP_A  a-db=$DB_IP  b-app=$FIP_B (prive $B_IP)"
```

✅ **Attendu :** trois VM `ACTIVE`, deux IP flottantes en `192.168.57.x`, `a-db` en `10.60.1.x`, `b-app` en `10.60.2.x`.

⚠ **Attendre 5 à 6 minutes** que les VM finissent de démarrer (guide `3-sunbeam-cli/DEMOS-ENTREPRISE.md`, démo 3 : le démarrage est lent sur ce lab). `A console log show lab-sec-a-web | grep "Cloud-init .* finished"` indique quand `a-web` est prête.

## 5.6 Prouver le cloisonnement

Chaque essai a un résultat attendu. Les essais depuis Lab-Sunbeam passent par le NAT de VirtualBox, donc par le PC : ils arrivent avec l’adresse `192.168.57.1` (constaté le 17 octobre, guide `3-sunbeam-cli/DEMOS-ENTREPRISE.md`, démo 5).

| # | Essai | Commande (sur Lab-Sunbeam) | Attendu | Ce qui le décide |
|---|---|---|---|---|
| 1 | PC → a-web, ping | `ping -c 2 $FIP_A` | ✅ répond | `sg-web` autorise l’ICMP depuis `192.168.57.1` |
| 2 | PC → a-web, SSH | `ssh -i ~/lab-sec-a.pem -o StrictHostKeyChecking=accept-new ubuntu@$FIP_A hostname` | ✅ `lab-sec-a-web` | `sg-web` autorise le port 22 |
| 3 | a-web → a-db, ping | `ssh -i ~/lab-sec-a.pem ubuntu@$FIP_A "ping -c 2 -W 2 $DB_IP"` | ✅ répond | `sg-db` autorise l’ICMP depuis le groupe `sg-web` |
| 4 | PC → a-db | — | ⛔ impossible | `a-db` n’a **pas** d’IP flottante : rien ne mène à elle depuis l’extérieur |
| 5 | a-web → b-app, adresse privée | `ssh -i ~/lab-sec-a.pem ubuntu@$FIP_A "ping -c 2 -W 2 $B_IP"` | ⛔ pas de réponse | Aucune route entre `10.60.1.0/24` et `10.60.2.0/24` : réseaux de deux projets |
| 6 | a-web → b-app, IP flottante | `ssh -i ~/lab-sec-a.pem ubuntu@$FIP_A "ping -c 2 -W 2 $FIP_B"` | ⛔ pas de réponse | Le paquet sort par le routeur de A avec l’adresse externe de ce routeur, qui n’est pas `192.168.57.1` : `sg-b-app` le refuse |
| 7 | PC → b-app | `ping -c 2 $FIP_B` | ✅ répond | `sg-b-app` autorise le PC |

**Retirer un droit et le voir disparaître.** Supprimer la règle ICMP de `sg-db`, puis rejouer l’essai 3 :

```bash
RULE=$(A security group rule list sg-db --protocol icmp -f value -c ID)
A security group rule delete "$RULE"
ssh -i ~/lab-sec-a.pem ubuntu@$FIP_A "ping -c 2 -W 2 $DB_IP"
```

✅ **Attendu :** `100% packet loss`. Le changement s’applique **immédiatement**, sans redémarrer la VM. Remettre la règle (`A security group rule create sg-db --ingress --protocol icmp --remote-group sg-web`) et vérifier que le ping revient.

**L’isolation, vue de l’administrateur :**

```bash
openstack server list --all-projects -c Name -c Networks --name lab-sec
openstack security group list --all-projects -c Name -c Project | grep -E "sg-|default"
```

✅ **Attendu :** les trois VM, et pour chaque projet ses groupes de sécurité **et** son propre groupe `default` : un `default` par projet.

## 5.7 Partager volontairement, sans tout ouvrir : le RBAC réseau

**Situation :** A doit offrir un réseau à B (un réseau de service commun, par exemple), sans le partager avec **tout** le cloud.

```bash
A network create lab-sec-a-partage
A subnet create lab-sec-a-partage-subnet --network lab-sec-a-partage --subnet-range 10.60.9.0/24
B_PROJECT=$(openstack project show lab-sec-b --domain lab-sec -f value -c id)
A network rbac create --target-project "$B_PROJECT" --action access_as_shared --type network lab-sec-a-partage
B network list -c Name
A network rbac list
```

⚠ Le quota de réseaux du tenant A est à 1 (5.2) : le relever **avant** ces commandes (`openstack quota set "$(openstack project show lab-sec-a --domain lab-sec -f value -c id)" --networks 2`), sinon la création est refusée (`Quota exceeded`).

| Morceau | Rôle |
|---|---|
| `network rbac create` | Une règle de partage d’un objet réseau |
| `--target-project` | Le **seul** projet bénéficiaire (identifiant, ici celui de B) |
| `--action access_as_shared` | B peut **brancher** des VM sur ce réseau, sans pouvoir le modifier |

✅ **Attendu :** B voit maintenant `lab-sec-a-partage` ; aucun autre projet ne le voit (`openstack network list --project demo …` pour s’en convaincre).

Retirer le partage :

```bash
A network rbac delete "$(A network rbac list -f value -c ID | head -1)"
A subnet delete lab-sec-a-partage-subnet
A network delete lab-sec-a-partage
```

## 5.8 La protection des ports (anti-usurpation)

```bash
PORT=$(A port list --server lab-sec-a-web -f value -c ID)
A port show "$PORT" -c port_security_enabled -c allowed_address_pairs -c fixed_ips -c security_group_ids
```

✅ **Attendu :** `port_security_enabled: True`, `allowed_address_pairs: []`.

**Comment le lire :** avec la sécurité de port, une VM ne peut émettre qu’avec **son** adresse et **son** adresse MAC : elle ne peut pas usurper l’adresse d’une autre VM. Les groupes de sécurité ne s’appliquent que si elle est active. Une adresse supplémentaire (adresse virtuelle d’une grappe, par exemple) se déclare explicitement avec `--allowed-address`, plutôt que de désactiver la protection.

Repérer les ports sans protection, en audit :

```bash
openstack port list --long -f json | python3 -c "import sys,json; [print(p['ID']) for p in json.load(sys.stdin) if p.get('Port Security') is False]"
```

⚠ **à vérifier :** le nom de la colonne (`Port Security`) dans la sortie de `port list --long -f json` du client 7.5.0.

## Nettoyage

Dans l’ordre : VM, IP flottantes, routeurs, réseaux, groupes, clés, puis l’identité.

```bash
A server delete lab-sec-a-web lab-sec-a-db --wait
B server delete lab-sec-b-app --wait
A floating ip delete "$FIP_A"; B floating ip delete "$FIP_B"
A router remove subnet lab-sec-a-router lab-sec-a-subnet; A router delete lab-sec-a-router
B router remove subnet lab-sec-b-router lab-sec-b-subnet; B router delete lab-sec-b-router
A network delete lab-sec-a-net; B network delete lab-sec-b-net
A security group delete sg-web sg-db; B security group delete sg-b-app
A keypair delete lab-sec-a-key; rm -f ~/lab-sec-a.pem
for t in a b; do openstack project delete lab-sec-$t --domain lab-sec; openstack user delete lab-sec-$t-user --domain lab-sec; done
openstack domain set --disable lab-sec; openstack domain delete lab-sec
openstack server list --all-projects --name lab-sec
openstack floating ip list -c "Floating IP Address" -c Project
```

✅ **Attendu :** plus aucune VM `lab-sec`, et aucune IP flottante des projets supprimés. ⚠ Supprimer les ressources **avant** les projets : une VM ou un réseau survit à son projet et devient orphelin.

## À retenir

- L’isolation entre tenants tient à **quatre** niveaux : visibilité par projet, réseaux privés distincts, routage par projet, groupes de sécurité.
- Un réseau **partagé à tous** casse l’isolation : on lui préfère le **RBAC réseau**, qui partage avec un projet désigné.
- La **micro-segmentation** se fait par groupes de sécurité chaînés (`--remote-group`) : les droits suivent le **rôle** de la VM, pas son adresse.
- Ne pas mettre les VM dans le groupe `default`, qui laisse ses membres se joindre librement.
- Chaque règle se **prouve** par un essai, et chaque retrait de règle aussi.

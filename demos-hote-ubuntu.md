# Jouer les démos 1 à 7 depuis un hôte Ubuntu

Les démos des dossiers `1-demo-openstack` à `7-securite` sont écrites pour un **PC Windows** qui héberge Lab-Sunbeam et Lab-Services dans VirtualBox. Ce document dit ce qui change quand le PC physique tourne sous **Ubuntu**, avec VirtualBox pour Linux et un terminal bash.

**État :** ces correspondances viennent d’une **relecture** des fichiers des démos (18 octobre 2026). Elles n’ont **pas été rejouées** sur un PC Ubuntu. Les points à vérifier en priorité sont regroupés à la fin.

**En bref :** presque tout se tape **dans les VM** (`ssh vagrant@192.168.56.10`), donc ne change pas. Côté hôte, il faut traduire les commandes PowerShell, les chemins Windows, l’approbation des certificats et les manipulations VirtualBox. **Aucune route n’est à ajouter** : l’hôte est directement sur le réseau des IP flottantes (`192.168.57.0/24`), comme sous Windows.

---

## 1. Avant toute démo : préparer l’hôte Ubuntu

Tout est décrit dans [deployment-apprenant-ubuntu.md](deployment-apprenant-ubuntu.md). Les points dont dépendent les démos :

| Prérequis | Pourquoi les démos en dépendent | Guide Ubuntu |
|---|---|---|
| `vboxnet0` = `192.168.56.1`, `vboxnet1` = **exactement** `192.168.57.1` | Les groupes de sécurité des démos 1, 2, 3 et 7.5 autorisent `192.168.57.1/32` ; le modèle Heat de la démo 3 prend `admin_cidr=192.168.57.1/32`. Une autre adresse sur `vboxnet1` = ping et SSH refusés | Étapes 4 à 6 |
| Carte 3 de Lab-Sunbeam sur `vboxnet1`, mode promiscuité **Tout autoriser** | Sans cela, pas de ping de l’IP flottante depuis le PC | Étape 6 |
| Veille désactivée : `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` | Remplace `powercfg /change standby-timeout-ac 0` (`6-labs-iam/README.md`, `7-securite/README.md`) | Étape 8.0 |
| `sudo bash ./Configurer-Lab-Sunbeam.sh` (dans le dossier du dépôt) | Remplace `Configurer-Lab-Sunbeam.ps1` : noms du lab dans `/etc/hosts`, CA du lab (`ca.crt`) approuvée pour le système | Étape 9 |
| `ca.crt` importée **dans le navigateur** | Firefox et Chrome n’utilisent pas le magasin du système : sans import, alerte de certificat sur Keycloak | Étape 9 |

Vérification rapide sur l’hôte :

```bash
ip -br addr show vboxnet0
```

```bash
ip -br addr show vboxnet1
```

```bash
getent hosts cloud.lab.test sso.lab.test
```

Attendu : `192.168.56.1/24`, `192.168.57.1/24`, puis `192.168.56.201 cloud.lab.test` et `192.168.56.20 sso.lab.test`.

## 2. Correspondances générales Windows → Ubuntu

| Dans les démos (Windows) | Sur l’hôte Ubuntu |
|---|---|
| Bloc ```` ```powershell ```` contenant `ssh vagrant@192.168.56.10` | La même commande dans un terminal |
| `"$env:USERPROFILE\.ssh\web-key"` | `~/.ssh/web-key` |
| `scp -r .\dossier …` / `scp .\fichier …` | `scp -r ./dossier …` / `scp ./fichier …` (depuis le dossier du dépôt) |
| `cd C:\Users\<toi>\Documents\formation\openstack-sunbeam` | `cd` vers le dossier où le dépôt est copié |
| `ping 192.168.57.1xx` | `ping -c 4 192.168.57.1xx` (sans `-c`, Linux ne s’arrête pas) |
| Sortie `Réponse de … : octets=32 temps=3 ms TTL=63` | Sortie `64 bytes from …: icmp_seq=1 ttl=63 time=3 ms` ; même `ttl=63` (un routeur traversé) |
| `(Invoke-WebRequest http://192.168.57.1xx/ -UseBasicParsing).Content` | `curl -s http://192.168.57.1xx/` |
| Fichier `hosts` de Windows | `/etc/hosts` (rempli par `Configurer-Lab-Sunbeam.sh`) |
| `powercfg /change standby-timeout-ac 0` | `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` |
| Adaptateur « VirtualBox Host-Only Ethernet Adapter #2 » | `vboxnet1` |
| « Navigateur de Windows » | Navigateur de l’hôte Ubuntu, mêmes adresses |

La clé créée par `ssh-keygen` sous Linux a déjà les droits `600` : pas de `chmod` à faire. Le premier `ssh` vers une instance demande de confirmer son empreinte (`yes`), comme sous Windows.

## 3. VirtualBox : interface ou ligne de commande

Les démos décrivent les menus de VirtualBox ; ils sont identiques sous Linux. Équivalents en ligne de commande, à lancer sur l’hôte (nom exact des VM : `VBoxManage list vms`) :

| Action dans les démos | Commande Ubuntu |
|---|---|
| Démarrer Lab-Services, puis Lab-Sunbeam | `VBoxManage startvm "Lab-Services" --type headless`, puis la même pour `"Lab-Sunbeam"` |
| VM figée : **Machine → Fermer → Éteindre la machine**, puis **Démarrer** (`4-juju-cli/DEPANNAGE-SERVICES.md`, `7-securite/README.md`) | `VBoxManage controlvm "Lab-Services" poweroff`, puis `startvm` |
| Instantané, **VM éteinte** (`7-securite`, démos 1, 2 et 6) | `VBoxManage snapshot "Lab-Sunbeam" take avant-demo` |
| Restaurer l’instantané, VM éteinte | `VBoxManage snapshot "Lab-Sunbeam" restore avant-demo` |
| État d’une VM | `VBoxManage showvminfo "Lab-Sunbeam" --machinereadable \| grep VMState=` |

`--type headless` démarre la VM sans fenêtre ; `VBoxManage startvm … --type separate` garde une console ouvrable. Jamais d’instantané VM allumée (constaté : blocage à 90 %).

---

## 4. Démo par démo

### 1-demo-openstack : instance Nginx joignable depuis le PC

Toutes les commandes `openstack` se tapent dans Lab-Sunbeam : inchangées. Côté hôte (`demo-vm-openstack.md`) :

| Étape | Windows | Ubuntu |
|---|---|---|
| 1. Créer la clé | `ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\web-key"` | `ssh-keygen -t ed25519 -f ~/.ssh/web-key` |
| 1. Copier la clé publique | `scp "$env:USERPROFILE\.ssh\web-key.pub" vagrant@192.168.56.10:~/web-key.pub` | `scp ~/.ssh/web-key.pub vagrant@192.168.56.10:~/web-key.pub` |
| Ping du routeur, puis de l’instance | `ping 192.168.57.1xx` | `ping -c 4 192.168.57.1xx` |
| 13. SSH | `ssh -i "$env:USERPROFILE\.ssh\web-key" ubuntu@192.168.57.1xx` | `ssh -i ~/.ssh/web-key ubuntu@192.168.57.1xx` |
| 13. Page web | `(Invoke-WebRequest … -UseBasicParsing).Content` | `curl -s http://192.168.57.1xx/`, ou le navigateur |

Le groupe de sécurité autorise `192.168.57.1/32` : c’est `vboxnet1`, donc l’hôte Ubuntu. Le texte `--description "SSH et HTTP depuis Windows"` est purement descriptif.

Dépannage « Ping du routeur impossible depuis Windows » : sous Ubuntu, vérifier `ip -br addr show vboxnet1` et que la carte 3 de Lab-Sunbeam est sur `vboxnet1` en promiscuité « Tout autoriser ».

### 2-trace : traceur des appels OpenStack

Le traceur (`openstack-trace.py`, `start.sh`, `stop.sh`, `scenario-demo.sh`) tourne **dans Lab-Sunbeam** ; l’hôte ne sert qu’à copier le dossier, à piloter en SSH et à regarder le navigateur. Les scripts sont en fins de ligne LF (`.gitattributes`).

| Étape (`PAS-A-PAS.md`, `README.md`) | Ubuntu |
|---|---|
| Fenêtre A « PowerShell » | Un terminal, dans le dossier du dépôt |
| `scp -r .\2-trace vagrant@192.168.56.10:~/` | `scp -r ./2-trace vagrant@192.168.56.10:~/` |
| Clé `web-key` (création, copie) | Comme la démo 1 |
| Navigateur : `http://192.168.56.10:8099/` | Inchangé |
| Étape 7 : `ping`, `ssh -i "$env:USERPROFILE\.ssh\web-key" …` | `ping -c 4 …`, `ssh -i ~/.ssh/web-key ubuntu@192.168.57.1xx` |

`scenario-demo.sh` affiche, à la fin, des conseils en syntaxe PowerShell (« Sous Windows : ssh-keygen … `$env:USERPROFILE` … », « ssh -i "$env:USERPROFILE\.ssh\web-key" … »). Sous Ubuntu, utiliser les commandes du tableau ci-dessus ; le script fonctionne par ailleurs tel quel.

### 3-sunbeam-cli : commandes Sunbeam et démos entreprise

Tout se tape dans Lab-Sunbeam, y compris le `ssh -i …/sunbeam ubuntu@…` affiché par `sunbeam launch` (la clé est dans la VM). Côté hôte :

| Passage | Ubuntu |
|---|---|
| `README.md` : « fichier `hosts` de Windows », « `Configurer-Lab-Sunbeam.ps1` inscrit dans le fichier `hosts` » | `/etc/hosts`, rempli par `Configurer-Lab-Sunbeam.sh` ; mêmes adresses (`.201`, `.202`, `.203`) |
| `README.md` : « Vérifier depuis Windows » `ping 192.168.57.199` | `ping -c 4 192.168.57.199` ; attendu `ttl=63` |
| `DEMOS-ENTREPRISE.md` étape 4 : `ping 192.168.57.122` | `ping -c 4 192.168.57.122` ; la sortie attendue est au format Linux (voir § 2) |
| `DEMOS-ENTREPRISE.md` : `ping 192.168.57.129` depuis Windows | `ping -c 4 192.168.57.129` |
| Horizon : `http://cloud.lab.test:80/openstack-horizon` | Inchangé, dans le navigateur ; vérifiable par `curl -s -o /dev/null -w '%{http_code}\n' http://cloud.lab.test/openstack-horizon/auth/login/` (attendu `200`) |
| « Pas de ping depuis Windows après 10 minutes » | `ip -br addr show vboxnet1` doit montrer `192.168.57.1/24` |

Le modèle Heat (`admin_cidr=192.168.57.1/32`) reste valable si `vboxnet1` porte bien `.1`.

Le diff du manifeste (`DEMOS-ENTREPRISE.md`) montre `nameservers: 192.168.1.254,192.168.56.20` : `192.168.1.254` est le DNS du réseau du PC de référence. Sur un autre PC, cette valeur peut différer : ce n’est pas une erreur.

`DEMOS-ENTREPRISE.md` indique qu’un ping lancé **depuis Lab-Sunbeam** vers une IP flottante sort par le NAT de VirtualBox, donc par le PC, avec l’adresse `192.168.57.1`. Ce comportement est à vérifier sous Linux (voir § 6).

### 4-juju-cli : Juju et pannes

Toutes les pannes se provoquent **dans** Lab-Sunbeam (`juju config`, `snap stop`, `sed` dans un pod) : aucune ne passe par VirtualBox. Côté hôte :

| Passage | Ubuntu |
|---|---|
| `ssh vagrant@192.168.56.10` (bloc `powershell`) | Inchangé |
| `DEMOS-PANNES.md`, Juju Dashboard, étape 8 : « nouveau PowerShell » | Nouveau terminal : `ssh -N -L 31666:localhost:PORT vagrant@192.168.56.10` (inchangé) |
| Navigateur : `http://localhost:31666` | Inchangé |
| Port local déjà pris : `Get-NetTCPConnection -LocalPort 31666 -State Listen` | `ss -ltnp 'sport = :31666'` (le processus est en fin de ligne), puis `kill <PID>` ; ou prendre un autre port, `-L 31667:localhost:PORT` |
| Fermer le tunnel : « fenêtre PowerShell : Ctrl+C » | `Ctrl+C` dans le terminal |
| Skyline : `http://192.168.56.20:9999` | Inchangé (adresse IP, pas de nom à résoudre) |
| `DEPANNAGE-SERVICES.md` : Lab-Services figée → **Éteindre**, puis **Démarrer** | Menu identique, ou `VBoxManage` (§ 3) |

Sous Windows, le message du port pris était `bind [127.0.0.1]:31666: Permission denied` (plage de ports réservée par Windows). Sous Linux, attendre plutôt `Address already in use`.

### 5-openstack-cli : client OpenStack et diagnostic

Tout se tape dans Lab-Sunbeam. Seul passage côté hôte : la **console graphique** (`openstack console url show --spice lab-diag`) donne une adresse `http://cloud.lab.test/openstack-nova-spiceproxy/spice_auto.html?…` à ouvrir dans le navigateur de l’hôte. Elle fonctionne si `/etc/hosts` contient `cloud.lab.test` (§ 1).

### 6-labs-iam : identité, cloisonnement, moindre privilège

Les cinq TP se tapent dans Lab-Sunbeam (`lab-env.sh`, `nettoyage.sh`, `os_as` : bash, déjà Linux). Côté hôte :

| Passage | Ubuntu |
|---|---|
| `README.md` : `powercfg /change standby-timeout-ac 0` | `systemctl mask …` (§ 1) |
| `README.md` : `scp -r .\labs-iam vagrant@192.168.56.10:~/` | `scp -r ./6-labs-iam vagrant@192.168.56.10:~/labs-iam` |
| `02-domaines.md` : « Depuis Windows, ouvre `http://cloud.lab.test/openstack-horizon` » | Navigateur de l’hôte, même adresse |

Le dossier du dépôt s’appelle `6-labs-iam`, mais les TP attendent `~/labs-iam` dans la VM : la commande ci-dessus le copie sous ce nom. Si `~/labs-iam` existe déjà dans la VM, `scp -r` le copierait **dedans** (`~/labs-iam/6-labs-iam`) : le supprimer d’abord (`rm -rf ~/labs-iam` dans la VM).

### 7-securite : TLS, fédération Keycloak, mots de passe, rotation, réseau, audit

C’est la démo qui change le plus côté hôte : **certificats** et **navigateur**.

**Avant la séance (`README.md`) :** veille (§ 1), démarrage et instantané des VM (§ 3), CA du lab approuvée par `Configurer-Lab-Sunbeam.sh` au lieu de `Configurer-Lab-Sunbeam.ps1`, puis importée dans le navigateur.

**Démo 1, TLS (`01-chiffrement-tls.md`), « Faire confiance à la CA sur Windows » :** après `sunbeam enable tls`, Horizon passe en `https://cloud.lab.test/…`, signé par `internal-ca`. Sous Ubuntu :

```bash
scp vagrant@192.168.56.10:.config/openstack/ca_bundle.pem ./sunbeam-ca.crt
```

```bash
sudo cp sunbeam-ca.crt /usr/local/share/ca-certificates/sunbeam-internal-ca.crt
```

```bash
sudo update-ca-certificates
```

`update-ca-certificates` doit annoncer `1 added`. Comme pour `ca.crt`, importer aussi `sunbeam-ca.crt` **dans le navigateur** (Firefox : Paramètres → Vie privée et sécurité → Certificats → Afficher les certificats → Autorités → Importer, cocher « identifier des sites web » ; Chrome / Chromium : `chrome://settings/certificates` → Autorités → Importer, même case à cocher). Le chemin `.config/openstack/ca_bundle.pem` est celui de `OS_CACERT` dans `~/admin-openrc` ; s’il diffère, prendre celui que montre l’étape 1.3 (`grep OS_CACERT ~/admin-openrc`).

**Retour arrière de la démo 1** (au lieu de `certlm.msc`) :

```bash
sudo rm /usr/local/share/ca-certificates/sunbeam-internal-ca.crt
```

```bash
sudo update-ca-certificates --fresh
```

Puis supprimer `internal-ca` dans les autorités du navigateur.

**Démo 2, fédération Keycloak (`02-federation-keycloak.md`) :**

| Étape | Windows | Ubuntu |
|---|---|---|
| 2.2 Copier la CA du lab | `scp .\ca.crt vagrant@192.168.56.10:~/` | `scp ./ca.crt vagrant@192.168.56.10:~/` |
| Console Keycloak | `https://sso.lab.test:8443/admin/`, « la CA du lab est approuvée sur Windows » | Même adresse ; `ca.crt` doit être importée **dans le navigateur** |
| Script d’extraction | `scp .\lab-sunbeam\scripts\extract-sp.py vagrant@192.168.56.10:~/` | `scp ./lab-sunbeam/scripts/extract-sp.py vagrant@192.168.56.10:~/` |
| Récupérer `keystone-sp.xml` (« PowerShell ») | `scp vagrant@192.168.56.10:~/keystone-sp.xml .` | Inchangé ; le fichier est ensuite importé dans Keycloak depuis le navigateur de l’hôte |
| Règles de correspondance | `scp .\lab-sunbeam\mapping.json vagrant@192.168.56.10:~/lab/` | `scp ./lab-sunbeam/mapping.json vagrant@192.168.56.10:~/lab/` |
| 2.7 Connexion SSO dans le navigateur | Prérequis : CA de Sunbeam importée sur Windows | Les **deux** CA dans le navigateur (`ca.crt` et `sunbeam-ca.crt`), et les deux noms dans `/etc/hosts` |
| Script de test | `scp .\6-securite\outils\test-sso-saml.py vagrant@192.168.56.10:~/` | `scp ./7-securite/outils/test-sso-saml.py vagrant@192.168.56.10:~/` |

`test-sso-saml.py` tourne dans Lab-Sunbeam (`python3`, `requests` présent dans la VM) : inchangé. L’étape `tr -d '\r' < ~/ca.crt` reste sans effet nuisible sous Linux.

**Démo 3, mots de passe (`03-mots-de-passe-anssi.md`) :** console Keycloak dans le navigateur de l’hôte, comme la démo 2. Si l’essai de connexion se fait depuis le navigateur de l’hôte (et non par `test-sso-saml.py` sur Lab-Sunbeam), les événements Keycloak montrent l’adresse de l’hôte, `192.168.56.1`, au lieu de `192.168.56.10`.

**Démo 4, rotation des secrets :** tout dans Lab-Sunbeam. Inchangée.

**Démo 5, micro-segmentation (`05-reseau-micro-segmentation.md`) :** les règles autorisent `192.168.57.1/32`, « le PC Windows » : sous Ubuntu, c’est `vboxnet1`. Les essais se font **depuis Lab-Sunbeam** et passent par le NAT de VirtualBox, donc par l’hôte, d’où l’adresse source `192.168.57.1`. Ce passage par le NAT est le point le plus incertain sous Linux (voir § 6). En cas d’échec, jouer les mêmes `ping -c 2 $FIP_A` et `ping -c 2 $FIP_B` **depuis l’hôte** : ils ont le même sens (le PC est autorisé sur a-web, pas en SSH sur b-app).

**Démo 6, audit (`06-audit-tracabilite.md`) :** Grafana à `http://192.168.56.205/observability-grafana` dans le navigateur de l’hôte (adresse IP, HTTP : ni `/etc/hosts` ni CA). Le mot de passe se lit dans la VM (`cat ~/lab-sec-grafana.pw`, hors écran projeté).

---

## 5. Erreurs relevées dans les fichiers d’origine (aussi sous Windows)

Relevées pendant cette vérification, puis **corrigées** dans les fichiers (18 octobre 2026).

| Fichier | Ligne | Problème | Correction |
|---|---|---|---|
| `6-labs-iam/README.md` | 84 | `scp -r .\labs-iam …` : le dossier s’appelle `6-labs-iam` | `scp -r .\6-labs-iam vagrant@192.168.56.10:~/labs-iam` |
| `7-securite/02-federation-keycloak.md` | 329 | `6-securite/outils/test-sso-saml.py` | `7-securite/outils/test-sso-saml.py` |
| `6-labs-iam/01-keystone-en-profondeur.md` | 309 | Renvoi à `6-securite/04-rotation-secrets.md` | `7-securite/04-rotation-secrets.md` |
| `6-labs-iam/04-roles-moindre-privilege.md` | 296 | Même renvoi | Même correction |

## 6. À vérifier sur un vrai PC Ubuntu

| Point | Pourquoi c’est incertain | Comment vérifier |
|---|---|---|
| Ping et SSH **depuis Lab-Sunbeam** vers une IP flottante (`3-sunbeam-cli/DEMOS-ENTREPRISE.md`, `7-securite/05`) | Ils passent par le NAT de VirtualBox puis par la pile réseau de l’hôte. Sous Linux, le NAT de VirtualBox envoie l’ICMP avec des sockets « ping » non privilégiées, permises par `net.ipv4.ping_group_range` | `sysctl net.ipv4.ping_group_range` sur l’hôte (Ubuntu 24.04 : `0 2147483647`, donc permis) ; puis, dans Lab-Sunbeam, `ping -c 2 <IP flottante>` et `ip route get <IP flottante>` (attendu : `via 10.0.2.2`) |
| Adresse source vue par les instances : `192.168.57.1` | L’hôte doit sortir vers `192.168.57.0/24` par `vboxnet1` | Sur l’hôte : `ip route get 192.168.57.100` doit afficher `dev vboxnet1 src 192.168.57.1` |
| Import des CA dans Firefox installé en snap | Le snap garde son propre profil ; l’import par l’interface fonctionne normalement | Ouvrir `https://sso.lab.test:8443/admin/` sans alerte |
| KVM imbriqué et EPT (démo 1, dépannage) | Dépend du processeur et de VirtualBox pour Linux, pas de Windows | `cat /sys/module/kvm_intel/parameters/ept` dans Lab-Sunbeam (guide Ubuntu, 10.3) |
| Juju Dashboard, port local | Message d’erreur différent de Windows | `ss -ltnp 'sport = :31666'` |

# Juju en entreprise : diagnostiquer de vraies pannes, installer Juju Dashboard

Ce document complète `README.md` du même dossier. Le guide de référence explique **ce que fait** chaque commande Juju ; ici, on s’en sert pour **résoudre des pannes réelles**, provoquées volontairement sur le lab, puis corrigées et vérifiées. La dernière partie installe **Juju Dashboard**, l’interface web de Juju.

**Tests :** tout a été exécuté le **17 octobre 2026** sur le lab de formation (Juju 3.6.29, contrôleur `sunbeam-controller`). Les sorties sont réelles ; les heures sont celles du test. Le lab a été remis dans son état d’origine à la fin.

| Partie | Situation en entreprise | Durée |
|---|---|---|
| 1 | Comprendre **à quoi sert** chaque commande Juju : s’informer, contrôler, diagnostiquer, corriger, vérifier | 10 min |
| 2 | Panne : « Je ne peux plus envoyer mes images » | 20 min |
| 3 | Panne : « Mes volumes restent en erreur » | 20 min |
| 4 | Donner aux équipes une vue web du cloud, en lecture seule : Juju Dashboard | 20 min |

---

## Avant de commencer

Dans **Lab-Sunbeam**, avec le compte `vagrant` :

```powershell
ssh vagrant@192.168.56.10
```

Puis, à chaque nouvelle session, se reconnecter à Juju :

```bash
sunbeam utils juju-login
```

**Ce que tu vois :** `Juju re-login complete.` Sans cela, les commandes `juju` peuvent répondre `Failed to login to Juju Controller`.

**Rôles pendant la formation :** les étapes « Provoquer la panne » sont jouées par le **formateur**, idéalement sans dire ce qu’il fait. Les participants partent du **symptôme**, comme en production.

---

## Partie 1 : à quoi sert chaque commande Juju

Juju est l’outil qui déploie, relie et surveille les composants du cloud. Pour un exploitant, ses commandes répondent à **cinq besoins**. Savoir dans quel besoin on se trouve évite de lancer des commandes au hasard.

| Besoin | Question posée | Commande | Ce qu’elle apporte |
|---|---|---|---|
| **S’informer** | Qu’y a-t-il dans le cloud ? | `juju models` | La liste des espaces de travail : `openstack` (les services), `admin/openstack-machines` (la machine), `admin/controller` (Juju lui-même) |
| | | `juju status -m MODÈLE` | Les applications, leurs unités, leur version, leurs liens |
| **Contrôler** | Tout va-t-il bien ? | `juju status -m MODÈLE --format short \| grep -v "workload:active"` | **Seulement** ce qui ne va pas : si rien ne s’affiche, tout est `active` |
| **Diagnostiquer** | Depuis quand ? | `juju show-status-log -m MODÈLE UNITÉ` | L’historique des états d’une unité, horodaté : on date le début de la panne |
| | Qu’est-ce qui a changé ? | `juju config -m MODÈLE APPLICATION` | Les réglages, avec leur **origine** : valeur par défaut ou valeur modifiée (`user`) |
| | Pourquoi ? | `juju debug-log -m MODÈLE --include UNITÉ` | Les journaux du **charm** (le programme qui pilote le service) |
| | Que se passe-t-il dans le service ? | `juju ssh -m MODÈLE UNITÉ_OU_MACHINE "commande"` | Une commande lancée **dans** le conteneur ou sur la machine |
| **Corriger** | Comment revenir à l’état voulu ? | `juju config … --reset OPTION` | Remet une option à sa valeur par défaut |
| | | `juju exec … -- "JUJU_DISPATCH_PATH=hooks/config-changed ./dispatch"` | Demande au charm de **réappliquer** sa configuration |
| | | `juju resolved UNITÉ` | Relance une étape du charm qui a échoué |
| **Vérifier** | La correction a-t-elle pris ? | Le **même test** que celui qui révélait la panne | La seule preuve qui compte : le symptôme a disparu |

### Le piège à connaître : « active » ne veut pas dire « fonctionne »

Juju affiche l’état **que le charm lui déclare**. Un charm vérifie ce qu’il sait vérifier : ses liens, sa configuration, souvent un test de santé du service. Il ne voit pas tout. Les deux pannes de ce document en sont la preuve : dans les deux cas, Juju affiche `active` alors que les utilisateurs sont bloqués.

D’où la **méthode**, toujours la même :

```text
 1. Symptôme      ce que l’utilisateur constate, reproduit par toi
 2. Localiser     quel service OpenStack est en cause         (commande openstack)
 3. Interroger    ce que Juju sait de ce service              (status, show-status-log, config)
 4. Comprendre    la cause précise                            (journaux, dans le conteneur ou sur la machine)
 5. Corriger      de préférence par Juju, qui garde la configuration cohérente
 6. Vérifier      en rejouant le test du symptôme
```

---

## Partie 2 : « Je ne peux plus envoyer mes images » (20 min)

### Provoquer la panne (formateur)

```bash
juju config -m openstack glance image-size-cap=50M
```

**Ce que fait la commande :** elle limite à 50 Mo la taille des images que Glance (le service des images) accepte. C’est une erreur réaliste : un administrateur pose une limite pour un test et l’oublie, ou confond `50M` et `50G`.

Juju applique le changement : le charm réécrit la configuration de Glance et le redémarre, en une trentaine de secondes.

Préparer aussi un fichier de 100 Mo, qui jouera le rôle d’une image :

```bash
head -c 100M /dev/urandom > ~/lab-image-100M.raw
```

### Étape 1 : constater le symptôme

Un utilisateur du projet `demo` envoie son image :

```bash
source ~/demo-openrc
openstack image create lab-image-test --disk-format raw --container-format bare --file ~/lab-image-100M.raw
```

**Ce que tu vois :**

```text
HttpException: 413: Client Error for url: http://cloud.lab.test:80/openstack-glance/v2/images/2f89737b-894d-405e-b332-5a754700ff4e/file, Request Entity Too Large
```

**Comment le lire :**

| Morceau | Signification |
|---|---|
| `413` | Code HTTP « requête trop grande » : le serveur **refuse la taille** de ce qu’on lui envoie |
| `…/openstack-glance/…/file` | C’est Glance qui refuse, au moment de l’envoi du fichier |
| `Request Entity Too Large` | La même chose en toutes lettres |

On sait **où** chercher : Glance, et une question de taille.

ℹ Si l’envoi est tenté pendant les 30 secondes qui suivent le changement, le message est différent : `Failed to contact the endpoint … The image service … does not have any supported versions`. Glance est alors en train de **redémarrer**. Attendre une minute et recommencer.

### Étape 2 : ce que Juju sait de Glance

```bash
juju status -m openstack glance --format short
```

**Ce que tu vois :**

```text
- glance/0:  (agent:idle, workload:active)
```

**Comment le lire :** le service est `active`, l’agent au repos. **Juju ne voit pas de panne.** Le charm a appliqué une configuration valide ; il ne sait pas que cette valeur bloque les utilisateurs.

On cherche donc ce qui a **changé**. D’abord **quand** :

```bash
juju show-status-log -m openstack glance/0 | tail -4
```

**Ce que tu vois :**

```text
16 Oct 2026 12:41:53Z  juju-unit  idle
17 Oct 2026 10:32:56Z  juju-unit  executing  running config-changed hook
17 Oct 2026 10:33:25Z  workload   active
17 Oct 2026 10:33:29Z  juju-unit  idle
```

**Comment le lire :** rien depuis la veille, puis aujourd’hui à **10:32:56** l’étape `config-changed` : quelqu’un a **modifié la configuration** de Glance. Si les utilisateurs se plaignent depuis 10 h 33, on tient la piste.

Puis **quoi** : lister uniquement les options **modifiées** par un humain ou par Sunbeam :

```bash
juju config -m openstack glance --format json | python3 -c "import sys,json; [print(k, '=', v.get('value')) for k,v in sorted(json.load(sys.stdin)['settings'].items()) if v.get('source')=='user']"
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `juju config … --format json` | Toutes les options de Glance, avec pour chacune sa valeur et sa **source** |
| `python3 -c "…"` | Ne garde que celles dont la source est `user`, c’est-à-dire **changées** par rapport au charm |

**Ce que tu vois :**

```text
ceph-osd-replication-count = 1
image-size-cap = 50M
```

**Comment le lire :**

| Option | Valeur | Normal ? |
|---|---|---|
| `ceph-osd-replication-count` | `1` | Oui : réglée par Sunbeam à l’installation, car le lab n’a qu’un disque Ceph |
| `image-size-cap` | `50M` | **Suspect** : une limite de 50 Mo sur les images. Une image Ubuntu pèse 600 Mo |

**Où trouver le sens d’une option :** il n’est pas à deviner, chaque charm documente ses options. La description est dans la même sortie JSON :

```bash
juju config -m openstack glance --format json | python3 -c "import sys,json; print(json.load(sys.stdin)['settings']['image-size-cap']['description'])"
```

**Ce que tu vois :**

```text
Maximum size of image a user can upload.
Example values: 500M, 500MB, 5G, 5TB.
Valid units: K, KB, M, MB, G, GB, T, TB, P, PB. If no units provided,
bytes are assumed.
Defaults to 30G for ceph storage and 1G for local storage.
.
and must be set to a value under 8EB (9223372036854775808 bytes).
```

**Comment le lire :** « taille maximale d’une image qu’un utilisateur peut envoyer », avec les unités acceptées, et la **valeur par défaut** : 30 Go quand les images sont stockées dans Ceph, ce qui est le cas sur ce lab. La même documentation figure sur la page du charm, `https://charmhub.io/glance-k8s` (adresse donnée par `juju info glance-k8s`, ligne `store-url`).

### Étape 3 : confirmer la cause

Deux confirmations indépendantes, dans le service lui-même. Il faut d’abord savoir **où** il tourne.

**Pourquoi quitter Juju ici :** Juju a dit ce qu’il sait : l’état déclaré par le charm (`active`), l’heure du changement, l’option modifiée. Mais `juju debug-log` ne contient que les journaux du **charm** (ses étapes `config-changed`, `update-status`…), pas ceux de **Glance**, où l’on verrait l’erreur de l’utilisateur. Et Juju ne montre pas le fichier de configuration que le service lit réellement. Pour **confirmer** la cause, avec la preuve vue par le service, il faut aller là où il tourne : dans Kubernetes.

**D’où vient `sudo k8s kubectl` :** le modèle `openstack` est de type Kubernetes (`juju models`, colonne `Type`). Ce Kubernetes est installé sur la machine par Sunbeam, sous la forme d’un snap nommé `k8s`. On le retrouve à deux endroits :

```bash
juju status -m admin/openstack-machines | grep "^k8s"
snap list k8s
```

```text
k8s                   1.32.13  active      1  k8s                   1.32/stable    1921  no       Ready
Name  Version   Rev   Tracking       Publisher    Notes
k8s   v1.32.13  5279  latest/stable  canonical**  classic,held
```

| Morceau de la commande | Rôle |
|---|---|
| `k8s` | La commande du snap Kubernetes de Canonical, installé par Sunbeam |
| `kubectl` | Le client standard de Kubernetes, **fourni par ce snap**. Il n’est pas installé à part sur la machine : `kubectl` seul répond `command not found` |
| `sudo` | Les accès d’administration du cluster Kubernetes sont réservés à `root` : sans `sudo`, la commande répond `You do not have enough permissions. Please re-run the command with sudo.` |
| `-n openstack` | L’espace de noms (*namespace*) Kubernetes où Juju place les pods du modèle `openstack` : même nom que le modèle |

**Où trouver le nom du pod et du conteneur :** chaque unité du modèle `openstack` est un **pod** Kubernetes, nommé comme l’unité, avec `-` à la place de `/` (`glance/0` → `glance-0`). Pour le vérifier, et connaître ses conteneurs :

```bash
sudo k8s kubectl -n openstack get pods | grep -E "NAME|glance"
sudo k8s kubectl -n openstack get pod glance-0 -o jsonpath="{.spec.containers[*].name}"; echo
```

**Ce que tu vois :**

```text
NAME                             READY   STATUS    RESTARTS       AGE
glance-0                         2/2     Running   7 (28h ago)    8d
glance-mysql-router-0            2/2     Running   18 (28h ago)   8d
charm glance-api
```

**Comment le lire :**

| Ligne / colonne | Signification |
|---|---|
| `glance-0` | **Le** pod de Glance. Il n’y en a **qu’un** : c’est l’unité `glance/0`. `juju status -m openstack glance` le confirme : `Scale 1` |
| `glance-mysql-router-0` | Un **autre** composant, pas une seconde instance de Glance : le routeur par lequel Glance accède à la base MySQL (application Juju `glance-mysql-router`) |
| `READY 2/2` | Ce n’est **pas** un nombre d’instances : c’est le nombre de **conteneurs** du pod prêts, sur le nombre total. Le pod `glance-0` en contient deux, tous deux prêts |
| `STATUS Running` | Le pod tourne |
| `RESTARTS 7 (32h ago)` | Ses conteneurs ont redémarré 7 fois depuis sa création, la dernière il y a 32 heures : ce sont les redémarrages du lab, pas une panne en cours |
| `AGE 8d` | Le pod existe depuis l’installation du cloud, il y a 8 jours |
| `charm glance-api` | Les noms des deux conteneurs : `charm`, où tourne le charm (ce que voit Juju), et **`glance-api`**, où tourne le service lui-même |

Les journaux et la configuration **du service** sont donc dans le conteneur `glance-api` : c’est lui qu’on vise avec `-c glance-api`.

Dans le **journal de Glance** :

```bash
sudo k8s kubectl -n openstack logs glance-0 -c glance-api --since=10m | grep -i "too large" | sed -E 's/.*\] //' | head -2
```

**Ce que tu vois :**

```text
Failed to store image 2f89737b-894d-405e-b332-5a754700ff4e Store Exception The provided image is too large.: glance.common.exception.ImageSizeLimitExceeded: The provided image is too large.
The incoming image is too large: The provided image is too large.: glance.common.exception.ImageSizeLimitExceeded: The provided image is too large.
```

**Comment le lire :** `ImageSizeLimitExceeded` = « limite de taille d’image dépassée ». Même identifiant d’image que dans l’erreur de l’utilisateur.

Dans le **fichier de configuration** du service. **Où le trouver :** les services OpenStack rangent leur configuration dans `/etc/NOM_DU_SERVICE/` ; on liste le dossier :

```bash
sudo k8s kubectl -n openstack exec glance-0 -c glance-api -- ls /etc/glance
```

```text
api_audit_map.conf
glance-api-paste.ini
glance-api.conf
glance-api.d
glance-cache.conf
…
```

Le fichier de l’API, celle qui reçoit les envois d’images, est `glance-api.conf`. Le nom du réglage dans ce fichier (`image_size_cap`) est celui de l’option Juju, avec `_` au lieu de `-` : on le cherche directement.

```bash
sudo k8s kubectl -n openstack exec glance-0 -c glance-api -- grep -n image_size_cap /etc/glance/glance-api.conf
```

**Ce que tu vois :**

```text
13:image_size_cap = 52428800
```

**Comment le lire :** 52 428 800 octets = 50 × 1 024 × 1 024 = **50 Mo**. Le charm a bien traduit l’option Juju dans le fichier du service.

**Diagnostic :** une option Juju modifiée à 10:32:56 limite les images à 50 Mo ; toute image plus grande est refusée.

### Étape 4 : corriger

```bash
juju config -m openstack glance --reset image-size-cap
```

**Pourquoi par Juju, et pas en modifiant le fichier dans le conteneur :** le charm réécrit ce fichier à chaque reconfiguration. Une correction à la main serait **écrasée**, et la valeur `50M` reviendrait. On corrige donc la **source** : l’option Juju.

`--reset` remet la valeur par défaut du charm, plutôt qu’une valeur choisie au hasard.

Suivre l’application :

```bash
watch -n 5 "juju status -m openstack glance --format short"
```

**Ce que tu vois :** `agent:executing` quelques secondes, puis `agent:idle, workload:active` (35 secondes sur le lab). `Ctrl+C` pour sortir.

### Étape 5 : vérifier que la correction a pris

À trois niveaux, du plus technique au plus concret :

```bash
juju config -m openstack glance image-size-cap
sudo k8s kubectl -n openstack exec glance-0 -c glance-api -- grep -n image_size_cap /etc/glance/glance-api.conf
openstack image create lab-image-test --disk-format raw --container-format bare --file ~/lab-image-100M.raw -c name -c status -c size
```

**Ce que tu vois :**

```text

13:image_size_cap = 32212254720
+--------+----------------+
| Field  | Value          |
+--------+----------------+
| name   | lab-image-test |
| size   | 104857600      |
| status | active         |
+--------+----------------+
```

**Comment le lire :**

| Niveau | Résultat | Signification |
|---|---|---|
| Juju | ligne vide | L’option n’a plus de valeur : le charm utilise la sienne par défaut |
| Fichier du service | `32212254720` | 30 Gio : la valeur par défaut annoncée par la description de l’option (« 30G for ceph storage ») |
| **Test utilisateur** | `active`, `104857600` octets | L’image de 100 Mo est acceptée : **le symptôme a disparu** |

### Nettoyer

```bash
openstack image delete lab-image-test
rm -f ~/lab-image-100M.raw
```

### À retenir

- Juju disait `active` : **la panne n’était pas visible** dans son état. Elle l’était dans son **historique** (`show-status-log`) et sa **configuration** (`juju config`, source `user`).
- La première question devant une panne soudaine : **qu’est-ce qui a changé, et quand ?**
- On corrige par Juju, la source de la configuration, jamais dans le conteneur.
- Juju **ne valide pas** toutes les valeurs : sur le lab, `juju config -m openstack keystone log-level=VERBOSE` (une valeur qui n’existe pas) a été acceptée sans erreur ni changement d’état. Une faute de frappe peut donc passer inaperçue.

---

## Partie 3 : « Mes volumes restent en erreur » (20 min)

### Provoquer la panne (formateur)

```bash
sudo snap stop cinder-volume
```

**Ce que fait la commande :** elle arrête, sur la machine, le service qui crée les volumes (disques) des VM. C’est réaliste : un administrateur arrête un service pour une maintenance et oublie de le relancer, ou un script le coupe.

### Étape 1 : constater le symptôme

```bash
source ~/demo-openrc
openstack volume create lab-vol-panne --size 1 -c status -f value
sleep 30
openstack volume show lab-vol-panne -c status -f value
```

**Ce que tu vois :**

```text
creating
error
```

**Comment le lire :** la création est acceptée (`creating`), puis échoue (`error`). Le volume ne dit pas pourquoi.

### Étape 2 : localiser, côté OpenStack

Le message adressé à l’utilisateur :

```bash
openstack --os-volume-api-version 3.3 volume message list -c "User Message" -c "Created At" | head -4
```

**Ce que tu vois :**

```text
+-------------------------------------------------------------------------+----------------------------+
| User Message                                                            | Created At                 |
+-------------------------------------------------------------------------+----------------------------+
| schedule allocate volume:Could not find any available weighted backend. | 2026-10-17T10:41:53.000000 |
```

**Comment le lire :** au moment de choisir où créer le volume (`schedule allocate volume`), Cinder n’a trouvé **aucun stockage disponible** (`backend`).

L’état des services de stockage, avec le compte administrateur :

```bash
source ~/admin-openrc
openstack volume service list -c Binary -c Host -c State -c "Updated At"
```

**Ce que tu vois :**

```text
| cinder-scheduler | cinder-0                   | up   | … |
| cinder-volume    | sunbeam@cinder-volume-ceph | down | … |
```

**Comment le lire :** le planificateur fonctionne, mais `cinder-volume`, sur la machine `sunbeam`, est **`down`** : il ne donne plus signe de vie. La colonne `Updated At` indique **depuis quand** (son dernier signal).

### Étape 3 : ce que Juju sait

```bash
juju status -m admin/openstack-machines cinder-volume --format short
sunbeam cluster list -f yaml | grep storage
```

**Ce que tu vois :**

```text
- cinder-volume/0: 10.0.2.15 (agent:idle, workload:active)
      storage: active
```

**Comment le lire :** **ni Juju ni Sunbeam ne voient la panne.** Sur le lab, ils ont affiché `active` pendant **plus de 8 minutes** après l’arrêt du service, y compris après le contrôle périodique du charm :

```bash
juju show-status-log -m admin/openstack-machines cinder-volume/0 | tail -1
```

```text
17 Oct 2026 10:42:08Z  workload   active
```

Le charm a vérifié à 10:42:08 et a conclu `active` : il contrôle ses liens et sa configuration, **pas** que le service tourne. C’est exactement le cas où il faut descendre **sur la machine**.

### Étape 4 : comprendre, sur la machine

**Où trouver la machine et le nom du service :** la colonne `Machine` de `juju status` dit sur quelle machine tourne l’unité.

```bash
juju status -m admin/openstack-machines cinder-volume
```

```text
Unit                     Workload  Agent  Machine  Public address  Ports  Message
cinder-volume/0*         active    idle   0        10.0.2.15
  cinder-volume-ceph/0*  active    idle            10.0.2.15
```

`cinder-volume/0` est sur la machine **`0`**, la VM Lab-Sunbeam. Sur une machine, les composants de Sunbeam sont installés sous forme de **snaps** (paquets Ubuntu) ; `snap list` donne leur nom exact :

```bash
juju ssh -m admin/openstack-machines 0 "snap list | grep -E 'Name|cinder'"
```

```text
Name                  Version                Rev    Tracking       Publisher    Notes
cinder-volume         2024.1-4c94ab3f        137    2024.1/stable  canonical**  held
```

Le snap s’appelle `cinder-volume`. `held` = ses mises à jour automatiques sont suspendues : elles sont pilotées par le charm, pas par le système.

`juju ssh` exécute des commandes sur la machine `0` :

```bash
juju ssh -m admin/openstack-machines 0 "snap services cinder-volume"
```

**Ce que tu vois :**

```text
Service                      Startup  Current   Notes
cinder-volume.cinder-volume  enabled  inactive  -
```

**Comment le lire :** `Startup enabled` = le service doit démarrer avec la machine ; `Current inactive` = il est **arrêté**. On tient la cause directe. Reste à savoir **pourquoi** il est arrêté : panne ou geste humain ?

L’historique des opérations sur les snaps :

```bash
juju ssh -m admin/openstack-machines 0 "snap changes | tail -2"
```

```text
265  Done    today at 06:37 UTC      today at 06:37 UTC      Running service command
266  Done    today at 10:37 UTC      today at 10:37 UTC      Running service command
```

```bash
juju ssh -m admin/openstack-machines 0 "snap change 266"
```

```text
Status  Spawn               Ready               Summary
Done    today at 10:37 UTC  today at 10:37 UTC  Run service command "stop" for services ["cinder-volume"] of snap "cinder-volume"
```

Et le journal du système :

```bash
juju ssh -m admin/openstack-machines 0 "sudo journalctl -u snap.cinder-volume.cinder-volume --since '-1h' --no-pager | tail -3"
```

```text
Oct 17 10:37:33 sunbeam systemd[1]: Stopping snap.cinder-volume.cinder-volume.service - Service for snap application cinder-volume.cinder-volume...
Oct 17 10:37:33 sunbeam systemd[1]: snap.cinder-volume.cinder-volume.service: Deactivated successfully.
Oct 17 10:37:33 sunbeam systemd[1]: Stopped snap.cinder-volume.cinder-volume.service - Service for snap application cinder-volume.cinder-volume.
```

**Comment le lire :**

| Source | Ce qu’elle dit |
|---|---|
| `snap change 266` | À 10:37, une commande **`stop`** a été lancée sur le service `cinder-volume` |
| `journalctl` | À 10:37:33, le service s’est arrêté **proprement** (`Deactivated successfully`), sans plantage |

**Diagnostic :** le service n’est pas tombé en panne, **il a été arrêté** volontairement à 10:37:33. Sans lui, Cinder n’a plus de stockage, et les volumes partent en erreur.

### Étape 5 : corriger, par le charm

On pourrait taper `sudo snap start cinder-volume`. On préfère **demander au charm** de réappliquer sa configuration : c’est lui qui sait dans quel état le service doit être, et il corrige au passage tout autre écart.

```bash
juju exec -m admin/openstack-machines --unit cinder-volume/0 -- "JUJU_DISPATCH_PATH=hooks/config-changed ./dispatch"
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `juju exec --unit cinder-volume/0 --` | Exécute une commande dans le contexte du charm de cette unité |
| `JUJU_DISPATCH_PATH=hooks/config-changed` | Fait croire au charm que sa configuration vient de changer |
| `./dispatch` | Lance le charm, qui refait toutes ses vérifications et remet le service dans l’état voulu |

Puis :

```bash
juju ssh -m admin/openstack-machines 0 "snap services cinder-volume"
```

**Ce que tu vois** (20 secondes après) :

```text
Service                      Startup  Current  Notes
cinder-volume.cinder-volume  enabled  active   -
```

Le charm a **redémarré** le service.

### Étape 6 : vérifier que la correction a pris

```bash
openstack volume service list -c Binary -c State -f value
openstack volume backend pool list -f value
source ~/demo-openrc
openstack volume create lab-vol-ok --size 1 -c status -f value
sleep 15
openstack volume show lab-vol-ok -c status -f value
```

**Ce que tu vois :**

```text
cinder-scheduler up
cinder-volume up
cinder-volume@cinder-volume-ceph#cinder-volume-ceph
creating
available
```

**Comment le lire :** le service est `up` (environ une minute après le redémarrage), le stockage Ceph est de nouveau proposé (`pool`), et un nouveau volume passe `available` : **le symptôme a disparu.**

### Nettoyer

```bash
openstack volume delete lab-vol-ok lab-vol-panne
openstack volume list
```

✅ **Attendu :** une liste vide.

### À retenir

- Juju et Sunbeam affichaient `active` **pendant toute la panne**. Le contrôle de santé de Sunbeam (`cluster list`) ne suffit pas : il faut aussi un **test du service** (`openstack volume service list`, création d’un volume).
- `snap changes` et `journalctl` distinguent un **arrêt volontaire** d’un **plantage** : c’est la différence entre « qui a fait quoi ? » et « quel bug ? ».
- Corriger **par le charm** remet le service dans l’état décidé par l’exploitation, pas seulement « redémarré ».

---

## Partie 4 : installer Juju Dashboard (20 min)

### Situation

Les équipes veulent **voir** l’état du cloud sans apprendre la ligne de commande, et le responsable veut leur donner cet accès **sans droit de modification**. Juju Dashboard est l’interface web officielle de Juju : modèles, applications, liens, journaux d’actions, secrets (sans leur contenu), machines.

### Comment ça fonctionne

```text
 Navigateur (Windows)
   │  http://localhost:31666
   ▼
 tunnel SSH ──────────────► Lab-Sunbeam : « juju dashboard » (un relais local, port tiré au hasard)
                                 │
                                 ▼
                           application « dashboard » dans le modèle admin/controller (un pod Kubernetes)
                                 │  relation « juju-dashboard »
                                 ▼
                           contrôleur Juju (ses données : modèles, statuts…)
```

Le dashboard n’est **pas exposé** sur le réseau : on y accède par un relais lancé par la commande `juju dashboard`, puis par un tunnel SSH depuis le poste. C’est voulu : seul quelqu’un qui a déjà un accès SSH au lab peut l’ouvrir.

### Avant d’installer : trouver chaque information soi-même

Pour installer un composant avec Juju, il faut répondre à **cinq questions**. Aucune réponse ne se devine : chacune se lit dans une commande. C’est la démarche à reproduire pour **n’importe quel** charm.

| Question | Où trouver la réponse | Étape |
|---|---|---|
| Quel charm installer ? | `juju find`, ou le site [charmhub.io](https://charmhub.io) | 1 |
| Dans quel modèle, et quelle variante du charm ? | `juju models`, colonne `Type` | 2 |
| Quelle branche (version) ? | `juju info CHARM`, rubrique `channels`, comparée au système du modèle (`juju status --format yaml`, rubrique `base`) | 3 |
| À quoi le relier ? | `juju info CHARM`, rubrique `relations`, comparée à celle de l’application cible | 4 |
| Comment l’utiliser ensuite ? | `juju help COMMANDE` ou `juju COMMANDE --help` | 7 |

### Étape 1 : trouver le charm

🎯 **Objectif : s’informer.** Savoir quels charms existent pour le besoin, et lequel choisir.

```bash
juju find juju-dashboard
```

**La commande, mot par mot :** `find` = chercher dans **Charmhub**, la bibliothèque publique des charms ; `juju-dashboard` = le mot cherché. Il faut que Lab-Sunbeam accède à Internet.

**Ce que tu vois :**

```text
Name                Bundle  Version  Publisher         Summary
juju-dashboard      -       78       Juju Engineering  View the real-time status of your Juju or
                                                       JAAS environment.
juju-dashboard-k8s  -       72       Juju Engineering  View the real-time status of your Juju environment.
```

**Comment le lire :**

| Colonne | Signification |
|---|---|
| `Name` | Le nom du charm, à donner à `juju deploy` |
| `Version` | La révision la plus récente publiée |
| `Publisher` | L’éditeur : `Juju Engineering` est l’équipe de Juju elle-même, une source sûre |
| `Summary` | La description courte |

Deux charms pour le même logiciel. Le suffixe **`-k8s`** est une convention de Charmhub : il désigne la variante qui s’installe dans un modèle **Kubernetes** ; sans suffixe, la variante s’installe sur une **machine**. L’étape 2 dit laquelle prendre.

Même information, avec la documentation, sur le site : `https://charmhub.io/juju-dashboard-k8s` (l’adresse est donnée par `juju info`, ligne `store-url`).

### Étape 2 : choisir le modèle, donc la variante

🎯 **Objectif : s’informer.** Le dashboard affiche les données **du contrôleur** Juju : on l’installe dans le modèle du contrôleur. Reste à savoir de quel type est ce modèle.

```bash
juju models
```

**Ce que tu vois :**

```text
Model                     Cloud/Region              Type        Status     Machines  Cores  Units  Access  Last connection
admin/controller          star-heron-k8s/localhost  kubernetes  available         0      -  1      admin   just now
openstack                 star-heron-k8s/localhost  kubernetes  available         0      -  25     admin   12 minutes ago
admin/openstack-machines  star-heron/default        manual      available         1      8  7      admin   12 minutes ago
```

**Comment le lire :** le modèle du contrôleur est `admin/controller`, et sa colonne `Type` vaut **`kubernetes`**. Il faut donc la variante **`juju-dashboard-k8s`**. (Le charm `juju-dashboard` sans suffixe conviendrait à un contrôleur installé sur des machines.)

### Étape 3 : choisir la branche

🎯 **Objectif : s’informer.** Un charm est publié en plusieurs **branches** (*channels*), chacune pour certaines versions d’Ubuntu. Il faut celle qui correspond au système du modèle.

D’abord, le système des applications du modèle du contrôleur :

```bash
juju status -m admin/controller --format yaml | grep -A2 "base:"
```

**Ce que tu vois :**

```text
    base:
      name: ubuntu
      channel: "24.04"
```

**Comment le lire :** le contrôleur tourne sur **Ubuntu 24.04**.

Ensuite, les branches du charm :

```bash
juju info juju-dashboard-k8s
```

**Ce que tu vois** (extrait) :

```text
name: juju-dashboard-k8s
publisher: Juju Engineering
store-url: https://charmhub.io/juju-dashboard-k8s
supports: ubuntu@24.04
relations:
  provides:
    dashboard: http
  requires:
    controller: juju-dashboard
    ingress: ingress
    nginx-route: nginx-route
channels: |
  0.15/stable:       72  2026-03-30  (72)  12MB  amd64  ubuntu@24.04
  0.15/candidate:    72  2026-03-30  (72)  12MB  amd64  ubuntu@24.04
  0.15/beta:         72  2026-03-30  (72)  12MB  amd64  ubuntu@24.04
  0.15/edge:         ^
  latest/stable:     16  2025-03-28  (16)  6MB  amd64  ubuntu@22.04, ubuntu@20.04
  0.16/stable:       75  2026-08-28  (75)  10MB  amd64  ubuntu@22.04
  0.16/candidate:    76  2026-08-28  (76)  12MB  amd64  ubuntu@24.04
```

**Comment lire la rubrique `channels` :** chaque ligne est `version/niveau : révision, date, taille, architecture, systèmes`.

| Niveau | Signification |
|---|---|
| `stable` | Version validée : **celle qu’on prend en production** |
| `candidate`, `beta`, `edge` | Versions de plus en plus expérimentales |
| `^` | Même révision que la ligne au-dessus |

**Le choix, ligne par ligne :**

| Branche | Systèmes | Retenue ? |
|---|---|---|
| `0.16/stable` | `ubuntu@22.04` seulement | Non : ne correspond pas à 24.04 |
| `0.16/candidate` | `ubuntu@24.04` | Non : pas `stable` |
| **`0.15/stable`** | **`ubuntu@24.04`** | **Oui** : la branche `stable` la plus récente pour 24.04 |
| `latest/stable` | `ubuntu@22.04, ubuntu@20.04` | Non |

D’où `--channel 0.15/stable`, révision 72. Ces branches évoluent : refais cette lecture le jour de l’installation, le résultat peut être différent.

### Étape 4 : savoir à quoi le relier

🎯 **Objectif : s’informer.** Un charm reçoit ce dont il a besoin par ses **relations**. La rubrique `relations` de `juju info` (étape 3) l’indique :

```text
  requires:
    controller: juju-dashboard
```

**Comment le lire :** le dashboard **a besoin** (`requires`) d’un point de connexion nommé `controller`, de type **`juju-dashboard`**. Qui le fournit ? On regarde la même rubrique pour l’application du contrôleur :

```bash
juju info juju-controller | sed -n '/relations/,/requires/p'
```

**Ce que tu vois :**

```text
relations:
  provides:
    dashboard: juju-dashboard
    metrics-endpoint: prometheus_scrape
    website: http
  requires:
```

**Comment le lire :** l’application `controller` **fournit** (`provides`) un point `dashboard` du même type `juju-dashboard`. Les deux types correspondent : c’est la relation à créer, d’où `juju integrate … dashboard controller` à l’étape 5. Les relations `ingress` et `nginx-route` du dashboard sont facultatives (publication web derrière un répartiteur) : on ne s’en sert pas ici.

### Étape 5 : installer le dashboard dans le modèle du contrôleur

```bash
juju deploy -m admin/controller juju-dashboard-k8s dashboard --channel 0.15/stable
juju integrate -m admin/controller dashboard controller
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `juju deploy` | Installer une application à partir d’un charm |
| `-m admin/controller` | Dans le modèle **du contrôleur** : le dashboard est un outil d’administration de Juju, pas du cloud |
| `juju-dashboard-k8s dashboard` | Le charm, et le nom donné à l’application : `dashboard` |
| `--channel 0.15/stable` | La branche choisie à l’étape 3 |
| `juju integrate … dashboard controller` | Relier le dashboard au contrôleur : c’est par ce lien qu’il obtient l’adresse du contrôleur |

ℹ C’est la seule commande de ce document qui **ajoute** un composant par Juju. C’est acceptable ici : le modèle `admin/controller` n’est pas géré par Sunbeam, qui ne viendra donc pas l’écraser.

**Ce que tu vois :**

```text
Deployed "dashboard" from charm-hub charm "juju-dashboard-k8s", revision 72 in channel 0.15/stable on ubuntu@24.04/stable
```

Suivre l’installation :

```bash
watch -n 10 "juju status -m admin/controller --format short"
```

**Ce que tu vois défiler** (relevé sur le lab) :

| Heure | État de `dashboard/0` | Signification |
|---|---|---|
| +30 s | `agent:allocating, workload:waiting` | Kubernetes télécharge et démarre le pod |
| +50 s | `agent:executing, workload:active` | Le charm configure le dashboard |
| +1 min 10 s | `agent:idle, workload:active` | Prêt |

Puis vérifier le lien :

```bash
juju status -m admin/controller --relations
```

**Ce que tu vois :**

```text
App         Version  Status  Scale  Charm               Channel      Rev  Address         Exposed  Message
controller           active      1  juju-controller     3.6/stable   361                  yes
dashboard            active      1  juju-dashboard-k8s  0.15/stable   72  10.152.183.213  no

Unit           Workload  Agent  Address     Ports      Message
controller/0*  active    idle   10.1.0.126  37017/TCP
dashboard/0*   active    idle   10.1.0.14

Integration provider  Requirer              Interface       Type     Message
controller:dashboard  dashboard:controller  juju-dashboard  regular
```

**Comment le lire :** l’application `dashboard` est `active`, avec une adresse **interne** à Kubernetes (`10.152.183.213`) et `Exposed no` : elle n’est pas joignable depuis le réseau. La ligne `controller:dashboard → dashboard:controller` confirme le lien.

### Étape 6 : créer un compte en lecture seule

**Pourquoi un compte dédié :** se connecter au dashboard avec un compte `superuser` donnerait à l’équipe le droit de tout modifier. On crée un compte qui ne peut que **lire** les deux modèles du cloud.

```bash
juju add-user lab-dashboard
juju grant lab-dashboard read openstack
juju grant lab-dashboard read admin/openstack-machines
juju change-user-password lab-dashboard
```

**La commande, mot par mot :**

| Ligne | Rôle |
|---|---|
| `add-user lab-dashboard` | Crée le compte. Il n’a encore accès à rien |
| `grant … read openstack` | Lecture du modèle des services OpenStack |
| `grant … read admin/openstack-machines` | Lecture du modèle de la machine |
| `change-user-password` | Définit son mot de passe ; la commande le demande deux fois, sans l’afficher |

**Ce que tu vois :**

```text
User "lab-dashboard" added
Please send this command to lab-dashboard:
    juju register <jeton>

"lab-dashboard" has not been granted access to any models. You can use "juju grant" to grant access.
new password:
type new password again:
Password for "lab-dashboard" has been changed.
```

**Comment le lire :** `add-user` propose un jeton `juju register`, qui sert à configurer la **ligne de commande** d’un autre poste ; pour le dashboard, on ne s’en sert pas, le mot de passe suffit. Le jeton est un secret : ne pas le diffuser.

Vérifier les droits :

```bash
juju users
juju show-model openstack | grep -A2 lab-dashboard
```

**Ce que tu vois :**

```text
Name           Display name  Access     Date created   Last connection
admin          admin         superuser  2026-10-09     2026-10-09
lab-dashboard                login      4 seconds ago  never connected
sunbeam*                     superuser  2026-10-09     just now

    lab-dashboard:
      access: read
      last-connection: never connected
```

**Comment le lire :** au niveau du contrôleur, `lab-dashboard` n’a que le droit `login` (se connecter) ; sur le modèle `openstack`, il a `read`. Il ne peut **rien modifier**.

### Étape 7 : lancer le relais

```bash
juju dashboard -m admin/controller --hide-credential
```

**La commande, mot par mot :** `dashboard` = ouvrir un relais vers l’interface web ; `-m admin/controller` = le modèle où elle est installée (sans `-m`, la commande répond `ERROR No selected model`) ; `--hide-credential` = ne pas afficher les identifiants du compte courant (`sunbeam`, administrateur) : on utilisera `lab-dashboard`.

**Ce que tu vois :**

```text
Dashboard for controller "sunbeam-controller" is enabled at:
  http://localhost:43959
```

**Comment le lire :** le relais écoute sur **Lab-Sunbeam**, port `43959`. **Note ce numéro** : il change à chaque lancement. Sur cette version, l’option `--port 31666` est **ignorée**. La commande reste au premier plan : **laisse cette session ouverte** tant que le dashboard sert.

### Étape 8 : ouvrir le tunnel depuis Windows

Dans un **nouveau** PowerShell, sur Windows, en remplaçant `43959` par le port de l’étape 7 :

```powershell
ssh -N -L 31666:localhost:43959 vagrant@192.168.56.10
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `-L 31666:localhost:43959` | Tout ce qui arrive sur le port **31666 de Windows** est envoyé, par SSH, au port **43959 de Lab-Sunbeam** |
| `-N` | Ne pas ouvrir de terminal : seulement le tunnel |

**Ce que tu vois :** la demande du mot de passe de `vagrant`, puis **rien** : c’est normal, le tunnel est ouvert tant que la fenêtre l’est.

### Étape 9 : se connecter

Dans le navigateur de Windows : **http://localhost:31666**

**Ce que tu vois :** une page Juju avec deux champs, **Username** et **Password**, et le bouton **Log in to the dashboard**. Saisir `lab-dashboard` et son mot de passe.

**Ce que tu vois ensuite :**

| Zone | Contenu sur le lab |
|---|---|
| Liste des modèles | **2 running** : `openstack` (25 applications, 25 unités) et `openstack-machines` (7 applications, 1 machine), avec leur propriétaire |
| Un clic sur un modèle | Onglets **Applications**, **Integrations**, **Action Logs**, **Secrets**, **Machines** ; pour `openstack-machines` : 7 applications locales `Running` (`cinder-volume`, `k8s`, `microceph`, `openstack-hypervisor`…), 9 applications distantes (les offres entre modèles) |
| Panneau de gauche | **ACCESS : read** : le compte ne peut que lire |

**Comment le lire :** le modèle `admin/controller` n’apparaît pas : le compte n’y a pas accès. C’est le principe du **moindre privilège** : chacun voit ce dont il a besoin, pas plus.

**Usage en formation et en exploitation :** pendant les parties 2 et 3, le dashboard affiche les mêmes `active` que la ligne de commande. Il montre l’**état déclaré par les charms** ; il ne remplace pas les tests des services.

### Si tu vois autre chose

| Ce que tu vois | Cause | Que faire |
|---|---|---|
| `ERROR No selected model` | `juju dashboard` sans `-m` | Ajouter `-m admin/controller` |
| Le navigateur ne répond pas | Tunnel fermé, mauvais port, ou session `juju dashboard` fermée | Relancer l’étape 7, noter le port, relancer l’étape 8 |
| `bind [127.0.0.1]:31666: Permission denied` puis `Could not request local forwarding.` dans PowerShell | Le port 31666 de Windows est **déjà pris**, le plus souvent par un ancien tunnel resté ouvert (observé sur le lab). Malgré le message, `ssh` reste connecté, sans tunnel | `Ctrl+C`, puis trouver le processus qui tient le port : `Get-NetTCPConnection -LocalPort 31666 -State Listen` (colonne `OwningProcess`), fermer l’ancien tunnel, et relancer. Ou utiliser un autre port local : `-L 31667:localhost:PORT`, puis http://localhost:31667 |
| Échec de connexion | Mauvais mot de passe | `juju change-user-password lab-dashboard` |
| `dashboard/0` reste `waiting` | Le lab n’accède pas à Internet (Charmhub) | Vérifier la sortie Internet de Lab-Sunbeam |

### Étape 10 : retirer le dashboard

Le dashboard n’est pas nécessaire au fonctionnement du cloud. Pour revenir à l’état d’origine :

1. fermer le tunnel (fenêtre PowerShell : `Ctrl+C`) et le relais (session de l’étape 7 : `Ctrl+C`) ;
2. puis :

```bash
juju remove-application -m admin/controller dashboard --no-prompt
juju remove-user lab-dashboard --yes
juju status -m admin/controller
juju users
```

**Ce que tu vois :**

```text
will remove application dashboard
- will remove unit dashboard/0
User "lab-dashboard" removed
```

puis, une minute plus tard, seule l’application `controller` dans le modèle, et seuls les comptes `admin` et `sunbeam`.

**Comment le lire :** pendant la suppression, `dashboard` apparaît quelques secondes en `unknown`, avec `Scale 0` : l’unité est déjà partie, l’application va suivre.

---

## Remettre le lab dans son état d’origine

Chaque partie se termine par son nettoyage. Pour tout vérifier en une fois :

```bash
juju config -m openstack glance image-size-cap
snap services cinder-volume
source ~/admin-openrc
openstack volume service list -c Binary -c State -f value
openstack volume list --all-projects
openstack image list -c Name
juju status -m openstack --format summary | grep -A1 "# Units"
juju status -m admin/openstack-machines --format short | grep -v "workload:active"
juju status -m admin/controller --format short
juju users
```

**Ce que tu vois sur le lab, après les tests :**

| Contrôle | Attendu |
|---|---|
| `image-size-cap` | Ligne vide (valeur par défaut) |
| `cinder-volume` | `enabled  active` |
| Services de volumes | `cinder-scheduler up`, `cinder-volume up` |
| Volumes | Aucun |
| Images | `ubuntu`, `ubuntu-24-04` uniquement |
| Modèle `openstack` | `active: 25` |
| Modèle `admin/openstack-machines` | Aucune ligne |
| Modèle `admin/controller` | Seulement `controller/0` |
| Comptes Juju | `admin` et `sunbeam` |

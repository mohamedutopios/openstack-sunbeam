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
sudo k8s kubectl -n openstack logs glance-0 -c glance-api --since=1h | grep -i "too large" | sed -E 's/.*\] //' | head -2
```

**La commande, mot par mot :**

| Morceau | Rôle |
|---|---|
| `logs glance-0 -c glance-api` | Le journal du conteneur `glance-api` du pod `glance-0` : celui du **service** |
| `--since=1h` | Seulement la dernière heure. **Adapter** à l’heure de l’envoi refusé : avec `--since=10m`, un refus vieux de 20 minutes n’apparaît pas, et la commande ne renvoie **rien** |
| `grep -i "too large"` | Garder les lignes qui contiennent « too large », quelle que soit la casse |
| `sed -E 's/.*\] //'` | Retirer le début technique de chaque ligne (date, numéro de processus…) |
| `head -2` | Les deux premières suffisent |

Si rien ne s’affiche : relancer l’envoi de l’étape 1, puis cette commande aussitôt.

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

**Puis attendre que Cinder constate l’arrêt.** Cinder ne déclare un service `down` qu’après **environ une minute sans signal de vie**. Avant, il le croit encore vivant. On attend donc qu’il l’affiche `down` :

```bash
source ~/admin-openrc
until openstack volume service list -c Binary -c State -f value | grep -q "cinder-volume down"; do sleep 10; done
openstack volume service list -c Binary -c State -f value
```

**La commande, mot par mot :** `until … ; do sleep 10; done` = répéter toutes les 10 secondes **jusqu’à ce que** la ligne `cinder-volume down` apparaisse. La commande rend la main au bout d’une à deux minutes.

**Ce que tu vois :**

```text
cinder-scheduler up
cinder-volume down
```

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

Un premier indice, avec le compte administrateur, qui seul voit l’hôte d’un volume :

```bash
source ~/admin-openrc
openstack volume show lab-vol-panne -c status -c os-vol-host-attr:host -f value
```

**Ce que tu vois :**

```text
None
error
```

**Comment le lire :** `-f value` affiche les colonnes dans l’ordre alphabétique : d’abord `os-vol-host-attr:host`, l’hôte de stockage chargé du volume, puis `status`. Hôte **`None`** : le planificateur ne l’a confié à **aucun** stockage. Le problème est en amont du stockage, au moment du choix.

**Si le volume reste en `creating`** : il a été demandé **avant** que Cinder déclare le service `down` (attente précédente sautée ou trop courte). Observé sur le lab en créant le volume aussitôt après `snap stop`, 55 secondes après le dernier signal du service. La même commande donne alors :

```text
sunbeam@cinder-volume-ceph#cinder-volume-ceph
creating
```

**Comment le lire :** le planificateur croyait encore le service vivant : il lui a confié le volume (hôte rempli, `sunbeam`). La demande attend dans la file de messages un service qui ne répond plus. C’est une **deuxième forme** de la même panne, fréquente en production : un volume qui reste en `creating` alors que son hôte est `down`. Elle se diagnostique avec les mêmes étapes 3 à 6.

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

### Pourquoi Juju et Sunbeam ne voient pas la panne

Ce n’est pas un bug, c’est la façon dont l’état est **calculé**. Pour le prouver, on lit le code qui le calcule : il est sur le lab.

**1. Ce que Juju affiche, c’est ce que le charm lui déclare.** Juju ne teste pas lui-même les services : il affiche l’état (`workload`) que le **charm** de chaque unité lui transmet à la fin de chacune de ses étapes (*hooks*). La question est donc : quand et comment le charm `cinder-volume` décide-t-il qu’il est `active` ?

Le code du charm est installé avec l’unité, dans `/var/lib/juju/agents/unit-NOM-NUMÉRO/charm/` :

```bash
sudo grep -n -A6 "def check_serving_backends" /var/lib/juju/agents/unit-cinder-volume-0/charm/src/charm.py
```

```text
285:    def check_serving_backends(self):
286-        """Check if backends are ready to serve."""
287-        if not self.cinder_volume.backends():
288-            msg = "Waiting for backends"
289-            self._backend_status.set(ops.WaitingStatus(msg))
290-            raise sunbeam_guard.WaitingExceptionError(msg)
291-        self._backend_status.set(ops.ActiveStatus())
```

**Comment le lire :** le charm se déclare `active` (`ActiveStatus`) dès qu’un **stockage est déclaré** par sa relation avec `cinder-volume-ceph` (`backends()`). Il ne vérifie **pas** que le service tourne.

**2. Ce calcul n’a lieu que lors d’une reconfiguration.** La fonction ci-dessus est appelée par la configuration du charm, déclenchée par des événements : changement de configuration, de relation, de secret, démarrage. Le contrôle périodique de Juju, l’étape `update-status`, lancée toutes les 5 minutes :

```bash
juju model-config -m admin/openstack-machines update-status-hook-interval
```

```text
5m
```

n’est **pas traité** par ce charm : ni son code, ni la bibliothèque commune des charms Sunbeam ne réagissent à cet événement.

```bash
sudo grep -c "update_status\|update-status" /var/lib/juju/agents/unit-cinder-volume-0/charm/src/charm.py /var/lib/juju/agents/unit-cinder-volume-0/charm/lib/ops_sunbeam/charm.py
```

```text
/var/lib/juju/agents/unit-cinder-volume-0/charm/src/charm.py:0
/var/lib/juju/agents/unit-cinder-volume-0/charm/lib/ops_sunbeam/charm.py:0
```

**Comment le lire :** zéro occurrence dans les deux fichiers. À chaque `update-status`, le charm ne refait aucune vérification : il **republie le dernier état mémorisé**, `active`, calculé lors de sa dernière configuration. C’est la ligne `10:42:08 workload active` vue plus haut : un contrôle périodique a bien eu lieu… et a recopié l’ancien résultat.

**3. Sunbeam ne regarde même pas `cinder-volume`.** La colonne `Storage` de `sunbeam cluster list` est calculée par le code de la commande `sunbeam`, installé dans le snap `openstack` :

```bash
grep -n -A8 "def applications_to_columns" /snap/openstack/current/lib/python3.12/site-packages/sunbeam/steps/cluster_status.py
```

```text
126:    def applications_to_columns(self) -> dict:
127-        """Mapping of applications to columns."""
128-        return {
129-            clusterd.APPLICATION: "clusterd",
130-            k8s.APPLICATION: "control",
131-            hypervisor.APPLICATION: "compute",
132-            microceph.APPLICATION: "storage",
133-            microovn.APPLICATION: "network",
134-        }
```

**Comment le lire :** chaque colonne recopie l’état Juju **d’une seule application** : `Storage` = l’état de **`microceph`**, `Control` = celui de `k8s`, `Compute` = celui de l’hyperviseur. `cinder-volume` n’est dans **aucune** colonne. Ceph allant bien, `Storage` reste `active`, quel que soit l’état du service des volumes.

**En résumé :**

| Outil | Ce qu’il affiche réellement | Pourquoi il ne voit pas l’arrêt |
|---|---|---|
| `juju status` | L’état déclaré par le charm lors de sa **dernière configuration** | Le charm ne vérifie que la présence d’un stockage, et ne refait rien lors du contrôle périodique |
| `sunbeam cluster list` | L’état Juju de `microceph` (colonne `Storage`) | `cinder-volume` ne fait partie d’aucune colonne |
| `openstack volume service list` | Le **signal de vie** que le service envoie lui-même toutes les quelques secondes | Il voit l’arrêt : c’est la seule des trois vues qui interroge le service |

**Ce qu’il faut retenir :** avant de faire confiance à un `active`, savoir **qui** le calcule et **sur quoi**. Pour un service, la preuve qu’il fonctionne vient du service lui-même (son signal de vie, un appel réel), pas de l’outil qui l’a installé. C’est pourquoi on descend maintenant **sur la machine**.

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

**Que veut dire le `0` de `juju ssh … 0` ?** C’est le **numéro de la machine** dans le modèle. Juju numérote les machines d’un modèle à partir de 0 ; la liste est la section `Machine` de `juju status` :

```bash
juju status -m admin/openstack-machines | sed -n '/^Machine/,/^$/p'
```

```text
Machine  State    Address    Inst id               Base          AZ  Message
0        started  10.0.2.15  manual:192.168.56.10  ubuntu@24.04      Manually provisioned machine
```

| Colonne | Signification |
|---|---|
| `Machine 0` | Le numéro, à donner à `juju ssh` |
| `Inst id manual:192.168.56.10` | La machine a été ajoutée à la main, par son adresse : c’est **la VM Lab-Sunbeam** |
| `Base ubuntu@24.04` | Son système |

`juju ssh` accepte soit un **numéro de machine** (`0`), soit une **unité** (`cinder-volume/0`). Sur ce lab à un seul nœud, on est **déjà** sur la machine `0` : `juju ssh … 0 "commande"` donne le même résultat que `commande` tapée directement. On passe quand même par Juju pour prendre l’habitude : sur un cloud de plusieurs nœuds, c’est Juju qui sait sur quelle machine tourne l’unité, et `juju ssh` y va sans qu’on ait à connaître son adresse.

**Qu’est-ce que `snap`, et quel rapport avec `systemctl` ?** `snap` est le système de **paquets** d’Ubuntu, comme `apt` : il installe, met à jour et retire des logiciels, chacun dans son paquet isolé (un *snap*). Un snap peut contenir des **services**, et pour les faire tourner, il s’appuie sur **systemd**, exactement comme les autres services Linux. Chaque service de snap est une unité systemd nommée `snap.NOM_DU_SNAP.NOM_DU_SERVICE` :

```bash
systemctl status snap.cinder-volume.cinder-volume --no-pager | head -3
```

```text
○ snap.cinder-volume.cinder-volume.service - Service for snap application cinder-volume.cinder-volume
     Loaded: loaded (/etc/systemd/system/snap.cinder-volume.cinder-volume.service; enabled; preset: enabled)
     Active: inactive (dead) since Sat 2026-10-17 15:04:12 UTC; 12min ago
```

| Commande `snap` | Équivalent `systemctl` | Ce que `snap` ajoute |
|---|---|---|
| `snap services cinder-volume` | `systemctl status snap.cinder-volume.*` | Tous les services du snap d’un coup, sans connaître leurs noms systemd |
| `sudo snap stop cinder-volume` | `sudo systemctl stop snap.cinder-volume.cinder-volume` | L’opération est **enregistrée** dans l’historique de `snapd` |
| `sudo snap start cinder-volume` | `sudo systemctl start …` | Idem |
| `snap changes` | *(aucun)* | **L’historique des opérations** : installations, mises à jour, arrêts, démarrages, avec leur heure |

C’est cette dernière ligne qui nous intéresse : `systemctl` sait qu’un service **est** arrêté, mais pas **qui** l’a arrêté ni par quel moyen ; `snap changes` garde la trace d’un `snap stop`. (Le journal `journalctl`, plus bas, montre l’arrêt vu par systemd.)

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
juju ssh -m admin/openstack-machines 0 "snap changes | tail -3"
```

```text
265  Done    today at 06:37 UTC      today at 06:37 UTC      Running service command
266  Done    today at 10:37 UTC      today at 10:37 UTC      Running service command
```

**Comment le lire :** une ligne par opération sur les snaps, la plus récente en bas ; la sortie de `snap changes` se termine par une ligne vide, d’où `tail -3` pour voir les deux dernières. La première colonne est le **numéro** de l’opération ; le résumé, « Running service command », ne dit pas laquelle. On demande le détail de la plus récente, celle qui correspond à l’heure de la panne :

```bash
juju ssh -m admin/openstack-machines 0 "snap change 266"
```

🔁 **Remplace `266` par le numéro de ta dernière ligne** : il change à chaque opération (en rejouant la démo plus tard dans la journée, c’était `279`).

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

**Et le volume de l’étape 1 ?** Tout dépend de son état pendant la panne :

```bash
openstack volume show lab-vol-panne -c status -f value
```

| État pendant la panne | État après la correction | Explication |
|---|---|---|
| `error` (volume demandé alors que le service était déjà `down`) | Reste `error` | Le planificateur a refusé la demande : elle n’existe plus. Il faut supprimer ce volume et en recréer un |
| `creating` (volume demandé moins d’une minute après l’arrêt) | **`available`**, sans rien refaire | Observé sur le lab : la demande attendait dans la file de messages ; le service redémarré l’a traitée. Un volume bloqué en `creating` n’est pas perdu, il attend son service |

⚠ Pendant la panne, un volume en `creating` **ne peut pas être supprimé** : `openstack volume delete` répond `Invalid volume: Volume status must be available or error … (HTTP 400)`. C’est une protection : Cinder ne supprime pas un volume qu’un service est peut-être en train de créer. On répare d’abord le service, puis on supprime.

### Nettoyer

```bash
openstack volume delete lab-vol-ok lab-vol-panne
openstack volume list
```

Si tu as créé d’autres volumes de test pendant la panne (`lab-vol-panne2`…), ajoute-les à la commande.

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

Puis vérifier le lien créé par `juju integrate` :

```bash
juju status -m admin/controller --relations
```

**La commande, mot par mot :** `juju status -m admin/controller` affiche les applications et les unités du modèle ; **`--relations`** ajoute une section **`Integration`** qui liste les **relations** entre applications (les liens créés par `juju integrate`). Sans cette option, `juju status` ne les affiche pas : sur le lab, la même commande sans `--relations` s’arrête après la section `Unit`. C’est donc l’option à ajouter pour vérifier qu’une relation existe.

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

**Comment le lire**, section par section :

| Section | Colonne | Valeur | Signification |
|---|---|---|---|
| `App` | `Status` | `active` | L’application `dashboard` fonctionne |
| | `Address` | `10.152.183.213` | Son adresse **interne** à Kubernetes : joignable seulement depuis le cluster |
| | `Exposed` | `no` | Juju ne l’a pas ouverte sur le réseau (`juju expose`) ; le contrôleur, lui, l’est (`yes`) car les clients Juju doivent le joindre |
| `Unit` | `dashboard/0*` | `active idle` | Son unité unique, prête et au repos |
| `Integration` | `Integration provider` | `controller:dashboard` | **Qui fournit** : l’application `controller`, par son point de connexion (*endpoint*) `dashboard` |
| | `Requirer` | `dashboard:controller` | **Qui consomme** : l’application `dashboard`, par son point `controller` |
| | `Interface` | `juju-dashboard` | Le « protocole » de la relation : les deux points doivent avoir le même. C’est celui qu’on a lu dans `juju info` à l’étape 4 |
| | `Type` | `regular` | Une relation ordinaire entre deux applications (`peer` désignerait une relation d’une application avec elle-même, entre ses unités ; `subordinate`, celle d’un charm greffé sur un autre) |
| | `Message` | *(vide)* | Pas de problème signalé sur la relation |

La ligne `controller:dashboard → dashboard:controller` est exactement ce que l’étape 4 avait prévu (`provides: dashboard` d’un côté, `requires: controller` de l’autre, même interface) : **la relation est en place**. Si cette ligne manquait, c’est que `juju integrate` n’a pas été lancé ou a échoué : le relancer.

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

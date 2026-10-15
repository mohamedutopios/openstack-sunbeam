# Juju : comprendre son rôle et utiliser les commandes utiles

Fiche pratique pour le lab OpenStack Sunbeam — syntaxe Juju 3.6, version observée dans notre déploiement : 3.6.29.

## 1. Qu’est-ce que Juju ?

**Juju est un outil d’orchestration du déploiement et de l’exploitation des applications.** Il utilise des opérateurs appelés **charms**, qui contiennent les procédures nécessaires pour installer, configurer, intégrer et administrer une application.

Une application comme Keystone doit connaître sa base de données, recevoir des informations d’authentification et publier ses endpoints. Avec Juju, les charms réagissent aux événements et échangent des informations par des intégrations pour automatiser ces opérations.

Juju s’utilise avec des applications sur machines et sur Kubernetes. Ce n’est ni un hyperviseur ni un remplacement de Kubernetes. Il ne remplace pas non plus Prometheus, Grafana et Loki pour les historiques de consommation et les logs centralisés.

### Dans notre lab

| Outil | Responsabilité |
|---|---|
| VirtualBox | Exécuter les VM Ubuntu Services et Sunbeam sur le PC |
| Sunbeam | Piloter le déploiement et la configuration globale d’OpenStack |
| Juju et ses charms | Automatiser la configuration et l’exploitation des composants |
| Kubernetes | Exécuter et maintenir les pods du plan de contrôle |
| OpenStack Nova | Gérer les instances des utilisateurs via les hyperviseurs |
| Prometheus, Grafana, Loki | Historiser les métriques, les afficher et centraliser les logs |

**Créer une instance Nova se fait avec Horizon ou `openstack server create`, pas avec `juju add-unit`.** Ajouter une unité Juju concerne un composant applicatif, pas une VM utilisateur Nova.

## 2. Les concepts à connaître

| Concept | Explication | Exemple dans le lab |
|---|---|---|
| Client Juju | Commande utilisée par l’administrateur | `juju status` |
| Controller | Service central qui gère les modèles et coordonne les agents | `sunbeam-controller` |
| Cloud | Infrastructure cible connue de Juju : Kubernetes, machines, etc. | Cloud Kubernetes du lab |
| Model | Périmètre contenant applications, unités, intégrations et configuration | `openstack` |
| Charm | Opérateur qui automatise le cycle de vie d’un logiciel | `neutron-k8s` |
| Application | Déploiement d’un charm sous un nom donné | `neutron` |
| Unit | Exemplaire d’une application | `neutron/0` |
| Agent | Processus Juju chargé d’exécuter la logique du charm | Agent de `neutron/0` |
| Workload | Logiciel réellement géré par le charm | Serveur Neutron |
| Integration / relation | Lien permettant aux charms d’échanger des données de configuration | Keystone et son routeur MySQL |
| Action | Opération administrative proposée par un charm | Sauvegarde, diagnostic ou récupération d’information, selon le charm |
| Offer | Endpoint d’application proposé à un autre modèle | Offre de credentials Keystone |

Une unité Kubernetes correspond généralement à un pod, mais **une unité n’est pas un conteneur unique**. Dans notre lab, `nova/0` correspond au pod `nova-0`, qui contient notamment `charm`, `nova-api`, `nova-conductor`, `nova-scheduler` et `nova-spiceproxy`.

Une unité marquée d’un astérisque, par exemple `mysql/0*`, est le **leader Juju** de l’application. Ce rôle ne doit pas être confondu systématiquement avec celui de serveur primaire du logiciel : le charm et le logiciel déterminent leurs rôles respectifs.

## 3. Les deux modèles importants de Sunbeam

| Modèle | Ce qu’il faut y vérifier |
|---|---|
| `openstack` | Applications du plan de contrôle sur Kubernetes : Keystone, Nova API, Neutron, Glance, Cinder, MySQL, RabbitMQ, Traefik… |
| `openstack-machines` | Applications gérées sur les machines : hyperviseur, MicroCeph, Cinder Volume selon le déploiement |

La section **SAAS** de `juju status` représente des applications distantes consommées via des offres inter-modèles. Dans notre cas, `microceph` et `cinder-volume` peuvent y figurer. Il ne s’agit pas nécessairement d’un service SaaS hébergé sur Internet.

## 4. Où exécuter les commandes ?

Dans la **VM Sunbeam**, avec le compte **vagrant** qui possède déjà les accès Juju.

```bash
juju version
juju controllers
juju models
```

Ne pas utiliser `sudo juju` : cela change le contexte utilisateur et peut faire disparaître les accès au contrôleur. En revanche, les commandes `sudo k8s kubectl` du lab nécessitent sudo.

Pour éviter d’agir dans le mauvais modèle, les exemples utilisent `-m` explicitement.

```bash
juju status -m openstack
juju status -m openstack-machines
```

Pour sélectionner un modèle par défaut :

```bash
juju switch sunbeam-controller:openstack
juju status
```

`switch` change le contexte du client, pas le placement des services. Si les noms diffèrent, reprendre ceux affichés par `juju controllers` et `juju models`.

## 5. Consulter l’état du cloud

### Vue générale

```bash
juju status -m openstack
juju status -m openstack-machines
```

Lire les colonnes **Status/Workload**, **Agent** et **Message**. Un message explique souvent immédiatement la relation ou la ressource manquante.

### Actualisation automatique

```bash
watch -n 5 'juju status -m openstack'
```

`Ctrl+C` arrête seulement l’affichage.

### Filtrer une application

```bash
juju status -m openstack neutron
juju status -m openstack nova
juju status -m openstack mysql
```

### Afficher les intégrations

```bash
juju status -m openstack --relations
```

Utile pour comprendre qui est connecté à MySQL, RabbitMQ ou Keystone. Une relation présente ne garantit pas à elle seule que les échanges réseau fonctionnent.

### Lire les états

| État | Sens pratique |
|---|---|
| active | Le charm estime que le workload est opérationnel selon ses contrôles |
| waiting | Attend une dépendance ou une étape de convergence |
| blocked | Intervention ou condition de configuration manquante |
| maintenance | Une opération de configuration ou maintenance est en cours |
| error | Une opération de l’agent/charm a échoué ; consulter les détails et logs |
| Agent idle | L’agent n’exécute pas de hook à cet instant ; c’est habituellement normal |
| Agent executing | L’agent traite une opération |

**`active` ne prouve pas que toutes les requêtes métier réussissent.** Dans notre lab, les pods pouvaient être prêts alors que Neutron échouait à résoudre `internal.lab.test` lors de la validation des tokens Keystone.

## 6. Inspecter une application et une unité

```bash
juju show-application -m openstack neutron
juju show-unit -m openstack neutron/0
juju show-unit -m openstack nova/0
juju show-model -m openstack
```

Ces commandes donnent des détails sur le déploiement, les unités et leurs relations. Les sorties peuvent contenir des informations sensibles : les relire avant de les partager.

Pour consulter les machines gérées par le modèle machine :

```bash
juju machines -m openstack-machines
```

Cette liste ne représente pas les instances Nova créées par les apprenants.

## 7. Lire les logs Juju

### Flux des erreurs d’orchestration

```bash
juju debug-log -m openstack --level ERROR
```

### Inclure avertissements et erreurs d’une unité

```bash
juju debug-log -m openstack --include unit-neutron-0 --level WARNING
```

Le nom d’unité affiché est `neutron/0`, mais le filtre d’entité des logs utilise **`unit-neutron-0`**.

### Rejouer l’historique disponible, puis suivre la suite

```bash
juju debug-log -m openstack --replay --include unit-neutron-0
```

### Lire l’historique sans rester en écoute

```bash
juju debug-log -m openstack --replay --no-tail \
  --include unit-neutron-0 --level WARNING
```

L’historique dépend de la rétention du contrôleur ; il ne couvre pas nécessairement toute la vie du cloud.

### Voir l’historique des changements de statut

```bash
juju show-status-log -m openstack neutron/0
```

Permet de voir quand l’unité est passée de `waiting` à `active`, ou si elle alterne entre plusieurs états.

### Différence essentielle : logs du charm et logs du service

`juju debug-log` suit les agents et les charms. Pour les erreurs HTTP de Neutron ou le comportement de Nova, consulter aussi les logs **du workload** :

```bash
sudo k8s kubectl logs -n openstack neutron-0 -c neutron-server \
  --since=20m --tail=150

sudo k8s kubectl logs -n openstack nova-0 -c nova-api \
  --since=20m --tail=150

sudo k8s kubectl logs -n openstack nova-0 -c nova-scheduler \
  --since=20m --tail=150
```

Les noms de conteneurs sont ceux observés dans notre lab. Pour les retrouver :

```bash
sudo k8s kubectl get pods -n openstack \
  -o custom-columns='POD:.metadata.name,CONTENEURS:.spec.containers[*].name'
```

## 8. Entrer dans une unité ou un conteneur

### Conteneur du service Neutron

```bash
juju ssh -m openstack --container neutron-server neutron/0
```

### Conteneur Nova API

```bash
juju ssh -m openstack --container nova-api nova/0
```

### Conteneur du charm

```bash
juju ssh -m openstack --container charm neutron/0
```

Utiliser `exit` pour quitter. Pour les modèles Kubernetes, Juju fournit un accès au conteneur ; cela ne suppose pas qu’un serveur SSH classique fonctionne dans le pod.

### Exécuter une commande de diagnostic

```bash
juju ssh -m openstack --container neutron-server neutron/0 \
  getent ahostsv4 internal.lab.test
```

Attendu dans notre lab : `192.168.56.202`. Un conteneur minimal peut ne pas contenir curl, ping ou d’autres outils ; leur absence ne constitue pas une panne OpenStack.

### Unité sur machine

Lister d’abord les unités :

```bash
juju status -m openstack-machines
```

Si `microceph/0` est bien présente :

```bash
juju ssh -m openstack-machines microceph/0
```

Éviter de modifier à la main les fichiers générés par un charm : une prochaine réconciliation peut écraser ces modifications.

## 9. Consulter et modifier la configuration

### Lecture seule

```bash
juju config -m openstack neutron
juju config -m openstack traefik-public
```

La sortie liste les options réellement disponibles, leur valeur et leur description. Une option Juju n’est pas forcément identique à une clé du fichier neutron.conf ou nova.conf.

### Forme d’une modification — exemple générique

```bash
# Remplacer APPLICATION, OPTION et VALEUR après lecture de la documentation.
juju config -m MODELE APPLICATION OPTION=VALEUR
```

Cette commande change l’état du déploiement et peut déclencher une reconfiguration ou un redémarrage. Pour les composants gérés par Sunbeam, privilégier les commandes et le manifeste Sunbeam lorsque le réglage y est pris en charge, afin de garder une configuration cohérente.

Il n’existe pas de bouton universel « tout réparer » dans Juju. Une modification doit répondre à un problème identifié.

## 10. Découvrir et utiliser les actions d’un charm

### Lister les actions proposées

```bash
juju actions -m openstack mysql
juju actions -m openstack keystone
```

Les actions varient selon le charm, son canal et sa révision. Ne pas supposer qu’une action existe parce qu’elle existe sur un autre déploiement.

### Lire les paramètres d’une action

```bash
# Noms à remplacer par ceux obtenus dans la liste précédente.
juju show-action -m MODELE APPLICATION ACTION
```

### Exécuter une action en Juju 3.6

```bash
# Exemple de syntaxe, pas une commande à copier sans adaptation.
juju run -m MODELE APPLICATION/leader ACTION parametre=valeur
```

`APPLICATION/leader` cible le leader actuel. Certaines actions doivent au contraire cibler une unité précise, par exemple `APPLICATION/0` : suivre leur description.

**Dans Juju 3.x, `juju run` exécute une action du charm.** Ne pas reprendre sans adaptation les anciens tutoriels Juju 2.x qui utilisent `juju run` pour une commande shell arbitraire.

### Exemple utile si l’observabilité a été installée

Après avoir confirmé le nom du modèle et de l’application :

```bash
juju models
juju actions -m observability grafana
juju show-action -m observability grafana get-admin-password
juju run -m observability grafana/leader get-admin-password
```

Cette action expose un secret d’administration. Ne pas en coller la sortie dans un compte rendu. Le modèle `observability` n’existe pas dans l’OVA de base tant que la fonctionnalité n’a pas été déployée.

## 11. Comprendre les intégrations inter-modèles

```bash
juju status -m openstack --relations
juju offers -m openstack
```

Une offre permet à un autre modèle de consommer un endpoint, par exemple les informations d’authentification Keystone. Les charms peuvent ensuite échanger les données nécessaires à la configuration.

La création d’une relation utilise la forme suivante :

```bash
# Exemple générique : cette commande modifie le modèle.
juju integrate -m MODELE APPLICATION_A:ENDPOINT_A APPLICATION_B:ENDPOINT_B
```

**Dans notre Sunbeam, ne pas recréer manuellement les relations existantes.** Lire d’abord `status --relations` et le message du charm. Un `Missing relation` peut aussi être une étape temporaire pendant le déploiement.

## 12. Réessayer un hook en échec

Lorsqu’une unité est en erreur à cause d’un hook, corriger d’abord la cause : réseau, DNS, configuration, ressource indisponible, etc.

```bash
juju status -m openstack neutron
juju debug-log -m openstack --replay --no-tail --include unit-neutron-0
```

**Seulement après correction**, si l’agent attend toujours la résolution de l’erreur :

```bash
juju resolved -m openstack neutron/0
```

Cette opération permet de retenter le hook échoué selon le comportement Juju 3.6. Elle ne corrige pas la cause initiale et n’est pas destinée à effacer un statut `blocked` normal. Éviter les options qui ignorent le hook sans l’avoir réparé.

## 13. Méthode de dépannage en six étapes

1. **Identifier le modèle et l’application** : `juju status` dans les deux modèles.
2. **Lire le message exact** : `show-unit` et `show-status-log`.
3. **Vérifier les dépendances** : `status --relations` et état de MySQL, RabbitMQ, Keystone, Ceph.
4. **Lire les bons logs** : `debug-log` pour le charm ; logs du conteneur pour le logiciel.
5. **Tester depuis le bon endroit** : résolution DNS et connectivité depuis le conteneur qui rencontre l’erreur.
6. **Corriger puis vérifier le résultat métier** : appel API, création Nova, accès SSH ou opération Cinder.

### Exemple de notre panne DNS

Symptôme : Neutron renvoie HTTP 503 avec « Keystone temporarily unavailable », alors que Keystone est accessible depuis Ubuntu.

```bash
juju status -m openstack neutron keystone
sudo k8s kubectl logs -n openstack neutron-0 -c neutron-server --tail=150
juju ssh -m openstack --container neutron-server neutron/0 \
  getent ahostsv4 internal.lab.test
```

La cause était la résolution DNS **dans le pod**, pas la disponibilité de l’API Keystone depuis la VM. Juju donne le contexte applicatif ; les logs et le test dans le conteneur permettent de trouver la cause.

## 14. Aide locale : vérifier la syntaxe de la version installée

```bash
juju help status
juju help debug-log
juju help ssh
juju help run
juju help resolved
```

L’aide locale est particulièrement utile pour les options qui ont changé entre Juju 2.x et 3.x.

## 15. Commandes à ne pas confondre avec des contrôles

| Commande | Effet |
|---|---|
| `juju config ... cle=valeur` | Modifie une configuration |
| `juju run` | Exécute une action, dont l’effet dépend du charm |
| `juju integrate` | Crée une intégration |
| `juju refresh` | Peut changer la révision ou le canal du charm |
| `juju add-unit` / `juju scale-application` | Change le nombre d’unités selon le type de modèle et de charm |
| `juju remove-unit` / `juju remove-application` | Supprime des unités ou applications |
| `juju destroy-model` / `juju destroy-controller` | Détruit un périmètre de gestion et potentiellement ses ressources |

Ces commandes ne doivent pas être utilisées au hasard pour « relancer OpenStack ». Pour les mises à niveau, la capacité et les changements de topologie Sunbeam, suivre le workflow prévu par Sunbeam.

## 16. Mémo quotidien pour notre lab

```bash
# 1. État des applications
juju status -m openstack
juju status -m openstack-machines

# 2. Relations
juju status -m openstack --relations

# 3. Détails d’une unité
juju show-unit -m openstack neutron/0
juju show-status-log -m openstack neutron/0

# 4. Logs du charm
juju debug-log -m openstack --replay --no-tail \
  --include unit-neutron-0 --level WARNING

# 5. Logs du service
sudo k8s kubectl logs -n openstack neutron-0 -c neutron-server --tail=100

# 6. Test fonctionnel OpenStack
source ~/demo-openrc
openstack network list
openstack server list
```

**Juju permet de comprendre et gérer les applications ; Kubernetes renseigne sur les pods ; les API OpenStack et les tests réels valident le service rendu aux utilisateurs.**

## Références officielles

- [Documentation Juju 3.6](https://documentation.ubuntu.com/juju/3.6/)
- [Gestion des applications](https://canonical.com/juju/docs/juju-cli/3.6/howto/manage-applications/)
- [Gestion des logs](https://canonical.com/juju/docs/juju-cli/3.6/howto/manage-logs/)
- [Commande debug-log](https://documentation.ubuntu.com/juju/3.6/reference/juju-cli/list-of-juju-cli-commands/debug-log/)
- [Commande show-action](https://documentation.ubuntu.com/juju/3.6/reference/juju-cli/list-of-juju-cli-commands/show-action/)
- [Inspection d’un cluster Canonical OpenStack](https://canonical.com/openstack/docs/latest/how-to/troubleshooting/inspecting-the-cluster/)

Les noms d’applications et de conteneurs sont adaptés aux sorties observées dans notre lab. Les commandes de cette fiche n’ont pas été exécutées sur les VM de l’apprenant.

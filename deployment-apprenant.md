Voici le pas à pas pour distribuer **tes deux OVA actuelles** aux apprenants : `Services-base.ova` et `Sunbeam-base.ova`.

**Cette version permet d’utiliser OpenStack avec le compte `demo`. Le SSO Keycloak reste à configurer et l’accès aux instances Nova reste à tester.**

**1. Formateur : préparer les fichiers et les accès avant l’export**

Dans la console de **chaque VM**, définis un mot de passe connu pour le compte Linux `vagrant` :

```bash
sudo passwd vagrant
```

Les apprenants utiliseront ce compte dans la console VirtualBox. La clé SSH créée par Vagrant sur ton PC ne sera pas automatiquement disponible chez eux.

Prépare ce dossier à distribuer :

```text
Lab-OpenStack/
  Services-base.ova
  Sunbeam-base.ova
  Configurer-Lab-Sunbeam.ps1
  ca.crt
```

Ajoute une fiche d’accès avec :

| Accès | Informations à transmettre |
|---|---|
| Console Linux des deux VM | `vagrant` et mot de passe défini |
| Horizon | Domaine `users`, utilisateur `demo`, mot de passe |
| Administration Keycloak | `kcadmin` et mot de passe du lab |

Le fichier `ca.crt` doit correspondre aux certificats présents dans **ces OVA**.

Arrête proprement les deux VM, puis exporte-les **en conservant toutes les adresses MAC**.

**2. Apprenant : vérifier les ressources du PC**

Pour reprendre les allocations de notre lab :

| Ressource | Valeur |
|---|---|
| Architecture | PC Intel/AMD x86-64 |
| RAM du PC | 32 Go minimum pour ce montage |
| VM Sunbeam | Environ 20 Go de RAM, 8 processeurs virtuels |
| VM Services | 3 Go de RAM, 2 processeurs virtuels |
| Stockage | SSD ; prévoir environ 300 Go libres pour l’import et la croissance du lab |
| Virtualisation matérielle | Intel VT-x ou AMD-V/SVM activée dans l’UEFI/BIOS |

Les apprenants n’ont **pas besoin d’installer Vagrant ou Docker dans Windows** : les logiciels du lab sont déjà dans les VM.

**3. Installer VirtualBox**

1. Télécharge VirtualBox pour **Windows hosts** depuis le [site officiel](https://www.virtualbox.org/wiki/Downloads).
2. Utilise de préférence **la même version que celle validée par le formateur**.
3. Lance l’installation.
4. Conserve les composants réseau proposés.
5. Accepte l’installation des pilotes Oracle.
6. Redémarre Windows si demandé.

Pour ce lab, la virtualisation imbriquée doit fonctionner : Sunbeam doit pouvoir exécuter des VM à l’intérieur de VirtualBox.

Sur un **PC personnel dédié au lab**, si l’hyperviseur Windows pose problème, ouvre PowerShell en administrateur :

```powershell
bcdedit /set hypervisorlaunchtype off
```

Puis redémarre Windows. Ce réglage empêche notamment les fonctions reposant sur Hyper-V, comme WSL2, de fonctionner dans ce démarrage. Pour le réactiver ultérieurement :

```powershell
bcdedit /set hypervisorlaunchtype auto
```

Si les protections VBS d’un PC administré maintiennent des contraintes, fais valider la configuration par l’administrateur. **Le contrôle décisif sera la présence de `/dev/kvm` dans Sunbeam, à l’étape 8.**

**4. Créer les deux réseaux Host-only**

Dans VirtualBox, ouvre **Outils → Réseau**, ou **Fichier → Outils → Gestionnaire de réseau**, selon la version.

Crée deux adaptateurs **Réseau privé hôte / Host-only**.

Configure-les ainsi :

| Paramètre | Réseau de gestion | Réseau des instances |
|---|---|---|
| Adresse IPv4 de l’adaptateur Windows | `192.168.56.1` | `192.168.57.1` |
| Masque | `255.255.255.0` | `255.255.255.0` |
| Serveur DHCP VirtualBox | **Désactivé** | **Désactivé** |

Note le nom exact de chaque adaptateur. Par exemple :

```text
VirtualBox Host-Only Ethernet Adapter
VirtualBox Host-Only Ethernet Adapter #2
```

**Ne te fie pas au numéro `#2` : vérifie l’adresse IP associée à chaque adaptateur.**

Ne configure aucune passerelle par défaut sur ces adaptateurs Windows. Le PC conserve sa connexion Internet habituelle.

**5. Importer les deux OVA**

Dans VirtualBox :

1. **Fichier → Importer un appareil virtuel**.
2. Sélectionne `Services-base.ova`.
3. Nomme la VM **`Lab-Services`**.
4. Choisis le dossier de stockage sur le SSD.
5. Pour la politique MAC, sélectionne **Conserver toutes les adresses MAC**.
6. Lance l’import.
7. Répète avec `Sunbeam-base.ova`, nommée **`Lab-Sunbeam`**.

Oracle distingue bien la conservation de toutes les MAC de celle des seules cartes NAT : il faut ici l’option équivalente à `keepallmacs`. [Oracle VirtualBox](https://www.virtualbox.org/browser/vbox/trunk/doc/manual/en_US/man_VBoxManage-import.xml?rev=87075\&utm_source=chatgpt.com)

**Ne démarre pas encore les VM.**

**6. Vérifier les cartes réseau des deux VM**

Pour **chacune** : **Configuration → Réseau**.

| Carte | Mode | Réseau à sélectionner |
|---|---|---|
| Adaptateur 1 | **NAT** | NAT simple |
| Adaptateur 2 | **Réseau privé hôte / Host-only** | Adaptateur Windows `192.168.56.1` |
| Adaptateur 3 | **Réseau privé hôte / Host-only** | Adaptateur Windows `192.168.57.1` |

Sur les trois cartes :

- coche **Activer l’interface réseau** ;
- coche **Câble branché** ;
- conserve le type de carte importé ;
- conserve l’adresse MAC importée.

Sur les adaptateurs **2 et 3**, ouvre **Avancé** et règle le **mode promiscuité sur Tout autoriser**.

Les noms des adaptateurs Host-only peuvent différer de ceux du formateur : il faut donc les rattacher manuellement après import.

**7. Activer la virtualisation imbriquée de Sunbeam**

VM **Lab-Sunbeam éteinte** :

1. Ouvre **Configuration → Système → Processeur**.
2. Vérifie la RAM et le nombre de processeurs.
3. Active **VT-x/AMD-V imbriqué** si l’option est disponible.

Tu peux aussi utiliser PowerShell :

```powershell
& "$env:ProgramFiles\Oracle\VirtualBox\VBoxManage.exe" modifyvm "Lab-Sunbeam" --nested-hw-virt on
```

Cette option expose la virtualisation matérielle à la VM. [download.virtualbox.org](https://download.virtualbox.org/virtualbox/7.2.0_RC1/UserManual.pdf?utm_source=chatgpt.com)

**8. Démarrer les VM dans le bon ordre**

Démarre d’abord **Lab-Services**.

Connecte-toi dans sa console VirtualBox avec `vagrant`, puis vérifie :

```bash
ip -br addr
sudo systemctl is-active docker
sudo systemctl is-active dnsmasq
sudo docker ps
```

Tu dois retrouver :

- `192.168.56.20` sur le réseau de gestion ;
- `192.168.57.2` sur le réseau des instances ;
- Docker et dnsmasq actifs ;
- les conteneurs Keycloak et PostgreSQL démarrés.

Démarre ensuite **Lab-Sunbeam**.

Dans sa console :

```bash
ip -br addr
ls -l /dev/kvm
```

L’adresse de gestion attendue est **`192.168.56.10`**.

**Le fichier `/dev/kvm` doit exister.** S’il est absent, il faut résoudre la virtualisation imbriquée avant de tenter de lancer des instances Nova.

L’interface `provider0` peut être intégrée à Open vSwitch : **ne lui ajoute pas manuellement une adresse IP**.

**9. Configurer les noms et les certificats dans Windows**

Place les deux fichiers fournis ensemble :

```text
C:\Lab-OpenStack\Configurer-Lab-Sunbeam.ps1
C:\Lab-OpenStack\ca.crt
```

Ouvre **PowerShell en administrateur**, puis :

```powershell
cd C:\Lab-OpenStack
Unblock-File .\Configurer-Lab-Sunbeam.ps1
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Configurer-Lab-Sunbeam.ps1
```

Le script fourni configure les noms du lab dans le fichier `hosts` de Windows et importe son autorité de certification.

Les correspondances attendues sont :

| Nom | Adresse |
|---|---|
| `sso.lab.test` | `192.168.56.20` |
| `cloud.lab.test` | `192.168.56.201` |
| `internal.lab.test` | `192.168.56.202` |
| `rgw.lab.test` | `192.168.56.203` |

**Avec nos OVA actuelles, Horizon est encore en HTTP** : un contrôle du port 443 peut échouer tant que le HTTPS OpenStack n’est pas configuré.

Vérifie plutôt :

```powershell
Test-NetConnection cloud.lab.test -Port 80
Test-NetConnection sso.lab.test -Port 8443
```

Le résultat attendu est :

```text
TcpTestSucceeded : True
```

**10. Vérifier OpenStack après l’import**

Dans **Lab-Sunbeam**, avec l’utilisateur `vagrant` :

```bash
juju status -m openstack
juju status -m openstack-machines
```

Laisse quelques minutes aux services pour reprendre. Vérifie ensuite :

```bash
sudo k8s get dns
```

Le DNS amont doit contenir **`192.168.56.20`**, conformément à notre correction.

Puis :

```bash
source ~/demo-openrc
openstack network list
openstack subnet list
openstack router list
openstack image list
openstack flavor list
```

Les commandes doivent afficher leurs tableaux sans erreur d’authentification ni erreur 503.

**Ne relance pas `sunbeam cluster bootstrap` ou `sunbeam configure` pour un simple import : les OVA contiennent déjà le cloud configuré.**

**11. Se connecter depuis Windows**

Ouvre :

[http://cloud.lab.test/openstack-horizon](http://cloud.lab.test/openstack-horizon)

Renseigne :

| Champ | Valeur |
|---|---|
| Domaine | `users` |
| Utilisateur | `demo` |
| Mot de passe | Celui transmis par le formateur |

Pour Keycloak :

[https://sso.lab.test:8443/admin/](https://sso.lab.test:8443/admin/)

La connexion à Horizon utilise encore le compte local OpenStack. **La présence de Keycloak ne signifie pas que la fédération SSO est déjà configurée.**

**12. Arrêter et conserver le lab**

En fin de séance :

1. Arrête les instances Nova depuis Horizon.
2. Dans la console **Lab-Sunbeam** :

   ```bash
   sudo shutdown -h now
   ```

3. Attends son arrêt complet.
4. Dans **Lab-Services** :

   ```bash
   sudo shutdown -h now
   ```

Pour reprendre : **Services d’abord, Sunbeam ensuite**.

Après validation de l’import, crée un snapshot des **deux VM éteintes**, nommé par exemple `Base-importee-validee`. Pour revenir à cette base, restaure les deux snapshots correspondants.

**Avant de distribuer à toute la classe, teste ce parcours sur un autre PC jusqu’à la création d’une instance Nova et son accès depuis Windows.** L’ouverture d’Horizon seule ne valide pas encore le calcul, Ceph et le réseau des instances.
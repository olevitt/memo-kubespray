# Notes d'installation Kubespray  

Quelques notes d'installation d'un cluster Kubernetes avec la distribution [Kubespray](https://github.com/kubernetes-sigs/kubespray)

## Rappels : Kubernetes 

Quelques tutos de rappels sur Kubernetes (et la notion sous-jacente : la conteneurisation) : https://github.com/olevitt/tutoriels

## Sur chaque noeud

- OS installé (Ubuntu / Debian 11)
- Adresse IP
- SSH entrant
- /!\ Ne pas mettre de swap /!\

## Sur la machine bootstrap

0. Le bootstrap est une machine dédiée ne servant qu'à l'installation. Elle ne fera pas partie du cluster mais servira de base avancée pour l'installation et le management du cluster.

1. Cloner le depot kubespray ( git clone permet de télecharger le contenu d'un depot)

```
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
git checkout v2.21.0
```

Le `git checkout v2.21.0` permet de se positionner sur une version stable pour éviter les versions de travail (branche master). Se référer aux releases https://github.com/kubernetes-sigs/kubespray/releases

2. Installer Ansible
   https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

3. Installer dépendances pip pour kubespray
   `pip3 install -r requirements.txt`

4. Définir l'inventaire
   https://github.com/kubernetes-sigs/kubespray#usage

- nano inventory/mycluster/hosts.yaml pour le retravailler

5. Configurer l'install

```
nano inventory/mycluster/group_vars/all/all.yml
nano inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

Pour que kubespray génère le kubeconfig sur bootstrap

```
kubeconfig_localhost: true
```

Optionnel : pour activer l'authentification openidconnect (nécessite création d'un client keycloak ou équivalent sur un autre provider OIDC) :

```
kube_oidc_auth: true
kube_oidc_url: https://keycloak.domaine.fr/realms/xxxx
kube_oidc_client_id: apiserver
## Optional settings for OIDC
# kube_oidc_ca_file: "{{ kube_cert_dir }}/ca.pem"
kube_oidc_username_claim: preferred_username
kube_oidc_username_prefix: oidc-
kube_oidc_groups_claim: groups
kube_oidc_groups_prefix: oidc-
```

6. Créer une clé et la déposer sur chaque machine pour avoir l'authentification ssh sans mot de passe

```
ssh-keygen
ssh-copy-id 192.168.x.y # 1 fois pour chaque machine de l'inventaire
```

Tester l'accès réseau aux machines :
Se positionner dans le dossier kubespray

```
ansible all -m ping -i inventory/mycluster/hosts.yaml
```

7. Lancement de l'install  
   `ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml -K`

Ajout de -K pour qu'il demande le mot de passe pour devenir root (sudo) sur les machines

## Post-install

1. Configuration d'un poste d'administration

- Installer `kubectl` : [Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) / [Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- Récupérer un fichier kubeconfig (un fichier `cluster-admin` est disponible dans `kubespray/inventory/mycluster/artifacts/admin.conf` sur la machine bootstrap). A défaut, on pourra se positionner sur un des masters pour administrer.
- Mettre le fichier kubeconfig au bon endroit : `~/.kube/config` (remplacer `~` par `$HOME` sur windows)

Pour valider l'install : `kubectl version` doit renvoyer la version du serveur en plus de celle du client. `kubectl get nodes` doit renvoyer la liste des noeuds présents dans le cluster.

2. Optionnel : Authentification OpenIDConnect (authentification déléguée via keycloak)

L'APIServer a été configuré pour valider les tokens signés par keycloak (cf configuration lors de l'install).  
Pour que kubectl utilise l'authentification openidconnect, on peut utiliser le plugin https://github.com/int128/kubelogin.  
Mise en place : https://github.com/int128/kubelogin#setup

- Télécharger la dernière release pour le bon système d'exploitation (`kubelogin_windows_amd64.zip
` ou `kubelogin_linux_amd64.zip`)
- Mettre le binaire présent dans le zip dans le PATH de votre système (ex : dossier INSTALL dans votre HOME), le renommer en `kubectl-oidc_login` (pour linux) ou `kubectl-oidc_login.exe` (pour windows) (le renommage est important pour que le système de plugins `kubectl` fonctionne)
- Optionnel : tester avec la commande `kubectl oidc-login setup --oidc-issuer-url https://keycloak.domaine.fr/realms/xxxx --oidc-client-id apiserver --oidc-client-secret dummysecret`
- Modifier la configuration user dans le fichier `kubeconfig` pour s'authentifier via oidc plutôt qu'avec le certificat root :

```
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://keycloak.domaine.fr/realms/xxxx
      - --oidc-client-id=apiserver
      - --oidc-client-secret=dummysecret
```

- Maintenant, n'importe quelle commande kubectl (par exemple `kubectl get nodes`) doit ouvrir le navigateur pour demander de s'authentifier sur keycloak
- Si vous avez un message du type `Error from server (Forbidden): pods is forbidden: User "oidc-olevitt" cannot list resource "pods" in API group "" in the namespace "olevitt"`, cela signifie que l'authentification s'est bien passée (cf `oidc-olevitt`, votre identifiant a été préfixé par `oidc-`) mais qu'aucun droit n'est associé à l'utilisateur `oidc-olevitt`. Vous ne pouvez donc rien faire sans ajout explicite de droits pour votre utilisateur (cf paragraphe suivant).

3. Gestion des droits

Par défaut, les comptes n'ont pas de droit (d'où le message d'erreur `Forbidden` même une fois connécté).  
Les associations de droits se font via des objets `RoleBinding` (pour les droits locaux à un `namespace`) ou `ClusterRoleBinding` (pour les droits globaux) qui sont des associations entre des `Subjects` (`ServiceAccount`, `Group` ou `User`) et des `Roles` (ou `ClusterRoles`).  
Exemple pour donner les droits globaux `cluster-admin` à l'utilisateur `oidc-olevitt` :  
Dans un fichier `crb.yaml` :

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-olevitt
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: oidc-olevitt
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Puis appliquer cet objet (en utilisant un compte ayant les droits `cluster-admin`) :  
`kubectl apply -f crb.yaml`  

Si vous n'avez pas fait l'authentification déléguée du chapitre précédent, vous pouvez créer des comptes ad-hoc (`ServiceAccount`) et leur générer un token. Cf 

4. Installation de Helm

Helm est un gestionnaire de paquets qui facilite grandement l'installation, la configuration et la montée de version des logiciels.

- Télécharger la bonne version dans les releases github : https://github.com/helm/helm/releases
- Mettre le binaire dans le PATH de votre machine
- Pour test l'installation : `helm ls`
- Exemple d'installation d'un `httpbin` basique à partir du dépôt `inseefrlab/helm-charts` : https://github.com/InseeFrLab/helm-charts

```
helm repo add inseefrlab https://inseefrlab.github.io/helm-charts # ajout du depot
helm search repo # pour voir ce qui est installable
helm template httpbin inseefrlab/httpbin # pour voir les manifests qui sont générés
helm install httpbin inseefrlab/httpbin # installation basique sans configuration
helm ls # on voit bien httpbin installé
kubectl get pods # les pods ont bien été créés
helm upgrade httpbin inseefrlab/httpbin --set replicaCount=3 # mise à jour de l'installation avec une nouvelle configuration
kubectl get pods # on a bien 3 pods maintenant
helm uninstall httpbin # on désinstalle tout
helm ls # c'est déinstallé
kubectl get pods # c'est propre :)
```

5. Installation du reverse proxy

- Doc d'install : https://kubernetes.github.io/ingress-nginx/deploy/
- Values surchargeables : https://github.com/kubernetes/ingress-nginx/blob/main/charts/ingress-nginx/values.yaml
- [Config](apps/ingress-nginx/values.yaml)

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx -f values.yaml
```

Comme `nodeSelector: reverse-proxy: "true"`, il faut labelliser les noeuds sur lesquels on veut que le reverse-proxy se lance :

```
kubectl label node worker1 reverse-proxy="true"
kubectl label node worker2 reverse-proxy="true"
```

Verification de la bonne installation :  
`kubectl get pods -n ingress-nginx`  
Comme le reverse-proxy est en mode `hostNetwork: true`, il écoute directement sur les noeuds donc on peut y accéder via `https://ipduworker1 et https://ipduworker2

6. Certificat HTTPS du reverse-proxy

Par défaut, le reverse-proxy génère un certificat auto-signé `Kubernetes Ingress Controller Fake Certificate` qui n'est du coup pas reconnu par les applications et les systèmes.  
Il faut donc générer un certificat puis le donner à ingress-nginx.  
Pour générer le certificat, on peut utiliser `let's encrypt` via le logiciel `certbot`.

- Installer Certbot : https://certbot.eff.org/instructions?ws=other&os=debianbuster
- `certbot --manual --preferred-challenges dns certonly`
- Modifier les enregistrements DNS publics comme demandés par certbot
- Le certificat généré est dans `/etc/letsencrypt/live/...`
- Prendre les fichiers `fullchain.pem` et `privkey.pem`
- Pour le fichier `fullchain.pem`, supprimer le 3ème bloc

```
kubectl create secret tls wildcard -n ingress-nginx --key privkey.pem --cert fullchain.pem
```

- Mettre à jour la configuration du ingress-nginx

```
controller:
  extraArgs:
    default-ssl-certificate: ingress-nginx/wildcard
```

```
helm upgrade ingress-nginx ingress-nginx/ingress-nginx -f values.yaml -n ingress-nginx
```

7. Noeuds particuliers

Appliquer des taints / tolerations pour guider les déploiements : https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

```
kubectl taint nodes machineinfra infra=true:NoSchedule
```

Effet : seuls les déploiements qui ont une tolération `infra` pourront être schedulés (affectés) au noeud `machineinfra`

Pour appliquer une toleration :

```
tolerations:
  - key: "infra"
    operator: "Exists"
    effect: "NoSchedule"
```

Pour ne se déployer que sur la machine `machineinfra` :

```
nodeSelector:
  kubernetes.io/hostname: machineinfra
```

8. GPU

Une fois pour toute :

```
containerd_additional_runtimes:
  - name: nvidia
    type: "io.containerd.runc.v2"
    engine: ""
    root: ""
    options:
      BinaryName: "\"/usr/bin/nvidia-container-runtime\""
      systemdCgroup: "true"
```

dans `inventory/mycluster/group_vars/all/containerd.yml`

Sur chaque machine :

Install driver à la main parce que operator supporte pas debian 10/11 ?

```
sudo apt install build-essential
sudo apt install --reinstall linux-headers-$(uname -r)
BASE_URL=https://us.download.nvidia.com/tesla
DRIVER_VERSION=525.60.13
curl -fSsl -O $BASE_URL/$DRIVER_VERSION/NVIDIA-Linux-x86_64-$DRIVER_VERSION.run
sudo sh NVIDIA-Linux-x86_64-$DRIVER_VERSION.run
```

Install toolkit

```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)     && curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -     && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update     && sudo apt-get install -y nvidia-container-toolkit
```

Workaround chelou pour bug chelou (https://github.com/NVIDIA/nvidia-docker/issues/614#issuecomment-423991632) :

```
sudo ln -s /sbin/ldconfig /sbin/ldconfig.real
```

Autre bug possible : https://github.com/NVIDIA/gpu-operator/issues/441  
Workaround :

```
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288
```

dans `/etc/sysctl.d/99-kubernetes-cri.conf` puis `sudo sysctl --system`

Modifier l'inventaire `kubespray` pour activer le runtime nvidia pour les noeuds concernés :

```
sp-kw01:
      ansible_host: 172.16.110.186
      ip: 172.16.110.186
      access_ip: 172.16.110.186
      containerd_default_runtime: nvidia
```

(rajouter la dernière ligne)

Pour vérifier l'installation des GPU :  
`nvidia-smi` sur une machine  
`kubectl describe node xxx` et chercher la partie `Allocatable` :

```
Allocatable:
  cpu:                256
  ephemeral-storage:  848160123934
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1056486412Ki
  nvidia.com/gpu:     7             <====== nombre total de GPU à dispositon des pods sur ce noeud
  pods:               110
```

Pour le split des GPU (MIG) : https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/gpu-operator-mig.html#configuring-mig-profiles

```
kubectl label nodes sp-kw01 nvidia.com/mig.config=all-1g.5gb --overwrite # 7x5 = 40
kubectl label nodes sp-gossner nvidia.com/mig.config=all-1g.10gb --overwrite # 7x10 = 80
```

A noter : max 7 instances par GPU (cf https://docs.nvidia.com/datacenter/tesla/mig-user-guide/#supported-gpus)

---

### Gestion des noeuds / upgrades

https://github.com/kubernetes-sigs/kubespray/blob/master/docs/nodes.md

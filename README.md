# TP 1 - Utilisation de Helm


Installer Helm https://helm.sh/docs/intro/install/ 

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```


**Déployer un nextcloud en utilisant la chart helm officielle**

https://artifacthub.io/packages/helm/nextcloud/nextcloud

- On doit d'abord créer un cluster kind

```
kind create cluster
```

**Ajouter et déployer nextcloud avec Helm Chart**

```
helm repo add nextcloud https://nextcloud.github.io/helm/
> "nextcloud" has been added to your repositories

helm install my-release nextcloud/nextcloud

NAME: my-release
LAST DEPLOYED: Thu Jan 26 09:20:56 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
#######################################################################################################
## WARNING: You did not provide an external database host in your 'helm install' call                ##
## Running Nextcloud with the integrated sqlite database is not recommended for production instances ##
#######################################################################################################

For better performance etc. you have to configure nextcloud with a resolvable database
host. To configure nextcloud to use and external database host:


1. Complete your nextcloud deployment by running:

  export APP_HOST=127.0.0.1
  export APP_PASSWORD=$(kubectl get secret --namespace default my-release-nextcloud -o jsonpath="{.data.nextcloud-password}" | base64 --decode)

  ## PLEASE UPDATE THE EXTERNAL DATABASE CONNECTION PARAMETERS IN THE FOLLOWING COMMAND AS NEEDED ##

 
```


```
helm upgrade my-release nextcloud/nextcloud \
    --set nextcloud.password=$APP_PASSWORD,nextcloud.host=$APP_HOST,service.type=ClusterIP,mariadb.enabled=false,externalDatabase.user=nextcloud,externalDatabase.database=nextcloud,externalDatabase.host=YOUR_EXTERNAL_DATABASE_HOST
    
Release "my-release" has been upgraded. Happy Helming!
NAME: my-release
LAST DEPLOYED: Thu Jan 26 09:27:26 2023
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
1. Get the nextcloud URL by running:

  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=nextcloud" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080/
  kubectl port-forward $POD_NAME 8080:80

2. Get your nextcloud login credentials by running:

  echo User:     admin
  echo Password: $(kubectl get secret --namespace default my-release-nextcloud -o jsonpath="{.data.nextcloud-password}" | base64 --decode)
```

**Se connecter a Nextcloud**

```
kubectl port-forward $POD_NAME 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

![](https://i.imgur.com/itfyr44.png)




**Créer un fichier values.yaml pour augmenter le nombre de replicas. Déployer la mise à jour.**


- création du fichier values

```
nano values.yml
replicaCount: 3
```

- Update avec le fichier values.yml

```
helm upgrade my-release nextcloud/nextcloud --set nextcloud.password=$APP_PASSWORD,nextcloud.host=$APP_HOST,service.type=ClusterIP,mariadb.enabled=false,externalDatabase.user=nextcloud,externalDatabase.database=nextcloud,externalDatabase.host=YOUR_EXTERNAL_DATABASE_HOST -f values.yml
```

- Et on vérifie le nombre de pod suite à l'update

```
kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
my-release-nextcloud-8c65698f9-qhcnt   1/1     Running   0          48m
my-release-nextcloud-8c65698f9-lm25p   1/1     Running   0          48m
my-release-nextcloud-8c65698f9-45jn7   1/1     Running   0          48m
```



# TP 2 - Créer sa première chart

- Créer le cluster kind en ouvrant le port 80 et 443

```
cat <<EOF | kind create cluster --config=
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

- Ingress Controler

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

- On crée le chemin pour Nginx

```
helm create tp2-helm
> Creating tp2-helm
```

```
└── tp2-helm
    ├── Chart.yaml
    ├── charts
    ├── templates
    │   ├── NOTES.txt
    │   ├── _helpers.tpl
    │   ├── deployment.yaml
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── service.yaml
    │   ├── serviceaccount.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
```

**Les déploiements, services, ingress, … seront donc créés par cette chart. L’image, tag, ports, type de service doivent être modifiables à l’aide d’un fichier values.yaml.**

- Les fichiers services et ingress n’ont pas besoin d’être modifiés
- On supprime le fichier serviceaccount.yaml qui n'est pas utilisé
- On modifie le fichier de déploiement pour utiliser seulement les paramètres qui nous concerne


**2 fichiers values-(prod / dev).yaml devront être créés. Le fichier prod déploiera un serveur nginx en version 1.22 avec comme domaine srv-prod.test dans le namespace production. Le fichier dev déploiera un serveur nginx en version 1.23 avec comme domaine srv-dev.test dans le namespace development.**

- Création des 2 fichiers **values-prod.yaml** et **values-dev.yaml**

```
nano tp2-helm/values-prod.yaml
nano tp2-helm/values-dev.yaml
```

- Paramètre des 2 fichiers pour charger la configuration par défaut du values.yaml

    - Ajouter le tag
    - Ajouter le host srv-prod.test et srv-dev.test

- déployer les 2 environnements nginx

    - values-prod.yaml / values-dev.yaml
    - Préciser le repository tp2-helm avec ‘.’
    - Donner un nom : nginx-prod / nginx-dev
    - Nom du namespace : production / development

```
$ helm install -f values-prod.yaml nginx-prod . -n production --create-namespace
NAME: nginx-prod
LAST DEPLOYED: Thu Jan 26 13:51:14 2023
NAMESPACE: production
STATUS: deployed
REVISION: 1

$ kubectl get pods -n development
NAME                                  READY   STATUS    RESTARTS   AGE  
nginx-dev-my-nginx-7o69e7g6v7-6pm45   1/1     Running   0          3m42s

$ helm install -f values-dev.yaml nginx-dev . -n development --create-namespace
NAME: nginx-dev
LAST DEPLOYED: Thu Jan 26 13:53:40 2023
NAMESPACE: development
STATUS: deployed
REVISION: 1
```

- On vérifie les pods

```
kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS      RESTARTS   AGE
production           nginx-prod-tp2-helm-5f84z54781-d2tgn              1/1     Running     0          23m
development          nginx-dev-tp2-helm-4fc87gc4cc-25663               1/1     Running     0          23m 
```


- On modifie le fichier /etc/hosts

```
nano /etc/hosts
192.168.222.124 srv-prod.test
192.168.222.124 srv-dev.test
```

- Accès à la page web Nginx

![](https://i.imgur.com/jVmKKyN.png)


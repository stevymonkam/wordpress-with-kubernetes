# Wordpress with Kubernetes


------------

Auteur : MONKAM NAKTAKWEN STEVY

LinkedIn : https://www.linkedin.com/in/stevy-monkam-naktakwen-a84299181/


## Voici le schéma d'infrastructure que je propose

![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20125644.png)

## Étape 1 : Création du namespace
Pour commencer, j'ai créé un namespace `dev` afin de séparer mes ressources des autres environnements.
 
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```


## Étape 2 : Persistent Volumes et Persistent Volume Claims

Ensuite, j'ai créé des Persistent Volumes (PV) et des Persistent Volume Claims (PVC) pour mes deux applications : MySQL et WordPress. J'ai choisi d'utiliser des volumes persistants pour assurer la persistance des données entre les mises à jour et les redémarrages de mes conteneurs.

Les PV et PVC sont configurés pour un accès en lecture et écriture unique (ReadWriteOnce). Les PVC demandent une capacité de stockage de 5 Gi pour MySQL et de 2 Gi pour WordPress.

Attention : ici les données sont stockée en local sur le cluster k8s

```yaml
# MySQL PV et PVC
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: standard
  hostPath:
    path: "/mnt/mysql-data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: dev
spec:
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

```yaml
# WordPress PV et PVC
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: standard
  hostPath:
    path: "/mnt/data-wordpress"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: dev
spec:
  resources:
    requests:
      storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

## Étape 3 : ConfigMap
J'ai créé une ConfigMap pour stocker les informations de configuration partagées entre MySQL et WordPress. Cela me permet de centraliser la configuration et de faciliter les mises à jour.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-mysql-config
  namespace: dev
data:
  WORDPRESS_DB_HOST: mysql-svc
  WORDPRESS_DB_USER: wp_user
  WORDPRESS_DB_PASSWORD: wp_password
  MYSQL_ROOT_PASSWORD: root_password
  WORDPRESS_DB_NAME: wordpress
```

## Étape 4 : Déploiements
J'ai créé des déploiements pour MySQL et WordPress avec la stratégie de déploiement Recreate, qui assure que les anciens pods sont supprimés avant de créer les nouveaux.

Les déploiements utilisent les images officielles de MySQL et WordPress et récupèrent les informations de configuration depuis la ConfigMap.

```yaml
# Déploiement MySQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: dev
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_NAME
        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_PASSWORD
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: MYSQL_ROOT_PASSWORD
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
```
```yaml
# Déploiement WordPress
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: dev
spec:
  strategy:
    type: Recreate
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:6
        env:
        - name: WORDPRESS_DB_HOST
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_HOST
        - name: WORDPRESS_DB_USER
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_USER
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_PASSWORD
        - name: WORDPRESS_DB_NAME
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql-config
              key: WORDPRESS_DB_NAME
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

## Étape 5 : Services
J'ai créé un service ClusterIP (backend) pour exposer l'applications MySQL à l'intérieur du cluster Kubernetes, tandis que le service que j'ai créé pour WordPress est de type NodePort (frontend) pour permettre l'accès depuis l'extérieur du cluster.

```yaml
# Service MySQL
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: dev
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
```

```yaml
# Service WordPress
apiVersion: v1
kind: Service
metadata:
  name: wordpress-svc
  namespace: dev
spec:
  selector:
    app: wordpress
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

À ce stade, notre application est déjà accessible depuis l'extérieur via l'adresse IP virtuelle (VIP) du cluster ou celle de l'un des nœuds, suivie du port 30080. Cependant, je souhaitais rendre mon application accessible via une URL simple. Pour ce faire, j'ai créé une règle Ingress permettant d'exposer mon application WordPress à l'aide d'un domaine personnalisé.


## Étape 6 : Ingress

# Configuration de MetalLB et Ingress Nginx pour Kubernetes

Ce dépôt contient des fichiers de configuration YAML et des instructions pour configurer MetalLB et déployer le contrôleur Ingress Nginx dans un cluster Kubernetes.

## MetalLB

MetalLB est un contrôleur de réseau pour Kubernetes qui permet la gestion des adresses IP pour les services exposés dans le cluster. Dans ce dépôt, vous trouverez les fichiers de configuration nécessaires pour déployer MetalLB.

### Instructions d'installation

Utilisez la commande suivante pour appliquer la configuration de MetalLB :

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```


Assurez-vous de vérifier que les ressources ont été déployées avec succès en utilisant la commande :

```
get all -n metallb-system
```
```yaml
 ipaddresspool.yml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.99.150-192.168.99.255 # set your local subnet free range
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
~             
```
```
 kubectl apply -f ipaddresspool.yml
```
Cette configuration permet à MetalLB d'allouer des adresses IP à partir du pool spécifié et de les annoncer sur le réseau local pour les services au sein de votre cluster Kubernetes.

```
 kubectl get IPAddresspool -n metallb-system
```

## installation ingress
```
 kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml  
```
```
kubectl get svc -n ingress-nginx -o wide
```
![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Immagine%202024-04-26%20124811.png)
![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%20(106).png)

```
 kubectl get po -n ingress-nginx
 kubectl get svc -n ingress-nginx
 kubectl get all -n dev
```
![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20130202.png)

En effet, lorsque http://dev-wordpress.pozos.fr sera sollicité, c'est l'ingress controller qui recevra la requête et la redirigera vers le service wordpress-svc à l'intérieur du cluster sur le port 80. Ce service se chargera ensuite de joindre le pod à l'intérieur duquel tourne le conteneur wordpress.


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-wordpress-ingress
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: dev-wordpress.pozos.fr
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: wordpress-svc
            port:
              number: 80
```

## Étape 7 : Déploiement
Pour déployer ces appplications avec leurs eléments infrastructure, j'utilise mon manifest de déploiement tout-en-un :


![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20125750.png)


## Étape 8 : Consommation de l'application

Après le déploiement, vous pouvez accéder à votre application WordPress en utilisant l'adresse http://dev-wordpress.pozos.fr (en supposant que vous ayez correctement configuré la résolution DNS pour pointer vers votre cluster Kubernetes).

![suggested-architecture](https://github.com/Abdel-had/mini-projet-kubernetes/blob/main/img/suggested-architecture.jpg)

--------

# Conclusion

En conclusion, ce projet Kubernetes m'a permis de mettre en place une infrastructure solide pour déployer une application WordPress couplée à une base de données MySQL. Grâce à l'utilisation des ressources Kubernetes telles que les namespaces, les volumes persistants, les ConfigMaps, les déploiements, les services et les Ingress, j'ai pu créer un environnement flexible, évolutif et facilement maintenable pour cette application.



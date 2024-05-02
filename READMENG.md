# Wordpress with Kubernetes


------------

Author: MONKAM NAKTAKWEN STEVY

LinkedIn : https://www.linkedin.com/in/stevy-monkam-naktakwen-a84299181/


## Here is the infrastructure plan that I proposed

![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20125644.png)

## Step 1: Creation of the namespace
To begin with, I have created a namespace `dev` to separate the resources of other environments.
 
```yaml
apiVersion: v1
kind: Namespace
metadata:
   name: dev
```


## Step 2: Persistent Volumes and Persistent Volume Claims

Ensuite, j'ai créé des Persistent Volumes (PV) et des Persistent Volume Claims (PVC) pour mes deux applications: MySQL et WordPress. I choose to use persistent volumes to ensure the persistence of women between daily wear and redundancy of my containers.

The PV and PVC are configured for unique reading and writing access (ReadWriteOnce). The PVC requires a storage capacity of 5 Gi for MySQL and 2 Gi for WordPress.

Attention: here the women are stocked locally on the k8s cluster

```yaml
# MySQL PV and PVC
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mysql-pv
specs:
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
specs:
   resources:
     requests:
       storage: 5Gi
   volumeMode: Filesystem
   accessModes:
     - ReadWriteOnce
```

```yaml
# WordPress PV and PVC
apiVersion: v1
kind: PersistentVolume
metadata:
   name: wordpress-pv
specs:
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
specs:
   resources:
     requests:
       storage: 2Gi
   volumeMode: Filesystem
   accessModes:
     - ReadWriteOnce
```

## Stage 3 : ConfigMap
I created a ConfigMap to stock the configuration information shared between MySQL and WordPress. This box allows me to centralize the configuration and facilitate daily setups.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
   name: wordpress-mysql-config
   namespace: dev
date:
   WORDPRESS_DB_HOST: mysql-svc
   WORDPRESS_DB_USER: wp_user
   WORDPRESS_DB_PASSWORD: wp_password
   MYSQL_ROOT_PASSWORD: root_password
   WORDPRESS_DB_NAME: wordpress
```

## Stage 4 : Déploiements
I have created deployments for MySQL and WordPress with the Recreate deployment strategy, here I assure you that the old pods are deleted before creating the new ones.

The deployment uses the official images of MySQL and WordPress and retrieves the configuration information from the ConfigMap.

```yaml
# Deployment MySQL
apiVersion: apps/v1
kind: Deployment
metadata:
   name: mysql
   namespace: dev
specs:
   replies: 1
   strategy:
     type: Recreate
   selector:
     matchLabels:
       app: mysql
   templates:
     metadata:
       labels:
         app: mysql
     specs:
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
# Deployment WordPress
apiVersion: apps/v1
kind: Deployment
metadata:
   name: wordpress
   namespace: dev
specs:
   strategy:
     type: Recreate
   replies: 1
   selector:
     matchLabels:
       app: wordpress
   templates:
     metadata:
       labels:
         app: wordpress
     specs:
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

## Stage 5: Services
I created a ClusterIP service (backend) to expose MySQL applications inside the Kubernetes cluster, so the service I created for WordPress is of the NodePort (frontend) type to allow access from there exterior of the cluster.

```yaml
# Service MySQL
apiVersion: v1
kind: Service
metadata:
   name: mysql-svc
   namespace: dev
specs:
   selector:
     app: mysql
   ports:
   - port: 3306
     targetPort: 3306
   type: ClusterIP
```

```yaml
# WordPress Service
apiVersion: v1
kind: Service
metadata:
   name: wordpress-svc
   namespace: dev
specs:
   selector:
     app: wordpress
   ports:
   - port: 8080
     targetPort: 80
     nodePort: 30080
   type: NodePort
```

At this stage, our application is now accessible from the exterior via the virtual IP (VIP) address of the cluster or cell of one of the nodes, on the port side of 30080. However, this makes the application accessible via a simple URL . Pour ce faire, j'ai créé une règle Ingress permettant d'exposer mon WordPress application à l'aide d'un domaine personalisé.

## Stage 6: ingress

# Configuration of MetalLB and Nginx Input for Kubernetes

Here you will find YAML configuration files and instructions for configuring MetalLB and deploying the Nginx input controller in a Kubernetes cluster.

## MetalLB

MetalLB is a Kubernetes controller that allows you to manage IP addresses for services displayed in the cluster. Here you will find the necessary configuration files for deploying MetalLB.

### Installation instructions

Use the following command to apply the MetalLB configuration:

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```


Please ensure that the resources you have deployed have been successfully used using the command:

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
specs:
   addresses:
   - 192.168.99.150-192.168.99.255 # set your local subnet free range
   autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
   name: default
   namespace: metallb-system
specs:
   ipAddressPools:
   - default
~
```
```
  kubectl apply -f ipaddresspool.yml
```
This configuration allows MetalLB to allocate IP addresses from a specific pool and to announce the local network for services in your Kubernetes cluster.

```
  kubectl get IPAddresspool -n metallb-system
```

## installation input
```
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```
```
kubectl get svc -n ingress-nginx -o wide
```
![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Image%202024-04-26%20124811.png)
![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%20(106).png)

```
  kubectl get po -n ingress-nginx
  kubectl get svc -n ingress-nginx
  kubectl get all -n dev
```
![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20130202.png)

Indeed, as http://dev-wordpress.pozos.fr will be requested, the input controller here will receive the request and redirect it to the wordpress-svc service inside the cluster on port 80. service is charged ensuite by joining the pod to the inside of the wordpress container.


```yaml
apiVersion: networking.k8s.io/v1
kind: Input
metadata:
   name: dev-wordpress-ingress
   namespace: dev
   annotations:
     kubernetes.io/ingress.class: "nginx"
specs:
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

## Stage 7 : Deployment
Pour deployer ces applications avec leurs eléments infrastructure, j'utilise la commande
```
kubectl apply -f <filename>
```

## Step 8: Consummation of the application

After deployment, you can access your WordPress application using the address http://dev-wordpress.pozos.fr (assuming you have correctly configured the DNS resolution to point to your Kubernetes cluster).

![suggested-architecture](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20125750.png)

--------

# Conclusion

In conclusion, the Kubernetes project allows me to put in place a solid infrastructure to deploy a WordPress application coupled on a MySQL database. Thanks to the use of Kubernetes resources, you can create a flexible, evolutionary and easily maintainable environment for this application.
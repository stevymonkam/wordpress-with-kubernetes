# Wordpress con Kubernetes


------------

Autore: MONKAM NAKTAKWEN STEVY

LinkedIn: https://www.linkedin.com/in/stevy-monkam-naktakwen-a84299181/


## Ecco il piano infrastrutturale che ho proposto

![architettura-suggerita](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20125644.png)

## Passaggio 1: creazione dello spazio dei nomi
Per cominciare, ho creato uno spazio dei nomi "dev" per separare le risorse di altri ambienti.
 
```yaml
APIVersione: v1
tipo: spazio dei nomi
metadati:
    nome: div
```


## Passaggio 2: Volumi persistenti e richieste di volume persistente

Ensuite, ho creato Persistent Volumes (PV) e Persistent Volume Claims (PVC) per le mie due applicazioni: MySQL e WordPress. Scelgo di utilizzare volumi persistenti per garantire la persistenza delle donne tra l'usura quotidiana e la ridondanza dei miei contenitori.

PV e PVC sono configurati per l'accesso univoco in lettura e scrittura (ReadWriteOnce). Il PVC richiede una capacità di archiviazione di 5 Gi per MySQL e 2 Gi per WordPress.

Attenzione: qui le donne sono rifornite localmente sul cluster k8

```yaml
# MySQL PV e PVC
APIVersione: v1
tipo: PersistentVolume
metadati:
    nome: mysql-pv
Specifiche:
    capacità:
      memoria: 5Gi
    modalità volume: file system
    modalità di accesso:
      - LeggiScriviUna volta
    persistentVolumeReclaimPolicy: Riciclare
    storageClassName: standard
    percorsohost:
      percorso: "/mnt/mysql-data"
---
APIVersione: v1
tipo: PersistentVolumeClaim
metadati:
    nome: mysql-pvc
    spazio dei nomi: dev
Specifiche:
    risorse:
      richieste:
        memoria: 5Gi
    modalità volume: file system
    modalità di accesso:
      - LeggiScriviUna volta
```

```yaml
# WordPress PV e PVC
APIVersione: v1
tipo: PersistentVolume
metadati:
    nome: wordpress-pv
Specifiche:
    capacità:
      spazio di archiviazione: 2Gi
    modalità volume: file system
    modalità di accesso:
      - LeggiScriviUna volta
    persistentVolumeReclaimPolicy: Riciclare
    storageClassName: standard
    percorsohost:
      percorso: "/mnt/data-wordpress"
---
APIVersione: v1
tipo: PersistentVolumeClaim
metadati:
    nome: wordpress-pvc
    spazio dei nomi: dev
Specifiche:
    risorse:
      richieste:
        spazio di archiviazione: 2Gi
    modalità volume: file system
    modalità di accesso:
      - LeggiScriviUna volta
```

## Fase 3: ConfigMap
Ho creato una ConfigMap per archiviare le informazioni di configurazione condivise tra MySQL e WordPress. Questa casella mi consente di centralizzare la configurazione e facilitare le impostazioni quotidiane.

```yaml
APIVersione: v1
tipo: ConfigMap
metadati:
    nome: wordpress-mysql-config
    spazio dei nomi: dev
data:
    WORDPRESS_DB_HOST: mysql-svc
    WORDPRESS_DB_USER: wp_utente
    WORDPRESS_DB_PASSWORD: wp_password
    MYSQL_ROOT_PASSWORD: password_root
    WORDPRESS_DB_NAME: WordPress
```

## Fase 4: Dispiegamenti
Ho creato dei deploy per MySQL e WordPress con la strategia Recreate deploy, qui ti assicuro che i vecchi pod vengono cancellati prima di creare quelli nuovi.

La distribuzione utilizza le immagini ufficiali di MySQL e WordPress e recupera le informazioni di configurazione da ConfigMap.

```yaml
# Distribuzione MySQL
apiVersion: app/v1
tipo: distribuzione
metadati:
    nome: mysql
    spazio dei nomi: dev
Specifiche:
    risposte: 1
    strategia:
      tipo: Ricrea
    selettore:
      matchEtichette:
        applicazione: mysql
    modelli:
      metadati:
        etichette:
          applicazione: mysql
      Specifiche:
        contenitori:
        - nome: mysql
          immagine: mysql:8
          ambiente:
          - nome: MYSQL_DATABASE
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_NAME
          - nome: MYSQL_USER
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_USER
          - nome: MYSQL_PASSWORD
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_PASSWORD
          - nome: MYSQL_ROOT_PASSWORD
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: MYSQL_ROOT_PASSWORD
          porti:
          - portocontainer: 3306
          volumeMontaggi:
          - nome: mysql-data
            percorsomontaggio: /var/lib/mysql
        volumi:
        - nome: mysql-data
          persistentVolumeClaim:
            nomerichiesta: mysql-pvc
```
```yaml
# Distribuzione WordPress
apiVersion: app/v1
tipo: distribuzione
metadati:
    nome: wordpress
    spazio dei nomi: dev
Specifiche:
    strategia:
      tipo: Ricrea
    risposte: 1
    selettore:
      matchEtichette:
        applicazione: wordpress
    modelli:
      metadati:
        etichette:
          applicazione: wordpress
      Specifiche:
        contenitori:
        - nome: wordpress
          immagine: wordpress:6
          ambiente:
          - nome: WORDPRESS_DB_HOST
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_HOST
          - nome: WORDPRESS_DB_USER
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_USER
          - nome: WORDPRESS_DB_PASSWORD
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_PASSWORD
          - nome: WORDPRESS_DB_NAME
            valoreDa:
              configMapKeyRef:
                nome: wordpress-mysql-config
                chiave: WORDPRESS_DB_NAME
          porti:
          - porto container: 80
          volumeMontaggi:
          - nome: dati wordpress
            percorsomontaggio: /var/www/html
        volumi:
        - nome: dati wordpress
          persistentVolumeClaim:
            nomerichiesta: wordpress-pvc
```

## Fase 5: Servizi
Ho creato un servizio ClusterIP (backend) per esporre le applicazioni MySQL all'interno del cluster Kubernetes, quindi il servizio che ho creato per WordPress è di tipo NodePort (frontend) per consentire l'accesso da lì all'esterno del cluster.

```yaml
# Servizio MySQL
APIVersione: v1
gentile: servizio
metadati:
    nome: mysql-svc
    spazio dei nomi: dev
Specifiche:
    selettore:
      applicazione: mysql
    porti:
    - porto: 3306
      porta di destinazione: 3306
    tipo: ClusterIP
```

```yaml
#Servizio WordPress
APIVersione: v1
gentile: servizio
metadati:
    nome: wordpress-svc
    spazio dei nomi: dev
Specifiche:
    selettore:
      applicazione: wordpress
    porti:
    - porto: 8080
      porta di destinazione: 80
      porta nodo: 30080
    tipo: NodePort
```

In questa fase, la nostra applicazione è ora accessibile dall'esterno tramite l'indirizzo IP virtuale (VIP) del cluster o della cella di uno dei nodi, sul lato porta di 30080. Ciò rende tuttavia l'applicazione accessibile tramite un semplice URL. Per farlo, ho creato una regola di ingresso che consente di esporre l'applicazione WordPress a supporto di un dominio personalizzato.

## Fase 6: ingress

# Configurazione di MetalLB e Nginx Input per Kubernetes

Qui troverai i file di configurazione YAML e le istruzioni per configurare MetalLB e distribuire il controller di input Nginx in un cluster Kubernetes.

##MetalloLB

MetalLB è un controller Kubernetes che consente di gestire gli indirizzi IP per i servizi visualizzati nel cluster. Qui troverai i file di configurazione necessari per la distribuzione di MetalLB.

### Istruzioni per l'installazione

Utilizzare il comando seguente per applicare la configurazione MetalLB:

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```


Assicurati che le risorse distribuite siano state utilizzate correttamente utilizzando il comando:

```
ottieni all -n metalb-system
```
```yaml
   indirizzoippool.yml

APIVersione: metallb.io/v1beta1
tipo: IPAddressPool
metadati:
    nome: predefinito
    spazio dei nomi: metallb-system
Specifiche:
    indirizzi:
    - 192.168.99.150-192.168.99.255 # imposta l'intervallo libero della sottorete locale
    Assegnazione automatica: vero
---
APIVersione: metallb.io/v1beta1
tipo: L2Advertisement
metadati:
    nome: predefinito
    spazio dei nomi: metallb-system
Specifiche:
    Pool di indirizzi ip:
    - predefinito
~
```
```
   kubectl apply -f ipaddresspool.yml
```
Questa configurazione consente a MetalLB di allocare indirizzi IP da un pool specifico e di annunciare la rete locale per i servizi nel tuo cluster Kubernetes.

```
   kubectl get IPAddresspool -n metallb-system
```

## input di installazione
```
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```
```
kubectl get svc -n ingresso-nginx -o wide
```
![architettura-suggerita](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Image%202024-04-26%20124811.png)
![architettura-suggerita](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%20(106).png)

```
   kubectl get po -n ingresso-nginx
   kubectl get svc -n ingresso-nginx
   kubectl ottieni tutto -n dev
```
![architettura-suggerita](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20130202.png)

Infatti, quando verrà richiesto http://dev-wordpress.pozos.fr, il controller di input qui riceverà la richiesta e la reindirizzerà al servizio wordpress-svc all'interno del cluster sulla porta 80. Il servizio viene addebitato ensuite unendosi al pod all'interno del contenitore WordPress.


```yaml
APIVersione: networking.k8s.io/v1
tipo: ingresso
metadati:
    nome: dev-wordpress-ingress
    spazio dei nomi: dev
    annotazioni:
      kubernetes.io/ingress.class: "nginx"
Specifiche:
    regole:
    - host: dev-wordpress.pozos.fr
      http:
        percorsi:
        - pathType: prefisso
          sentiero: /
          back-end:
            servizio:
              nome: wordpress-svc
              porta:
                numero: 80
```

## Fase 7: distribuzione
Per distribuire queste applicazioni con i loro elementi infrastrutturali, utilizza il comando
```
kubectl apply -f <nomefile>
```

## Passo 8: Consumazione della domanda

Dopo la distribuzione, puoi accedere alla tua applicazione WordPress utilizzando l'indirizzo http://dev-wordpress.pozos.fr (supponendo che tu abbia configurato correttamente la risoluzione DNS in modo che punti al tuo cluster Kubernetes).

![architettura-suggerita](https://github.com/stevymonkam/wordpress-with-kubernetes/blob/main/img/Screenshot%202024-04-26%20125750.png)

--------

# Conclusione

In conclusione, il progetto Kubernetes mi consente di realizzare una solida infrastruttura per distribuire un'applicazione WordPress accoppiata su un database MySQL. Grazie all'utilizzo delle risorse Kubernetes, puoi creare un ambiente flessibile, evolutivo e facilmente manutenibile per questa applicazione.
# Documentación - Clúster con K3S

## Pasos previos

------

Descargamos la imagen de Raspbian Lite desde la página web oficial de Raspberry:                                                                                      https://www.raspberrypi.org/software/operating-systems/

Nos descargamos también el programa Etcher desde la página oficial: https://www.balena.io/etcher/          Y, a continuación, flasheamos la tarjeta SD de la Raspberry con la imagen que acabamos de descargar.

Cuando Etcher termine, accedemos a la tarjeta SD desde el explorador de archivos y editamos el archivo `cmdline.txt` en el que tenemos que añadir la siguiente línea:

```
cgroup_memory=1 cgroup_enable=memory ip=<DIRECCION-IP>::<GATEWAY>:<MASCARA-DE-RED>:<HOSTNAME>:eth0:off
```

> Las opciones `cgroup_memory=1` y `cgroup_enable=memory` son necesarias para que K3S funcione correctamente, `eth0` es el nombre de la interfaz a la cual queremos asignar la configuración de red y `off` sirve para apagar la autoconfiguración
>

A continuación, editamos también el archivo `config.txt` y añadimos la siguiente línea al final del archivo para forzar a nuestra Raspberry a utilizar la versión de 64 bits de Raspbian.

```shell
arm_64bit=1
```

Abrimos una terminal de Powershell, accedemos a la unidad de la SD de nuestra Raspberry y creamos un nuevo archivo vacío llamado `ssh`. Esto nos permitirá conectarnos por SSH a nuestra Raspberry sin necesidad de configurar nada.

```powershell
new-item ssh
```

Por último, extraemos la tarjeta SD de nuestro equipo, la introducimos en la Raspberry y comprobamos que la red funciona correctamente y que podemos acceder mediante SSH con el usuario `pi`y la contraseña `raspberry` por defecto.

```shell
ping <DIRECCION-IP>
ssh pi@<DIRECCION-IP>
```



## Configuración inicial

------

Para empezar, ejecutamos el siguiente comando que nos mostrará un menú desde el cual podremos configurar nuestra Raspberry fácilmente.

```shell
sudo raspi-config
```

En el apartado **System Options** > **Hostname** cambiaremos el nombre de equipo de la Raspberry, en nuestro caso las llamaremos `k8s-0X` donde X es el número de nodo de cada Raspberry.

Para asegurarnos de que disponemos de todo el espacio de la tarjeta SD disponible, es necesario acceder al apartado **Advanced Options** y seleccionar la opción **Expand Filesystem**.



### Instalación de K3S

------

Preparamos el entorno para K3S habilitando `iptables-legacy` en todos los nodos que queramos que formen parte del clúster.

```shell
update-alternatives --set iptables /usr/sbin/iptables-legacy 
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy 
```

A continuación, ejecutamos el script que nos instala K3s en nuestra Raspberry. Este comando es necesario ejecutarlo solo en la Raspberry que queramos que actúe como `master` en el clúster.

```shell
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--disable servicelb" sh -s -
```

Nos copiamos el token que nos permitirá añadir más nodos al clúster.

```shell
cat /var/lib/rancher/k3s/server/node-token
```

A continuación, ejecutaremos el siguiente comando en todos los `workers` que queramos agregar al clúster.

```shell
curl -sfL https://get.k3s.io | K3S_TOKEN="<TOKEN>" K3S_URL="https://<MASTER-IP>:6443" K3S_NODE_NAME="<NODE-NAME>" sh - 
```

> NOTA: Posteriormente, podremos convertir un nodo `worker` en `master` y viceversa.

Comprobamos que los nodos se han añadido correctamente al clúster.

```shell
kubectl get nodes
```

> NOTA: Si deseamos visualizar el archivo `kubeconfig` para conectar nuestro clúster con una aplicación externa, como por ejemplo Lens, lo podemos encontrar en `/etc/rancher/k3s/k3s.yaml`



### Instalación de HELM

------

En el nodo `master`, nos descargamos desde el [repositorio de HELM](https://github.com/helm/helm/releases/tag/v3.6.3) en Github la última versión del binario para nuestra arquitectura, en este caso `arm64`.

```shell
wget https://get.helm.sh/helm-v3.6.3-linux-arm64.tar.gz
```

Descomprimimos el archivo que hemos descargado.

```shell
tar -zxvf helm-v3.6.3-linux-arm64.tar.gz
```

Movemos el binario a una carpeta que esté dentro de nuestro `$PATH`.

```shell
chmod +x linux-arm64/helm
mv linux-arm64/helm /usr/local/bin
```

> NOTA: Si al instalar cualquier paquete con HELM nos aparece el siguiente error:
>
> ```shell
> Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version?timeout=32s": dial tcp [::1]:8080: connect: connection refused
> ```
>
> Lo podemos solucionar con el siguiente comando: 
>
> ```shell
> export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
> ```



### Instalación de MetalLB

Añadimos el repositorio HELM de MetalLB:

```shell
helm repo add metallb https://metallb.github.io/metallb
helm repo update
```

A continuación, creamos un archivo llamado `values.yaml` que será donde indiquemos la configuración de MetalLB:

```yaml
configInline:
          address-pools:
          - name: generic-cluster-pool
            protocol: layer2
            addresses:
            - 192.168.110.252-192.168.110.253
```

Por último, instalamos MetalLB indicando el archivo de configuración que acabamos de crear:

```shell
helm install metallb metallb/metallb -f values.yaml --namespace metallb --create-namespace
```



### Acceso al dashboard de Traefik

------

Comprobamos que todos los recursos de Traefik que se han desplegado en la instalación del clúster están en ejecución.

```shell
kubectl get all -n kube-system
```

En el servicio de Traefik de tipo `Load Balancer` es importante fijarnos en la IP que aparece en `EXTERNAL_IP`. Si intentamos acceder a esa IP en un navegador, nos va a aparecer un error 404. 

Para acceder al dashboard de Traefik, tenemos que editar el `IngressRoute` llamado `traefik-dashboard` y añadir el entrypoint `web`:

```yaml
root@k8s-01:~# kubectl edit ingressroute traefik-dashboard -n kube-system

...
spec:
  entryPoints:
    - traefik
    - web
  routes:
...
```

Ahora, si en un navegador accedemos a la URL `http://<EXTERNAL-IP>/dashboard/#/` nos debe aparecer el dashboard de Traefik.



## Almacenamiento

------

### Longhorn

#### Pre-requisitos

Primero, instalamos en todos los nodos los paquetes necesarios para que Longhorn funcione correctamente:

```shell
sudo apt install -y open-iscsi jq
```

#### Despliegue de Longhorn

Desplegamos Longhorn en nuestro clúster con HELM:

 ```shell
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
 ```

Comprobamos que los recursos de Longhorn están funcionando:

```shell
kubectl get pod -n longhorn-system 
```

Por último, creamos un archivo llamado `longhorn-ingress.yaml` que utilizaremos para desplegar un Ingress que nos permitirá acceder a la UI de Longhorn:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
spec:
  rules:
  - host: "longhorn.example.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80

              
root@k8s-02:~# kubectl apply -f longhorn-ingress.yaml
```

Comprobamos que podemos acceder a la UI de Longhorn accediendo desde un navegador a la URL que hemos indicado en el apartado `host` del Ingress. 

> NOTA: Para poder acceder al servicio, es necesario editar el archivo hosts del equipo o añadir una entrada a nuestro servidor DNS.

#### Almacenamiento

Para utilizar el almacenamiento de Longhorn, tan solo desplegamos un POD junto con un PVC que se vinculará a un StorageClass que ha creado Longhorn durante la instalación:

```fallback
kubectl create -f https://raw.githubusercontent.com/longhorn/longhorn/v1.1.2/examples/pod_with_pvc.yaml
```

A continuación, en el dashboard podremos ver que se ha creado un nuevo volúmen.



### OpenEBS - CStor

------

#### Pre-requisitos

Primero, es necesario instalar el paquete `open-iscsi` en todos los nodos para que OpenEBS funcione correctamente:

```shell
sudo apt-get update
sudo apt-get install open-iscsi
sudo systemctl enable --now iscsid
```

#### Despliegue de OpenEBS

Instalamos OpenEBS utilizando HELM:

```
root@k8s-01:~# helm repo add openebs https://openebs.github.io/charts
root@k8s-01:~# helm repo update
root@k8s-01:~# helm install openebs openebs/openebs --set cstor.enabled=true --namespace openebs --create-namespace
```

Comprobamos que los PODs está funcionando correctamente y que los nodos tienen discos conectados:

```shell
kubectl get pods -n openebs -w
kubectl get bd -n openebs
```

A continuación, creamos un nuevo recurso de tipo `CStorPoolCluster` e indicamos los nodos que queramos que aporten almacenamiento y los `blockdevice` que tiene conectados cada uno.

```yaml
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-b6242f6a804cf27d0232c79e36fe6054"
           - blockDeviceName: "blockdevice-aac14e66b1b78b818df0e55c5055c187"
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
     poolConfig:
       dataRaidGroupType: "stripe"
       
       
root@k8s-01:~# kubectl apply -f cstorpool.yaml
```

Verificamos que el `CStorPoolCluster` se ha creado correctamente y que la instancia de cada nodo está `ONLINE`:

```shell
kubectl get cspc -n openebs
kubectl get cspi -n openebs
```

Ahora, crearemos un `StorageClass` en el que indicaremos el `CStorPoolCluster` que acabamos de crear y un número de replicas que tiene que ser menor o igual al número de `CSPI` que disponemos. Es importante que al menos **(n/2 + 1)** réplicas estén habilitadas y funcionando, donde **n** es el número total de réplicas que tenemos. Es decir, si establecemos `replicaCount` en 6, es necesario que al menos 4 estén funcionando. 

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cstor-csi-disk
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
parameters:
  cas-type: cstor
  # cstorPoolCluster should have the name of the CSPC 
  cstorPoolCluster: cstor-disk-pool
  # replicaCount should be <= no. of CSPI created in the selected CSPC
  replicaCount: "2"
```

Para comprobar que se ha creado correctamente, ejecutamos el siguiente comando:

```shell
kubectl get sc
```

Para que un POD pueda utilizar el almacenamiento del `StorageClass` que acabamos de crear, es necesario crear un `PersistentVolumeClaim`: 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cstor-pvc
spec:
  storageClassName: cstor-csi-disk
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### Almacenamiento

A continuación, desplegamos un POD de prueba:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - command:
       - sh
       - -c
       - 'date >> /mnt/openebs-csi/date.txt; hostname >> /mnt/openebs-csi/hostname.txt; sync; sleep 5; sync; tail -f /dev/null;'
    image: busybox
    imagePullPolicy: Always
    name: busybox
    volumeMounts:
    - mountPath: /mnt/openebs-csi
      name: demo-vol
  volumes:
  - name: demo-vol
    persistentVolumeClaim:
      claimName: cstor-pvc
```

El POD desplegado escribe la fecha actual en el directorio que hemos montado `/mnt/openebs-csi/date.txt` cada vez que se inicia:

```shell
kubectl exec -it busybox -- cat /mnt/openebs-csi/date.txt
```



### Rook

------

#### Pre-requisitos

Antes de instalar Rook, tenemos que asegurarnos de que disponemos de un disco o una partición de un disco sin formatear en los nodos `worker`.

Podemos comprobar si nuestro disco o partición están formateados con el siguiente comando:

```shell
lsblk -f
```

> ```shell
> NAME                FSTYPE     LABEL  UUID                                   MOUNTPOINT
> sda
> └─sda1              LVM2_member       >eSO50t-GkUV-YKTH-WsGq-hNJY-eKNf-3i07IB
> ├─ubuntu--vg-root   ext4              c2366f76-6e21-4f10-a8f3-6776212e2fe4   /
> └─ubuntu--vg-swap_1 swap              9492a3dc-ad75-47cd-9596-678e8cf17ff9   [SWAP]
> sdb
> ```

#### Despliegue de Rook

Primero, clonamos el repositorio de Rook en nuestro equipo y accedemos a la carpeta `.../kubernetes/ceph`.

```shell
git clone https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
```

Creamos los recursos desplegando los siguientes archivos YAML.

```shell
kubectl create -f crds.yaml 
kubectl create -f common.yaml 
kubectl create -f operator.yaml
```

Deberíamos ver el POD `Operator` funcionando.

```shell
kubectl get pods -n rook-ceph
```

Ahora, desplegaremos el archivo `cluster.yaml` que detectará nuestros volúmenes sin formatear y Ceph creará un clúster con ellos.

```shell
kubectl create -f cluster.yaml
```

#### Comprobar salud del clúster

Una vez tengamos todos los servicios en funcionamiento, podemos verificar la salud del clúster desplegando el archivo `toolbox.yaml`.

```shell
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f toolbox.yaml
```

Cuando el POD `rook-ceph-tools` esté en ejecución, accedemos a él.

```shell
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Dentro del POD, podemos ejecutar los siguientes comandos para comprobar el estado de los discos.

- `ceph status`
- `ceph osd status`
- `ceph df`
- `rados df`

Si nos aparece `HEALTH_OK` está todo bien.

> [root@rook-ceph-tools-77f4f75cdd-n2vr4 /]# ceph status
> cluster:
>  id:     f42f5ba1-0591-4bcc-9944-339855562500
>  health: HEALTH_OK
>
> services:
>  mon: 3 daemons, quorum a,b,c (age 26m)
>  mgr: a(active, since 15m)
>  osd: 3 osds: 3 up (since 23m), 3 in (since 23m)

#### Almacenamiento

Para poder utilizar el almacenamiento del clúster de Ceph, accedemos a la carpeta `../ceph/csi/rbd` y desplegamos el archivo `storageclass.yaml`

````shell
cd rook/cluster/examples/kubernetes/ceph/csi/rbd
kubectl apply -f storageclass.yaml
````

A continuación, tenemos que crear un `PersistentVolumeClaim` para que los PODs puedan utilizar el disco.



## Monitorización

------

### Prometheus, Alertmanager y Grafana

Clonamos el repositorio `kube-prometheus` del Github de `prometheus-operator`:

```shell
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus
```

Desplegamos todos los recursos de las carpetas `manifests` y `setup`

```shell
kubectl create -f manifests/setup
kubectl create -f manifests/
```

A continuación, creamos un archivo llamado `prometheus-ingress.yaml` que utilizaremos para desplegar un Ingress y así poder acceder a los servicios.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  rules:
  - host: "alertmanager.example.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: alertmanager-main
            port:
              name: web

  - host: "grafana.example.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: grafana
            port:
              name: http

  - host: "prometheus.example.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: prometheus-k8s
            port:
              name: web
```

Por último, en un navegador accedemos a las URL que hemos indicado en el apartado `host` de nuestro ingress. 

> NOTA: Para poder acceder a los servicios, es necesario editar el archivo hosts del equipo o añadir esas entradas a nuestro servidor DNS.

#### Monitorización de Longhorn

Para monitorizar nuestro Longhorn, tan solo debemos desplegar el siguiente `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn
  namespace: monitoring
  labels:
    name: longhorn
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - longhorn-system
  endpoints:
  - port: manager
```



### Thanos

Thanos es una herramienta que nos permite exportar todas las métricas que recoge Prometheus para almacenarlas en un servicio cloud como AWS, Azure o GCP.



## Service Mesh

### Istio

Comenzamos descargándonos la herramienta `istioctl`

```shell
curl -L https://istio.io/downloadIstio | sh -
```

Movemos el binario a una ruta dentro de nuestro $PATH

```shell
cd istio-1.11.1/bin
mv istioctl /usr/local/bin
```

A continuación, instalamos Istio:

```shell
istioctl install
```

Añadimos una etiqueta al namespace que queramos para que Istio despliegue su `sidecar` junto con los PODs para poder monitorizar el tráfico entre ellos. 

```shell
kubectl label namespace default istio-injection=enabled
```

Desplegamos un par de PODs de prueba:

```shell
kubectl apply -f https://raw.githubusercontent.com/pablokbs/peladonerd/master/kubernetes/30/01-hello-app-v1.yaml
kubectl apply -f https://raw.githubusercontent.com/pablokbs/peladonerd/master/kubernetes/30/02-hello-app-v2.yaml
```

Al comprobar el estado de los PODs, podemos ver que nos aparece `2/2`, eso significa que hay 2 contenedores en ejecución, la aplicación que hemos desplegado y el proxy de Istio:

```shell
kubectl get pods
```

Para poder acceder al dashboard de Istio tenemos que desplegar más aplicaciones. Primero, clonamos el repositorio de Istio:

```shell
git clone https://github.com/istio/istio.git
```

Ahora, desplegamos los siguientes archivos YAML:

```shell
kubectl apply -f istio/samples/addons/
```

Iniciamos el dashboard de Kiali:

```shell
istioctl dashboard kiali
```

A continuación, accedemos a la terminal del POD `hello-v1`:

```shell
kubectl exec -it <HELLO-V1_POD_NAME> -- sh
```

Instalamos la herramienta `curl`:

```shell
apk add --update curl
```

Simulamos peticiones HTTP al POD `hello-v2` con el siguiente bucle:

```shell
while sleep 1; do curl -o /dev/null -s -w %{http_code} http://hello-v2-svc:80/; done
```



## Integración continua

### ArgoCD

Comenzamos creando un nuevo namespace para Argo:

```shell
kubectl create namespace argocd
```

Desplegamos todos los recursos de Argo:

```shell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Editamos el deployment llamado `argocd-server` y, en el apartado `commands` añadimos el flag                       `--insecure`:

```shell
kubectl edit deployment argocd-server -n argocd

...
containers:                                         
      - command:                                          
        - argocd-server                                   
        - --insecure                                      
        env: 
...
```

Creamos un ingress para poder acceder al dashboard de Argo:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
spec:
  rules:
  - host: "argocd.example.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

Una vez hemos accedido al dashboard de Argo, el nombre de usuario por defecto es `admin` y para saber la contraseña tendremos que ejecutar el siguiente comando:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```



## Rancher

------

Primero, crearemos un nuevo namespace llamado `cattle-system` que será donde se desplieguen todos los recursos de Rancher.

```shell
kubectl create namespace cattle-system
```

Con HELM, añadimos el repositorio de Rancher.

```shell
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Crear certificado autofirmado con SSL

Creamos una nueva carpeta que será donde guardemos todos los archivos.

```
mkdir openssl && cd openssl
```

Generamos una clave privada para la CA

```shell
openssl genrsa -out ca.key 2048
```

Generamos un certificado x509 utilizando la clave privada de la CA. Indicamos nuestro país, nombre de organización, departamento, etc.

```shell
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1825 -out ca.crt
```

Ahora, generamos la clave privada del servidor.

```shell
openssl genrsa -out tls.key 2048
```

Creamos un archivo de configuración llamado `csr.conf` para generar una `Certificate Signing Request`

```json
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = ES
ST = Spain
L = Logrono
O = Bosonit
OU = DevOps
CN = rancher.example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = 192.168.10.4
DNS.2 = 192.168.10.5
IP.1 = 192.168.10.4
IP.2 = 192.168.10.5
```

Generamos el CSR utilizando la clave privada del servidor y el archivo de configuración.

```shell
openssl req -new -key tls.key -out server.csr -config csr.conf
```

Por último, generamos el certificado utilizando la clave y el certificado de la CA junto con el archivo CSR que acabamos de crear.

```shell
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -out tls.crt -days 10000 \
-extfile csr.conf
```



------

A continuación, creamos un nuevo secreto que va a contener una clave y un certificado que hemos generado previamente con `openssl`.

```shell
kubectl -n cattle-system create secret tls tls-rancher-ingress \
  --cert=tls.crt \
  --key=tls.key
```

Instalamos Rancher con el siguiente comando:

```shell
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set ingress.tls.source=secret
```

Editamos el `deployment` de rancher y cambiamos las réplicas de 3 a 1 .

```shell
kubectl edit deployment rancher -n cattle-system
```

Esperamos a que Rancher se inicie, puede tardar varios minutos. Accedemos a la interfaz de Rancher escribiendo en un navegador el hostname que hemos indicado cuando hemos instalado Rancher con HELM. 



## Copias de seguridad

### MinIO

Para poder almacenar nuestras copias de seguridad, instalaremos Minio que es un servidor privado compatible con el servicio de almacenamiento cloud Amazon S3. 

Comenzaremos descargando el binario para nuestra arquitectura y moviendo el archivo a la carpeta `/usr/local/bin`

```shell
wget https://dl.minio.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

A continuación, prepararemos un disco para que Minio lo utilice como almacenamiento.

```shell
sudo parted -s -a optimal -- /dev/sdb mklabel gpt
sudo parted -s -a optimal -- /dev/sdb mkpart primary 0% 100%
sudo parted -s -- /dev/sdb align-check optimal 1
sudo mkfs.ext4 /dev/sdb1
echo "/dev/sdb1 /data ext4 defaults 0 0" | sudo tee -a /etc/fstab
sudo mkdir /data
sudo mount -a
```

Comprobamos que el disco se ha montado correctamente.

```shell
$ df -h | grep /data
/dev/sdb1       9.8G   37M  9.3G   1% /data
```

Creamos un usuario y un grupo para el servicio de Minio:

```shell
sudo groupadd --system minio
sudo useradd -s /sbin/nologin --system -g minio minio
```

Y asignamos al usuario Minio como propietario de la carpeta `/data`:

```shell
chown -R minio:minio /data/
```

Ahora, creamos un archivo de configuración para el servicio de Minio:

```shell
sudo nano /etc/systemd/system/minio.service
```

Pegamos el siguiente texto en el archivo de configuración del servicio de Minio:

```shell
[Unit]
Description=Minio
Documentation=https://docs.minio.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/data
User=minio
Group=minio

EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"

ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
```

A continuación, creamos un archivo de variables de entorno donde podremos configurar el puerto de Minio, el volumen que utiliza como almacenamiento y el usuario de acceso entre otras opciones:

```shell
sudo mkdir -p /etc/default/

cat <<EOF | sudo tee /etc/default/minio
# Volume to be used for Minio server.
MINIO_VOLUMES="/data"
# Use if you want to run Minio on a custom port.
MINIO_OPTS="--address :9000"
# Access user of the server.
MINIO_ROOT_USER=minio
# Secret password of the server.
MINIO_ROOT_PASSWORD=miniostorage
EOF
```

#### Instalación de MinIO en Kubernetes

Para empezar, añadimos el repositorio HELM de MinIO y lo instalamos:

```shell
helm repo add minio https://operator.min.io/
helm repo update
helm install --namespace minio-operator --create-namespace --generate-name minio/minio-operator
```

A continuación, desplegamos el siguiente `ingress` para poder acceder al servicio de MinIO:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-ingress
  namespace: minio-operator
spec:
  rules:
  - host: "minio.example.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: operator
            port:
              name: http
```

Una vez hemos accedido al servicio de MinIO, tendremos que crear un nuevo `tenant` para poder comenzar a utilizar el almacenamiento.



### Velero

Empezaremos descargando la última versión del binario de Velero desde el [repositorio](https://github.com/vmware-tanzu/velero/releases) de Github para nuestra arquitectura, en este caso `arm64`

```shell
wget https://github.com/vmware-tanzu/velero/releases/download/v1.6.3/velero-v1.6.3-linux-arm64.tar.gz
```

Extraemos el archivo `.tar.gz` que acabamos de descargar:

```shell
tar -xvf velero-v1.6.3-linux-arm64.tar.gz
```

Movemos el archivo  `velero` a una carpeta que esté dentro de nuestro $PATH:

```shell
cd velero-v1.6.3-linux-arm64
mv velero /usr/local/bin
```

Creamos el archivo `credentials-velero` en nuestro directorio local con el usuario y contraseña de nuestro Minio:

```shell
[default]
aws_access_key_id=minio
aws_secret_access_key=miniostorage
```

A continuación, instalamos Velero:

```shell
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.1 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=true \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://<MINIO-IP>:9000
```

Añadimos el plugin de OpenEBS para poder realizar una copia de los datos de los PODs:

```shell
velero plugin add openebs/velero-plugin:2.12.0
```

Por último, creamos un `VolumeSnapshotLocation` donde indicaremos los datos de acceso a nuestro Minio:

```yaml
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: aws
  namespace: velero
spec:
  provider: openebs.io/cstor-blockstore
  config:
    bucket: velero
    provider: aws
    region: minio
    s3ForcePathStyle: "true"
    s3Url: http://<ipaddress:port>
```

> NOTA: Podemos eliminar el `VolumeSnapshotLocation` llamado `default`
>
> ```shell
> kubectl delete VolumeSnapshotLocation default -n velero
> ```



### Crear una copia de seguridad

Para crear una copia de seguridad de un namespace, tan solo ejecutamos el siguiente comando:

```shell
velero backup create <BACKUP_NAME> --include-namespaces=<NAMESPACE> --snapshot-volumes --volume-snapshot-locations=<SNAPSHOT_LOCATION>
```

Comprobamos que se ha completado correctamente:

```shell
velero backup describe <BACKUP_NAME>
```

Vemos todas las copias de seguridad que tenemos:

```shell
velero backup get
```

También es posible programar nuestras copias de seguridad utilizando expresiones de `cron` para que se ejecuten automáticamente cuando queramos:

```shell
velero schedule create <SCHEDULE_NAME> --schedule="0 1 * * *" --selector app=nginx

velero schedule create <SCHEDULE_NAME> --schedule="@daily" --selector app=nginx
```



### Restaurar una copia de seguridad

Para restaurar la copia de seguridad que hemos hecho, ejecutaremos el siguiente comando:

```shell
velero restore create --from-backup <BACKUP_NAME> --restore-volumes=true
```

Podemos restaurar la copia de seguridad en un namespace distinto al original:

```shell
restore create --from-backup backup_name --restore-volumes=true --namespace-mappings <SOURCE_NS>:<DESTINATION_NS>
```

De nuevo comprobamos que la restauración de la copia de seguridad se ha completado correctamente:

```shell
velero restore describe <RESTORE_NAME>
```

Vemos todas restauraciones que hemos hecho:

```shell
velero restore get
```


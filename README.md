# Kubernetes Lab — k8k-lab

## 1. Crear un clúster Kind de 3 nodos

Este laboratorio utiliza Kind para crear un clúster Kubernetes local
con:

- 1 nodo `control-plane`
- 2 nodos `worker`

### Configuración

Archivo:

`cluster.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
    - role: control-plane
    - role: worker
    - role: worker
```

```bash
kind create cluster \
  --name k8k-lab \
  --config ~/projects-nvim/kubernetes/k8k-lab/cluster.yaml
```

```bash
kubectl get nodes
```

```text
NAME                   STATUS   ROLES           ...
k8k-lab-control-plane  Ready    control-plane   ...
k8k-lab-worker         Ready    <none>          ...
k8k-lab-worker2        Ready    <none>          ...

```

## 2. Incidencia durante la creación del clúster

Durante el primer intento de creación del clúster, los workers no
pudieron completar el proceso de unión al clúster.

El error principal encontrado en `kubelet.log` fue:

```text
too many open files
inotify_init: too many open files
```

También se produjo:

```text
The kubelet is not healthy
```

### Diagnóstico

Se comprobaron los límites actuales de `inotify`:

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
```

Valores iniciales:

```text
fs.inotify.max_user_watches = 26711
fs.inotify.max_user_instances = 128
```

### Solución

Se aumentaron temporalmente los límites:

```bash
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
```

Se verificaron los nuevos valores:

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
```

Resultado:

```text
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

Después se eliminó el clúster fallido:

```bash
kind delete cluster --name k8k-lab
```

Y se volvió a crear utilizando la misma configuración:

```bash
kind create cluster \
  --name k8k-lab \
  --config ~/projects-nvim/kubernetes/k8k-lab/cluster.yaml
```

### Verificación final

El clúster se creó correctamente y los tres nodos quedaron en estado `Ready`:

```bash
kubectl get nodes
```

Resultado:

```text
NAME                   STATUS   ROLES           VERSION
k8k-lab-control-plane  Ready    control-plane   v1.34.0
k8k-lab-worker         Ready    <none>          v1.34.0
k8k-lab-worker2        Ready    <none>          v1.34.0
```

### Concepto aprendido

Kind utiliza contenedores para ejecutar los nodos de Kubernetes.
El `control-plane` gestiona el clúster y los `worker` ejecutan las
cargas de trabajo.

Los límites de `inotify` del sistema operativo pueden afectar al
arranque de `kubelet`. Cuando estos límites son insuficientes, pueden
aparecer errores como `too many open files`.

## 3. Namespace

Los recursos del laboratorio se organizarán dentro del namespace
`devops-lab`.

### Crear el namespace

```bash
kubectl create namespace devops-lab
```

### Comprobamos que namespace hay disponibles

```bash
kubectl get namespaces
```

```text
NAME                 STATUS   AGE
default              Active   43m
devops-lab           Active   6s
kube-node-lease      Active   43m
kube-public          Active   43m
kube-system          Active   43m
local-path-storage   Active   43m
```

\*\*Un namespace permite separar y organizar recursos dentro del mismo clúster Kubernetes...

### Siguiente paso: Pod

Ahora vamos a crear **nuestro primer Pod manualmente**, sin Deployment.

La idea es entender primero qué es realmente un Pod antes de introducir `Deployment` y `ReplicaSet`.

Primero comprueba que estamos utilizando el namespace correcto:

```bash
kubectl config current-context
```

```text
kind-k8k-lab
```

### Manifiesto

Archivo:

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod

metadata:
    name: nginx-pod
    namespace: devops-lab

spec:
    containers:
        - name: nginx
          image: nginx:alpine
          ports:
              - containerPort: 80
```

Crear el Pod

```bash
kubectl apply -f pod.yaml
```

Resultado:

pod/nginx-pod created

Comprobar los Pods del namespace

```bash
kubectl get pods -n devops-lab
```

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5m41s
```

Incidencia con la imagen

En el primer intento se utilizó una etiqueta de imagen incorrecta:

```yaml
nginx:alpaine
```

Kubernetes mostró:

```bash
ErrImagePull
ImagePullBackOff
```

Con:

```bash
kubectl describe pod nginx-pod -n devops-lab
```

se pudo comprobar en la sección Events que la imagen no existía:

```bash
Failed to pull image "nginx:alpaine"
docker.io/library/nginx:alpaine: not found
```

Se corrigió el manifiesto a:

```yaml
image: nginx:alpine
```

Después se volvió a aplicar:

```bash
kubectl apply -f pod.yaml
```

Verificación final

```bash
kubectl get pods -n devops-lab
```

Resultado:

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5m41s
```

### Conceptos aprendidos

`kubectl apply -f pod.yaml` crea o actualiza el recurso definido en el manifiesto.

El namespace del Pod está definido en `metadata.namespace`.

`kubectl get pods` consulta por defecto el namespace configurado en el contexto.

`kubectl get pods -n devops-lab` consulta específicamente el namespace devops-lab.

`kubectl describe pod` permite inspeccionar el Pod y sus eventos.

`ErrImagePull` indica que Kubernetes no pudo descargar la imagen.

`ImagePullBackOff` indica que Kubernetes está reintentando la descarga después de un fallo.

## 4. Deployment

Después de crear y probar un Pod manualmente, el siguiente paso es
utilizar un `Deployment`.

Un Deployment permite declarar el estado deseado de una aplicación.
En este caso queremos mantener **3 réplicas de NGINX** ejecutándose

dentro del namespace `devops-lab`.

### Manifiesto

Archivo:

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
    name: nginx-deployment
    namespace: devops-lab

spec:
    replicas: 3

    selector:
        matchLabels:
            app: nginx

    template:
        metadata:
            labels:
                app: nginx

        spec:
            containers:
                - name: nginx
                  image: nginx:alpine
                  ports:
                      - containerPort: 80
```

### Crear el Deployment

```bash
kubectl apply -f deployment.yaml
```

- Resultado
  deployment.apps/nginx-deployment created

### Comprobar el Deployment

```bash
kubectl get deployment -n devops-lab
```

- Resulto

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           ...
```

El Deployment mantiene 3 réplicas disponibles.

### Comprobar los pods

- Con este comando comprobamos el estado de los pods (running, name, ready etc..)
  ![comprobacion de Pods](/images/image.png)

- Este comando nos entrega una informacion mas detallada (en que worker esta cada pod...)

![Comprobacion de Pods detallada](/images/image-1.png)

Nota. El deployment crea y gestiona un replicaSet, y este mantiene el numero de Pods definido mediante spec.replicas.

### Prueba de autoreparación

- Para comprobar el comportamiento del Deployment, eliminaremos manualmente uno de los Pods:

```bash
kubectl delete pod nginx-deployment-54fc99c8d-twqnz -n devops-lab
```

![Eliminamod un Pod manualmente](/images/image-2.png)

- Al comprobar nuevamente los Pods

```bash
kubectl get pods -n devops-lab
```

![Comprobamos los Pods](/images/image-3.png)

Kubernetes crea automáticamente un nuevo Pod para volver alcanzar las 3 replicas deseadas.

### Concepto importante

El Pod eliminado no se recupera.
Kubernetes crea un nuevo Pod porque el ReplicaSet detecta que el
estado actual no coincide con el estado deseado.
Estado deseado:

```text
3 Pods

Estado actual después de eliminar uno:

2 Pods

        ↓

ReplicaSet detecta la diferencia

        ↓

Crea un nuevo Pod

        ↓

3 Pods nuevamente
```

Este mecanismo es una de las características fundamentales de
Kubernetes: mantener continuamente el estado real del clúster
alineado con el estado declarado en los manifiestos.

### Conceptos aprendidos

- Un Deployment define el estado deseado de una aplicación.
- spec.replicas determina cuántas réplicas queremos mantener.
- El Deployment crea y gestiona un ReplicaSet.
- El ReplicaSet mantiene el número deseado de Pods.
- Los Pods gestionados por un Deployment pueden ser reemplazados
  automáticamente.
- El Pod reemplazado es un recurso nuevo, no una recuperación del
  Pod anterior.

## 5. Service

Después de crear un `Deployment` con tres réplicas de NGINX, el siguiente paso es exponer esas réplicas mediante un `Service`.

Un `Service` proporciona un punto de acceso estable para un conjunto de Pods. En este laboratorio utilizamos el tipo `ClusterIP`, que permite acceder a la aplicación desde dentro del clúster.

### Configuración del Service

El Service utilizado en el laboratorio tiene estas características:

- **Nombre:** `nginx-service`
- **Namespace:** `devops-lab`
- **Tipo:** `ClusterIP`
- **IP interna:** `10.96.201.113`
- **Puerto del Service:** `80/TCP`
- **Puerto de destino de los Pods:** `80/TCP`
- **Selector:** `app: nginx`

![service.yaml](/images/image-4.png)

El selector `app: nginx` coincide con las etiquetas de los Pods creados por `nginx-deployment`. De esta manera, el Service puede dirigir las peticiones a los Pods que cumplen ese selector.

### Comprobar el Service

Para consultar el Service dentro del namespace del laboratorio:

```bash
kubectl get service nginx-service -n devops-lab
```

La dirección `10.96.201.113` es una IP virtual interna del clúster. No es una dirección pública ni implica que el Service sea accesible directamente desde el equipo anfitrión.

### Comprobar los endpoints

Para comprobar qué Pods están asociados al Service:

```bash
kubectl get endpoints nginx-service -n devops-lab
```

![comprobacion de los endpoints](/images/image-5.png)

Durante el laboratorio, el Service tenía tres endpoints de NGINX en el puerto `80`, correspondientes a las réplicas del Deployment.

Los endpoints permiten verificar que el selector del Service encuentra Pods disponibles. Si no aparecen endpoints, conviene revisar las etiquetas de los Pods, el selector del Service y el estado de las réplicas.

### Prueba de resolución DNS

Kubernetes permite localizar un Service mediante DNS interno. Para comprobarlo se ejecutó un Pod temporal de BusyBox y se consultó el nombre DNS completo del Service:

```bash
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.36 --context kind-k8k-lab -- nslookup nginx-service.devops-lab.svc.cluster.local
```

Resultado obtenido:

```text
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:      nginx-service.devops-lab.svc.cluster.local
Address:   10.96.201.113
```

![comprobamos mediante DNS](/images/image-6.png)

La resolución fue correcta: CoreDNS devolvió la IP `10.96.201.113` del Service.

El mensaje final indicando que `dns-test` fue eliminado es esperado, porque se utilizó la opción `--rm` para borrar el Pod temporal al terminar la prueba.

### Prueba de conectividad HTTP

Después se realizó una petición HTTP al Service desde otro Pod temporal:

```bash
kubectl run http-test --rm -it --restart=Never --image=curlimages/curl --context kind-k8k-lab -- curl -sS -o /dev/null -w 'HTTP %{http_code}\\n' http://nginx-service.devops-lab.svc.cluster.local
```

Resultado obtenido:

```text
HTTP 200
pod "http-test" deleted from default namespace
```

![Comprobamos por HTTP](/images/image-7.png)

La respuesta `HTTP 200` confirma que la petición llegó correctamente a NGINX a través del Service. El Pod temporal se eliminó automáticamente al finalizar la prueba.

### Incidencia durante las pruebas: `too many open files`

Durante las comprobaciones del laboratorio, algunos Pods de `kube-proxy` entraron en `CrashLoopBackOff` y CoreDNS no estaba preparado. El registro de `kube-proxy` mostraba:

```text
command failed
err="failed complete: too many open files"
```

Se consultó la documentación oficial de Kind, que identifica los límites de `inotify` como una posible causa de este tipo de error. En AlmaLinux, los valores iniciales eran:

```text
fs.inotify.max_user_watches = 26711
fs.inotify.max_user_instances = 128
```

Se aumentaron los valores en ejecución y se guardaron de forma persistente en `/etc/sysctl.conf`:

```text
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

Se reinició AlmaLinux y se verificó que los valores persistieran:

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
```

Resultado:

```text
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

Después del reinicio, también se comprobaron esos valores dentro del nodo de control de Kind. Los Pods de `kube-proxy` y CoreDNS pasaron a `Running` y `Ready`, la resolución DNS volvió a funcionar y la petición HTTP al Service devolvió `200`.

La secuencia respalda que los límites de `inotify` estaban relacionados con la incidencia, aunque no demuestra de forma aislada que fueran la única causa.

### Conceptos aprendidos

- Un `Service` proporciona un punto de acceso estable a un conjunto de Pods.
- `ClusterIP` expone el Service únicamente dentro de la red del clúster.
- El selector `app: nginx` vincula el Service con los Pods que tienen esa etiqueta.
- `port` define el puerto del Service y `targetPort` el puerto del contenedor al que se dirige el tráfico.
- Los endpoints permiten comprobar qué Pods están asociados al Service.
- CoreDNS permite resolver el nombre DNS del Service, por ejemplo `nginx-service.devops-lab.svc.cluster.local`.
- La prueba DNS confirmó que el nombre resolvía a `10.96.201.113`.
- La prueba HTTP devolvió `200`, confirmando el acceso a NGINX a través del Service.
- Los límites del sistema operativo pueden afectar al funcionamiento de los componentes de Kubernetes en un laboratorio local; conviene consultar la documentación oficial y validar la recuperación con pruebas.

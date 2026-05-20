# Conectar un Cluster Consumidor a un Cluster Proveedor OpenShift Data Foundation 4.x

## Arquitectura del escenario

```
┌─────────────────────────────────────┐        ┌─────────────────────────────────────┐
│        CLUSTER A (Proveedor)        │        │       CLUSTER B (Consumidor)        │
│                                     │        │                                     │
│  namespace: openshift-storage       │◄──────►│  namespace: openshift-storage       │
│  ODF Operator instalado             │  gRPC  │  ODF Operator instalado             │
│  StorageCluster (modo proveedor)    │  50051 │  StorageCluster (modo consumidor)   │
│  Ceph cluster activo                │        │  Sin discos locales                 │
└─────────────────────────────────────┘        └─────────────────────────────────────┘
```

En este modelo:
- El **Cluster A (Proveedor)** tiene ODF con un cluster Ceph completo y expone un endpoint gRPC.
- El **Cluster B (Consumidor)** instala ODF en modo consumidor, se registra contra el proveedor y obtiene StorageClasses para crear PVCs/PVs.

---

## Prerequisitos

- [ ] Acceso de `cluster-admin` a ambos clusters
- [ ] Conectividad de red entre ambos clusters (se requiere el puerto gRPC del proveedor, generalmente `50051` o via Route en `443`)
- [ ] ODF Operator instalado en ambos clusters (misma versión o compatible)
- [ ] El namespace `openshift-storage` existe en ambos clusters
- [ ] CLI `oc` configurada con acceso a ambos clusters (usar contextos o dos terminales separadas)

---

## PARTE 1: Configuracion del Cluster Proveedor (Cluster A)

> Todos los comandos de esta sección se ejecutan con contexto apuntando al **Cluster A**.

### Paso 1.1 — Verificar que el StorageCluster esta activo

```bash
oc get storagecluster -n openshift-storage
```

La salida esperada debe mostrar `Ready` en la columna `PHASE`:

```
NAME                 AGE   PHASE   EXTERNAL   CREATED AT
ocs-storagecluster   10d   Ready              2025-01-01T00:00:00Z
```

Verificar también que Ceph esté sano:

```bash
oc get cephcluster -n openshift-storage
```

`HEALTH` debe ser `HEALTH_OK`.

---

### Paso 1.2 — Habilitar el modo proveedor en el StorageCluster

El StorageCluster debe tener habilitada la capacidad de aceptar consumidores remotos. Editar el objeto:

```bash
oc edit storagecluster ocs-storagecluster -n openshift-storage
```

Agregar o verificar que exista el campo `allowRemoteStorageConsumers: true` bajo `spec`:

```yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  allowRemoteStorageConsumers: true
  # ... resto de la configuracion existente
```

Guardar y esperar a que el operador reconcilie:

```bash
oc get storagecluster ocs-storagecluster -n openshift-storage -o jsonpath='{.status.phase}'
```

Debe volver a `Ready`.

---

### Paso 1.3 — Verificar el servicio del proveedor

El operador crea automáticamente el servicio gRPC del proveedor al habilitar `allowRemoteStorageConsumers`:

```bash
oc get service ocs-provider-server -n openshift-storage
```

Salida esperada:

```
NAME                  TYPE        CLUSTER-IP      PORT(S)     AGE
ocs-provider-server   ClusterIP   172.30.x.x      50051/TCP   5m
```

---

### Paso 1.4 — Exponer el endpoint del proveedor fuera del cluster

El servicio `ocs-provider-server` usa gRPC sobre TLS. Para que el Cluster B pueda alcanzarlo, se debe exponer via una Route con terminacion `passthrough` (no termina el TLS en el router, lo pasa directo al pod).

Crear el archivo `odf-provider-route.yaml`:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ocs-provider-server
  namespace: openshift-storage
spec:
  to:
    kind: Service
    name: ocs-provider-server
    weight: 100
  port:
    targetPort: 50051
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: None
  wildcardPolicy: None
```

Aplicar:

```bash
oc apply -f odf-provider-route.yaml
```

Obtener el hostname del endpoint (guardarlo, lo usaremos en el Cluster B):

```bash
oc get route ocs-provider-server -n openshift-storage -o jsonpath='{.spec.host}'
```

Ejemplo de salida:

```
ocs-provider-server-openshift-storage.apps.cluster-a.example.com
```

El endpoint completo a usar sera: `ocs-provider-server-openshift-storage.apps.cluster-a.example.com:443`

> **Nota:** Las Routes de OpenShift escuchan en el puerto `443` para TLS passthrough, aunque el servicio interno use `50051`.

---

### Paso 1.5 — Obtener el endpoint desde el status del StorageCluster

Tras crear la Route, el operador puede actualizar el campo `storageProviderEndpoint` en el status. Verificarlo:

```bash
oc get storagecluster ocs-storagecluster -n openshift-storage \
  -o jsonpath='{.status.storageProviderEndpoint}'
```

Si el campo esta vacio, usar el hostname de la Route obtenido en el paso anterior con formato `<hostname>:443`.

---

### Paso 1.6 — Crear el objeto StorageConsumer y generar el ticket de onboarding

El ticket de onboarding es un token firmado que autoriza al Cluster B a conectarse. Se crea un `StorageConsumer` en el proveedor:

```yaml
apiVersion: ocs.openshift.io/v1alpha1
kind: StorageConsumer
metadata:
  name: cluster-b-consumer
  namespace: openshift-storage
spec:
  capacity: 1Ti
```

Aplicar:

```bash
oc apply -f - <<EOF
apiVersion: ocs.openshift.io/v1alpha1
kind: StorageConsumer
metadata:
  name: cluster-b-consumer
  namespace: openshift-storage
spec:
  capacity: 1Ti
EOF
```

Esperar a que el `StorageConsumer` genere el ticket (puede tardar unos segundos):

```bash
oc get storageconsumer cluster-b-consumer -n openshift-storage -o yaml
```

Extraer el ticket de onboarding (guardarlo, se necesita en el Cluster B):

```bash
oc get storageconsumer cluster-b-consumer -n openshift-storage \
  -o jsonpath='{.status.onboardingTicket}'
```

La salida es un string en base64. Guardarlo completo.

---

### Resumen de datos a llevar al Cluster B

| Dato | Comando para obtenerlo |
|------|------------------------|
| Endpoint del proveedor | `oc get route ocs-provider-server -n openshift-storage -o jsonpath='{.spec.host}'` → agregar `:443` |
| Ticket de onboarding | `oc get storageconsumer cluster-b-consumer -n openshift-storage -o jsonpath='{.status.onboardingTicket}'` |

---

## PARTE 2: Configuracion del Cluster Consumidor (Cluster B)

> Todos los comandos de esta sección se ejecutan con contexto apuntando al **Cluster B**.

### Paso 2.1 — Instalar el ODF Operator en el Cluster B

Si el operador no esta instalado, instalarlo via OperatorHub:

```bash
# Crear el namespace
oc create namespace openshift-storage

# Crear el OperatorGroup
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
    - openshift-storage
EOF

# Crear la suscripcion al operador
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: odf-operator
  namespace: openshift-storage
spec:
  channel: stable-4.20
  name: odf-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Verificar que el operador se instalo correctamente:

```bash
oc get csv -n openshift-storage | grep odf
```

Esperar hasta que el estado sea `Succeeded`.

---

### Paso 2.2 — Etiquetar los nodos worker (si aplica)

Si el cluster consumidor no tiene nodos de storage dedicados, los PVs se crean de forma remota (sin uso de disco local), pero igualmente se deben etiquetar los nodos para que los pods del CSI puedan correr:

```bash
# Listar nodos worker
oc get nodes

# Etiquetar los nodos worker (reemplazar los nombres segun el entorno)
oc label node <worker-node-1> cluster.ocs.openshift.io/openshift-storage=''
oc label node <worker-node-2> cluster.ocs.openshift.io/openshift-storage=''
oc label node <worker-node-3> cluster.ocs.openshift.io/openshift-storage=''
```

> En modo consumidor puro (sin discos locales), generalmente con etiquetar 3 nodos es suficiente para que los daemonsets del CSI corran.

---

### Paso 2.3 — Crear el StorageCluster en modo consumidor

Reemplazar los valores de `storageProviderEndpoint` y `onboardingTicket` con los obtenidos del Cluster A en el Paso 1.6.

```yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  storageProviderEndpoint: "ocs-provider-server-openshift-storage.apps.cluster-a.example.com:443"
  onboardingTicket: "<ticket-base64-obtenido-del-cluster-a>"
  storageDeviceSets: []
```

Aplicar:

```bash
oc apply -f - <<EOF
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  storageProviderEndpoint: "ocs-provider-server-openshift-storage.apps.cluster-a.example.com:443"
  onboardingTicket: "<ticket-base64-obtenido-del-cluster-a>"
  storageDeviceSets: []
EOF
```

---

### Paso 2.4 — Monitorear el proceso de registro

El operador intentara conectarse al proveedor y registrar el cluster consumidor. Seguir el progreso:

```bash
# Ver el estado del StorageCluster
oc get storagecluster ocs-storagecluster -n openshift-storage -w

# Ver logs del operador ODF para diagnostico
oc logs -n openshift-storage \
  $(oc get pod -n openshift-storage -l app=rook-ceph-operator -o jsonpath='{.items[0].metadata.name}') \
  -f

# Ver eventos del namespace
oc get events -n openshift-storage --sort-by='.lastTimestamp'
```

El proceso puede tardar entre 5 y 15 minutos. El `StorageCluster` debe llegar a `Phase: Ready`.

---

### Paso 2.5 — Verificar StorageClasses creadas

Una vez que el StorageCluster este en `Ready`, el operador crea automaticamente las StorageClasses en el Cluster B:

```bash
oc get storageclass | grep openshift-storage
```

Salida esperada (los nombres exactos pueden variar segun la version de ODF):

```
ocs-storagecluster-ceph-rbd          Delete          Immediate              true       5m
ocs-storagecluster-cephfs            Delete          Immediate              true       5m
ocs-storagecluster-ceph-rgw          Delete          Immediate              true       5m
```

| StorageClass | Tipo de acceso | Caso de uso |
|---|---|---|
| `ocs-storagecluster-ceph-rbd` | `ReadWriteOnce` | Bases de datos, almacenamiento de bloque |
| `ocs-storagecluster-cephfs` | `ReadWriteMany` | Almacenamiento compartido entre pods |
| `ocs-storagecluster-ceph-rgw` | Object Storage | S3-compatible (via aplicacion) |

---

## PARTE 3: Verificacion end-to-end

### Paso 3.1 — Verificar en el Cluster A que el consumidor se registro

```bash
# Volver al contexto del Cluster A
oc get storageconsumer -n openshift-storage
```

El `StorageConsumer` debe mostrar estado `Ready`:

```
NAME               AGE   STATE   HEALTH
cluster-b-consumer  15m   Ready   Health
```

---

### Paso 3.2 — Crear un PVC de prueba en el Cluster B

```bash
# Volver al contexto del Cluster B

# Crear un PVC de prueba con RBD (ReadWriteOnce)
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc-rbd
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-ceph-rbd
EOF

# Crear un PVC de prueba con CephFS (ReadWriteMany)
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc-cephfs
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
EOF
```

Verificar que los PVCs lleguen a estado `Bound`:

```bash
oc get pvc -n default -w
```

Salida esperada:

```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                    AGE
test-pvc-rbd      Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWO            ocs-storagecluster-ceph-rbd     30s
test-pvc-cephfs   Bound    pvc-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy   5Gi        RWX            ocs-storagecluster-cephfs       30s
```

---

### Paso 3.3 — Prueba de escritura con un Pod

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-odf
  namespace: default
spec:
  containers:
    - name: test
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      command: ["/bin/sh", "-c", "echo 'ODF Consumer OK' > /mnt/data/test.txt && cat /mnt/data/test.txt && sleep 3600"]
      volumeMounts:
        - name: storage
          mountPath: /mnt/data
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: test-pvc-rbd
EOF
```

Verificar que el pod arranque y pueda escribir:

```bash
oc logs test-pod-odf -n default
# Debe mostrar: ODF Consumer OK
```

---

### Paso 3.4 — Limpieza de recursos de prueba

```bash
oc delete pod test-pod-odf -n default
oc delete pvc test-pvc-rbd test-pvc-cephfs -n default
```

---

## Troubleshooting

### El StorageCluster del consumidor no llega a Ready

```bash
# Ver descripcion del objeto
oc describe storagecluster ocs-storagecluster -n openshift-storage

# Ver condiciones
oc get storagecluster ocs-storagecluster -n openshift-storage \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

Causas comunes:
- **Conectividad**: verificar que el Cluster B puede alcanzar el endpoint del proveedor en el puerto 443
  ```bash
  # Desde un pod de debug en el Cluster B
  oc debug node/<worker-node> -- curl -k https://ocs-provider-server-openshift-storage.apps.cluster-a.example.com:443
  ```
- **Ticket expirado o invalido**: regenerar el `onboardingTicket` en el Cluster A y actualizar el StorageCluster del Cluster B
- **Version incompatible**: verificar que ambos clusters usen la misma version de ODF

---

### El PVC queda en estado Pending

```bash
# Ver eventos del PVC
oc describe pvc <nombre-pvc> -n <namespace>

# Ver logs del CSI provisioner
oc logs -n openshift-storage \
  $(oc get pod -n openshift-storage -l app=csi-rbdplugin-provisioner -o jsonpath='{.items[0].metadata.name}') \
  -c csi-provisioner
```

Causas comunes:
- StorageClass incorrecta o inexistente
- Problemas de conectividad entre el nodo y el proveedor
- Capacidad insuficiente en el proveedor

---

### Ver estado del StorageConsumer en el proveedor

```bash
# En el Cluster A
oc get storageconsumer cluster-b-consumer -n openshift-storage -o yaml
oc describe storageconsumer cluster-b-consumer -n openshift-storage
```

---

## Referencias

- [ODF Provider-Consumer Architecture (Red Hat Docs)](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation)
- [StorageConsumer API Reference](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation)
- `oc explain storagecluster.spec` — documentacion inline del CRD
- `oc explain storageconsumer.spec` — documentacion inline del CRD

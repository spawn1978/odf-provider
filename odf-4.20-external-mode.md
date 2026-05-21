# ODF 4.20 en Modo Externo Multi-Tenant: Guía Paso a Paso

## Un cluster OpenShift con ODF brindando storage a múltiples clusters OpenShift externos

-----

## 1. ¿Es posible? ¿Está soportado por Red Hat para producción?

**Sí, está soportado oficialmente por Red Hat para entornos productivos.** OpenShift Data Foundation (ODF) 4.20 soporta el despliegue en **modo externo** (*external mode*), donde un cluster OpenShift central ejecuta ODF en modo interno (desplegando Ceph vía Rook-Ceph) y **múltiples clusters OpenShift consumer** se conectan a ese Ceph para consumir storage de forma aislada.

Red Hat recomienda explícitamente este modelo cuando:

- Los requerimientos de storage son significativos
- Múltiples clusters OpenShift necesitan consumir storage desde un punto central
- Un equipo dedicado de infraestructura gestiona el storage de forma centralizada
- Se desea separar la responsabilidad del storage de la responsabilidad de las aplicaciones

### Aislamiento Multi-Tenant

El modelo multi-tenant se logra mediante:

- **Usuarios Ceph dedicados por consumer** (CSI keyrings independientes)
- **Pools RBD aislados por tenant** (opcionalmente compartidos)
- **Permisos restringidos** via `--restricted-auth-permission`
- **Identificación única** via `--k8s-cluster-name` por cada cluster consumer

### Referencia oficial

- [ODF 4.20 - Deploying in External Mode](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20/html/deploying_openshift_data_foundation_in_external_mode/)
- [ODF 4.20 - Planning Your Deployment](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20/html/planning_your_deployment/infrastructure-requirements_rhodf)
- [ODF Supportability and Interoperability Checker](https://access.redhat.com/labs/odfsi)

-----

## 2. Arquitectura Multi-Tenant

```
                    ┌──────────────────────────────────────┐
                    │     CLUSTER CENTRAL (Provider)        │
                    │         OpenShift 4.20                │
                    │                                       │
                    │  ┌─────────────────────────────────┐  │
                    │  │    ODF 4.20 - Modo Interno      │  │
                    │  │    (Rook-Ceph desplegado)        │  │
                    │  │                                   │  │
                    │  │    MON x3   MGR x2   MDS x2      │  │
                    │  │    OSD x9+  RGW x1+              │  │
                    │  │                                   │  │
                    │  │  ┌─────────────────────────────┐  │  │
                    │  │  │        Pools Ceph            │  │  │
                    │  │  │                               │  │  │
                    │  │  │  pool-tenant-produccion       │  │  │
                    │  │  │  pool-tenant-desarrollo       │  │  │
                    │  │  │  pool-tenant-qa               │  │  │
                    │  │  │  pool-tenant-N                │  │  │
                    │  │  │  cephfs (compartido o aislado)│  │  │
                    │  │  └─────────────────────────────┘  │  │
                    │  └─────────────────────────────────┘  │
                    └──────────┬──────────┬──────────┬──────┘
                               │          │          │
              ┌────────────────┘          │          └────────────────┐
              │                           │                          │
   ┌──────────┴──────────┐   ┌───────────┴──────────┐   ┌──────────┴──────────┐
   │  CLUSTER PRODUCCIÓN │   │  CLUSTER DESARROLLO  │   │    CLUSTER QA       │
   │  OpenShift 4.20     │   │  OpenShift 4.20      │   │  OpenShift 4.20     │
   │                     │   │                      │   │                     │
   │  ODF Externo        │   │  ODF Externo         │   │  ODF Externo        │
   │  k8s-cluster-name:  │   │  k8s-cluster-name:   │   │  k8s-cluster-name:  │
   │   "produccion"      │   │   "desarrollo"       │   │   "qa"              │
   │                     │   │                      │   │                     │
   │  Pool asignado:     │   │  Pool asignado:      │   │  Pool asignado:     │
   │  pool-tenant-prod   │   │  pool-tenant-dev     │   │  pool-tenant-qa     │
   │                     │   │                      │   │                     │
   │  Usuarios Ceph:     │   │  Usuarios Ceph:      │   │  Usuarios Ceph:     │
   │  csi-rbd-prod       │   │  csi-rbd-dev         │   │  csi-rbd-qa         │
   │  csi-cephfs-prod    │   │  csi-cephfs-dev      │   │  csi-cephfs-qa      │
   └─────────────────────┘   └──────────────────────┘   └─────────────────────┘
```

### Tipos de storage disponibles para cada cluster consumer

|StorageClass                          |Tipo         |Access Mode  |Uso típico                           |
|--------------------------------------|-------------|-------------|-------------------------------------|
|`ocs-external-storagecluster-ceph-rbd`|Block (RBD)  |ReadWriteOnce|Bases de datos, aplicaciones stateful|
|`ocs-external-storagecluster-cephfs`  |File (CephFS)|ReadWriteMany|Archivos compartidos, CMS, CI/CD     |
|`ocs-external-storagecluster-ceph-rgw`|Object (RGW) |N/A (OBC)    |Object Storage via RADOS Gateway     |
|`openshift-storage.noobaa.io`         |Object (MCG) |N/A (OBC)    |Object Storage multi-cloud           |

-----

## 3. Prerrequisitos

### 3.1 Cluster Central (Provider)

- OpenShift Container Platform 4.20 instalado y funcional
- ODF 4.20 desplegado en modo interno con:
  - **Mínimo 3 nodos worker/storage** con discos dedicados (recomendado: nodos de infraestructura dedicados exclusivamente a storage)
  - MONs (x3), OSDs, MGR desplegados y saludables
  - MDS configurado (si se desea compartir CephFS)
  - RGW configurado (si se desea compartir Object Storage)
  - Ceph Dashboard instalado y configurado
  - PG Autoscaler habilitado (recomendado)
- Conectividad de red hacia todos los clusters consumer

**Dimensionamiento del cluster central:**

El cluster provider debe dimensionarse para soportar la carga agregada de todos los consumers. Considerar:

|Factor             |Recomendación                                                     |
|-------------------|------------------------------------------------------------------|
|**Capacidad bruta**|(Capacidad útil deseada) × 3 (réplica) × 1.3 (overhead)           |
|**IOPS**           |Sumar las IOPS esperadas de todos los consumers                   |
|**Nodos storage**  |Mínimo 3, escalar según OSDs necesarios                           |
|**Red**            |10 Gbps mínimo entre nodos storage; 10 Gbps hacia consumers       |
|**Pools**          |1 pool RBD por tenant (aislamiento) o pools compartidos con quotas|
|**MONs**           |3 (estándar) o 5 (para clusters muy grandes)                      |

### 3.2 Cada Cluster Consumer

- OpenShift Container Platform 4.20 instalado y funcional
- Acceso `cluster-admin`
- Suscripción válida de ODF Advanced
- Conectividad de red hacia los nodos Ceph del cluster central

### 3.3 Puertos de red requeridos (Provider ← Consumer)

|Puerto   |Protocolo|Servicio                      |
|---------|---------|------------------------------|
|3300     |TCP      |Ceph MON (v2 msgr2)           |
|6789     |TCP      |Ceph MON (v1 legacy)          |
|6800-7300|TCP      |Ceph OSD / MDS                |
|8080     |TCP      |Ceph MGR (métricas)           |
|9283     |TCP      |Ceph MGR (Prometheus exporter)|
|80/443   |TCP      |RGW (si se usa Object Storage)|


> **⚠️ Todos los nodos worker de cada cluster consumer deben poder alcanzar estos puertos en los nodos storage del cluster central.**

### 3.4 Verificar compatibilidad

Antes de comenzar, verificar la interoperabilidad en el [Red Hat ODF Supportability and Interoperability Checker](https://access.redhat.com/labs/odfsi):

1. Seleccionar **Service Type** → `ODF as Self-Managed Service`
1. Seleccionar **Version** → `4.20`
1. Revisar la pestaña **Supported RHCS versions in the External Mode**

-----

## 4. Paso a Paso: Configuración del Cluster Central (Provider)

### 4.1 Instalar el Operador ODF

1. Acceder a la consola web de OpenShift del cluster central
1. Ir a **Ecosystem → Software Catalog**
1. Buscar **OpenShift Data Foundation**
1. Click en **Install**
1. Configurar:
- **Update Channel:** `stable-4.20`
- **Installation Mode:** `A specific namespace on the cluster`
- **Installed Namespace:** `openshift-storage` (se crea automáticamente)
- **Approval Strategy:** `Automatic` o `Manual`
- **Console plugin:** `Enable`
1. Click **Install**

**Verificar:**

```bash
oc get csv -n openshift-storage | grep odf
# Debe mostrar Phase: Succeeded
```

### 4.2 Crear el StorageCluster en modo interno

Seguir la documentación oficial de despliegue interno para tu plataforma (bare metal, VMware, AWS, etc.).

**Verificar que Ceph está saludable:**

```bash
oc get storagecluster -n openshift-storage
# NAME                  AGE   PHASE   EXTERNAL   CREATED AT             VERSION
# ocs-storagecluster    1h    Ready              2025-xx-xxTxx:xx:xxZ   4.20.0

oc get cephcluster -n openshift-storage
# Debe mostrar HEALTH_OK
```

### 4.3 Preparar el aislamiento Multi-Tenant

Este es el paso más importante para un entorno multi-tenant productivo. Se debe crear infraestructura Ceph dedicada por tenant.

#### 4.3.1 Acceder al Toolbox de Rook-Ceph

```bash
oc rsh -n openshift-storage $(oc get pods -n openshift-storage \
  -l app=rook-ceph-tools -o name)
```

> Si el toolbox no está desplegado, habilitarlo:
> 
> ```bash
> oc patch OCSInitialization ocsinit -n openshift-storage \
>   --type json --patch '[{"op":"replace","path":"/spec/enableCephTools","value":true}]'
> ```

#### 4.3.2 Crear pools RBD dedicados por tenant

```bash
# Pool para el tenant "produccion"
ceph osd pool create pool-tenant-produccion 128 128 replicated
ceph osd pool application enable pool-tenant-produccion rbd
rbd pool init pool-tenant-produccion

# Pool para el tenant "desarrollo"
ceph osd pool create pool-tenant-desarrollo 64 64 replicated
ceph osd pool application enable pool-tenant-desarrollo rbd
rbd pool init pool-tenant-desarrollo

# Pool para el tenant "qa"
ceph osd pool create pool-tenant-qa 64 64 replicated
ceph osd pool application enable pool-tenant-qa rbd
rbd pool init pool-tenant-qa
```

> **Nota sobre PG count:** El número de Placement Groups (128, 64, etc.) debe ajustarse según la capacidad esperada. Con PG Autoscaler habilitado, Ceph los ajustará automáticamente.

#### 4.3.3 (Opcional) Configurar quotas por pool

Para limitar el consumo de storage por tenant:

```bash
# Limitar el pool de producción a 500 GB
ceph osd pool set-quota pool-tenant-produccion max_bytes 536870912000

# Limitar el pool de desarrollo a 200 GB
ceph osd pool set-quota pool-tenant-desarrollo max_bytes 214748364800

# Limitar el pool de QA a 100 GB
ceph osd pool set-quota pool-tenant-qa max_bytes 107374182400

# Verificar quotas
ceph osd pool get-quota pool-tenant-produccion
ceph osd pool get-quota pool-tenant-desarrollo
ceph osd pool get-quota pool-tenant-qa
```

#### 4.3.4 (Opcional) Crear CephFS filesystems aislados por tenant

Por defecto, CephFS se comparte. Si necesitas aislamiento a nivel filesystem:

```bash
# Habilitar múltiples filesystems (requiere flag experimental)
ceph fs flag set enable_multiple true

# Crear filesystem para producción
ceph osd pool create cephfs-metadata-prod 32
ceph osd pool create cephfs-data-prod 64
ceph fs new cephfs-produccion cephfs-metadata-prod cephfs-data-prod

# Crear filesystem para desarrollo
ceph osd pool create cephfs-metadata-dev 16
ceph osd pool create cephfs-data-dev 32
ceph fs new cephfs-desarrollo cephfs-metadata-dev cephfs-data-dev
```

> **Nota:** Para múltiples filesystems CephFS se necesitan instancias MDS dedicadas por filesystem. Esto agrega complejidad operacional. Evaluar si realmente se necesita este nivel de aislamiento o si un filesystem compartido con subdirectorios es suficiente.

#### 4.3.5 Verificar los pools creados

```bash
ceph osd pool ls detail | grep tenant
# o
rados lspools
```

-----

## 5. Paso a Paso: Generar configuración para cada Tenant (en el Cluster Central)

Este paso se repite **una vez por cada cluster consumer** que se conectará. El script `ceph-external-cluster-details-exporter.py` crea usuarios Ceph dedicados y genera un JSON con las credenciales específicas para ese tenant.

### 5.1 Obtener el script de exportación

```bash
# Opción 1: Extraer del operador ODF
oc get configmap rook-ceph-external-cluster-script \
  -n openshift-storage \
  -o jsonpath='{.data.ceph-external-cluster-details-exporter\.py}' \
  > ceph-external-cluster-details-exporter.py

# Opción 2: Descargarlo desde la consola web del cluster consumer
# (Storage → External Systems → Connect → Download Script)
```

### 5.2 Generar JSON para el Tenant “Producción”

```bash
python3 ceph-external-cluster-details-exporter.py \
  --rbd-data-pool-name pool-tenant-produccion \
  --monitoring-endpoint <ip-mgr-1>,<ip-mgr-2> \
  --monitoring-endpoint-port 9283 \
  --rgw-endpoint <ip-rgw>:<puerto> \
  --cephfs-filesystem-name cephfs \
  --run-as-user client.produccion \
  --restricted-auth-permission true \
  --k8s-cluster-name produccion \
  --cluster-name ceph-central \
  > tenant-produccion.json
```

### 5.3 Generar JSON para el Tenant “Desarrollo”

```bash
python3 ceph-external-cluster-details-exporter.py \
  --rbd-data-pool-name pool-tenant-desarrollo \
  --monitoring-endpoint <ip-mgr-1>,<ip-mgr-2> \
  --monitoring-endpoint-port 9283 \
  --rgw-endpoint <ip-rgw>:<puerto> \
  --cephfs-filesystem-name cephfs \
  --run-as-user client.desarrollo \
  --restricted-auth-permission true \
  --k8s-cluster-name desarrollo \
  --cluster-name ceph-central \
  > tenant-desarrollo.json
```

### 5.4 Generar JSON para el Tenant “QA”

```bash
python3 ceph-external-cluster-details-exporter.py \
  --rbd-data-pool-name pool-tenant-qa \
  --monitoring-endpoint <ip-mgr-1>,<ip-mgr-2> \
  --monitoring-endpoint-port 9283 \
  --rgw-endpoint <ip-rgw>:<puerto> \
  --cephfs-filesystem-name cephfs \
  --run-as-user client.qa \
  --restricted-auth-permission true \
  --k8s-cluster-name qa \
  --cluster-name ceph-central \
  > tenant-qa.json
```

### 5.5 Parámetros clave para Multi-Tenant

|Parámetro                     |Propósito en Multi-Tenant                                                                                   |Obligatorio                 |
|------------------------------|------------------------------------------------------------------------------------------------------------|----------------------------|
|`--rbd-data-pool-name`        |Pool RBD dedicado para este tenant                                                                          |**Sí**                      |
|`--restricted-auth-permission`|Restringe permisos CSI a pools y cluster específicos. **Crítico para aislamiento**                          |**Sí (para multi-tenant)**  |
|`--k8s-cluster-name`          |Identificador único del cluster consumer. Crea usuarios CSI con nombre único (ej: `csi-rbd-node-produccion`)|**Sí (con restricted-auth)**|
|`--cluster-name`              |Nombre del cluster Ceph. Permite diferenciar si hay múltiples clusters Ceph                                 |**Sí (con restricted-auth)**|
|`--run-as-user`               |Nombre del usuario healthchecker de este tenant                                                             |Recomendado                 |
|`--rgw-endpoint`              |Endpoint RGW (compartido entre tenants)                                                                     |Opcional                    |
|`--cephfs-filesystem-name`    |Filesystem CephFS a usar (compartido o dedicado)                                                            |Opcional                    |
|`--rados-namespace`           |Namespace RBD dentro del pool (aislamiento adicional dentro de un pool compartido)                          |Opcional                    |

### 5.6 Qué se crea en Ceph por cada tenant

Al ejecutar el script con `--restricted-auth-permission`, se crean estos usuarios Ceph **por tenant**:

```
client.csi-rbd-node-<k8s-cluster-name>          → Acceso RBD desde los nodos worker
client.csi-rbd-provisioner-<k8s-cluster-name>    → Provisioner de volúmenes RBD
client.csi-cephfs-node-<k8s-cluster-name>        → Acceso CephFS desde los nodos
client.csi-cephfs-provisioner-<k8s-cluster-name> → Provisioner de volúmenes CephFS
client.<run-as-user>                              → Healthchecker del tenant
```

Cada usuario tiene permisos **restringidos únicamente al pool asignado**, lo que garantiza que un cluster consumer no pueda acceder a los datos de otro.

**Verificar los usuarios creados:**

```bash
ceph auth ls | grep -A5 "client.csi.*produccion"
ceph auth ls | grep -A5 "client.csi.*desarrollo"
ceph auth ls | grep -A5 "client.csi.*qa"
```

### 5.7 Contenido del JSON generado

Cada JSON contiene la configuración específica del tenant:

|Recurso                      |Tipo        |Descripción                                           |
|-----------------------------|------------|------------------------------------------------------|
|`rook-ceph-mon-endpoints`    |ConfigMap   |Endpoints de los MONs del cluster central             |
|`rook-ceph-mon`              |Secret      |FSID y secrets de MON                                 |
|`rook-ceph-operator-creds`   |Secret      |Credenciales del operador para este tenant            |
|`rook-csi-rbd-node`          |Secret      |Keyring CSI RBD nodo (específico del tenant)          |
|`rook-csi-rbd-provisioner`   |Secret      |Keyring CSI RBD provisioner (específico del tenant)   |
|`rook-csi-cephfs-provisioner`|Secret      |Keyring CSI CephFS provisioner (específico del tenant)|
|`rook-csi-cephfs-node`       |Secret      |Keyring CSI CephFS nodo (específico del tenant)       |
|`ceph-rbd`                   |StorageClass|Config del pool RBD del tenant                        |
|`cephfs`                     |StorageClass|Config CephFS                                         |
|`ceph-rgw`                   |StorageClass|Config RGW                                            |
|`monitoring-endpoint`        |CephCluster |Endpoint de monitoreo                                 |
|`rook-ceph-dashboard-link`   |Secret      |Link al dashboard Ceph                                |
|`rgw-admin-ops-user`         |Secret      |Credenciales admin RGW                                |


> **⚠️ SEGURIDAD:** Cada archivo JSON contiene credenciales sensibles. Almacenarlos de forma segura (vault, secretos cifrados, etc.) y transferirlos por canales seguros a los administradores de cada cluster consumer.

-----

## 6. Paso a Paso: Configuración de cada Cluster Consumer

**Repetir esta sección completa por cada cluster consumer** (producción, desarrollo, QA, etc.).

### 6.1 Instalar el Operador ODF

1. Acceder a la consola web del cluster consumer
1. Ir a **Ecosystem → Software Catalog**
1. Buscar **OpenShift Data Foundation**
1. Click en **Install**
1. Configurar:
- **Update Channel:** `stable-4.20`
- **Installation Mode:** `A specific namespace on the cluster`
- **Installed Namespace:** `openshift-storage`
- **Approval Strategy:** `Automatic` o `Manual`
- **Console plugin:** `Enable`
1. Click **Install**

**Verificar:**

```bash
oc get csv -n openshift-storage | grep odf
# Debe mostrar Phase: Succeeded
```

### 6.2 (Opcional) Configurar node selector

```bash
oc annotate namespace openshift-storage openshift.io/node-selector=
```

### 6.3 Crear el StorageSystem en modo externo

#### Vía consola web (recomendado):

1. Ir a **Storage → External Systems**
1. Click en **Connect external systems**
1. Seleccionar **Red Hat/IBM Ceph Cluster** → **Next**
1. Seleccionar **Full deployment** como tipo de despliegue
1. En Connection details:
- Click **Browse** → subir el JSON correspondiente al tenant (ej: `tenant-produccion.json`)
- El contenido se muestra en el cuadro de texto
- Click **Next**
1. Revisar la configuración → Click **Create StorageSystem**

#### Vía CLI (alternativa):

```bash
# 1. Crear namespace
oc create namespace openshift-storage

# 2. Crear el StorageCluster en modo externo
cat <<EOF | oc apply -f -
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-external-storagecluster
  namespace: openshift-storage
spec:
  externalStorage:
    enable: true
  labelSelector: {}
EOF
```

> **Nota:** La vía por consola web es la recomendada porque procesa automáticamente el JSON y crea todos los Secrets, ConfigMaps y recursos necesarios.

-----

## 7. Verificación por Cluster Consumer

### 7.1 Verificar los Pods

```bash
oc get pods -n openshift-storage
```

**Pods esperados en cada consumer (modo externo):**

|Componente        |Pods                                                                                                                               |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------|
|ODF Operator      |`ocs-operator-*`, `ocs-metrics-exporter-*`, `odf-operator-controller-manager-*`, `odf-console-*`, `csi-addons-controller-manager-*`|
|Rook-Ceph Operator|`rook-ceph-operator-*`                                                                                                             |
|NooBaa (MCG)      |`noobaa-operator-*`, `noobaa-core-*`, `noobaa-db-pg-cluster-*`, `noobaa-endpoint-*`, `cnpg-controller-manager-*`                   |
|CSI CephFS        |`cephfs.csi.ceph.com-ctrlplugin-*` (x2), `...nodeplugin-*` (1/nodo)                                                                |
|CSI RBD           |`rbd.csi.ceph.com-ctrlplugin-*` (x2), `...nodeplugin-*` (1/nodo)                                                                   |
|CSI NFS           |`nfs.csi.ceph.com-ctrlplugin-*` (x2), `...nodeplugin-*` (1/nodo)                                                                   |


> **Nota:** Si MDS no está desplegado en el cluster central, los pods CephFS no se crearán. Si RGW no está desplegado, la StorageClass RGW no se creará.

### 7.2 Verificar conexión al Ceph central

```bash
oc get cephcluster -n openshift-storage
```

**Esperado:**

```
NAME                                      PHASE       MESSAGE                          HEALTH      EXTERNAL
ocs-external-storagecluster-cephcluster   Connected   Cluster connected successfully   HEALTH_OK   true
```

### 7.3 Verificar el StorageCluster

```bash
oc get storagecluster -n openshift-storage
```

**Esperado:**

```
NAME                          AGE   PHASE   EXTERNAL   CREATED AT             VERSION
ocs-external-storagecluster   10m   Ready   true       2025-xx-xxTxx:xx:xxZ   4.20.0
```

### 7.4 Verificar StorageClasses

```bash
oc get storageclass | grep ocs-external
```

**Esperado:**

```
ocs-external-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com   Delete   Immediate   true
ocs-external-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com Delete  Immediate   true
ocs-external-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket Delete  Immediate   false
openshift-storage.noobaa.io            openshift-storage.noobaa.io/obc       Delete   Immediate   false
```

### 7.5 Verificar vía consola web

1. **Storage → Data Foundation → Storage System** → `ocs-external-storagecluster`
1. Verificar Status con ✓ verde
1. Pestaña **Block and File** → Storage Cluster ✓ verde

### 7.6 Verificar Ceph Object Store (si aplica)

```bash
oc get cephobjectstore -n openshift-storage
```

-----

## 8. Pruebas de Storage por Tenant

### 8.1 Probar Block Storage (RBD)

```yaml
# test-rbd-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-rbd-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-external-storagecluster-ceph-rbd
---
apiVersion: v1
kind: Pod
metadata:
  name: test-rbd-pod
  namespace: default
spec:
  containers:
    - name: test
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      command: ["sleep", "3600"]
      volumeMounts:
        - name: rbd-vol
          mountPath: /mnt/rbd
  volumes:
    - name: rbd-vol
      persistentVolumeClaim:
        claimName: test-rbd-pvc
```

```bash
oc apply -f test-rbd-pvc.yaml
oc get pvc test-rbd-pvc         # Debe estar Bound
oc exec test-rbd-pod -- df -h /mnt/rbd
```

### 8.2 Probar File Storage (CephFS)

```yaml
# test-cephfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-cephfs-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-external-storagecluster-cephfs
---
apiVersion: v1
kind: Pod
metadata:
  name: test-cephfs-pod
  namespace: default
spec:
  containers:
    - name: test
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      command: ["sleep", "3600"]
      volumeMounts:
        - name: cephfs-vol
          mountPath: /mnt/cephfs
  volumes:
    - name: cephfs-vol
      persistentVolumeClaim:
        claimName: test-cephfs-pvc
```

```bash
oc apply -f test-cephfs-pvc.yaml
oc get pvc test-cephfs-pvc      # Debe estar Bound
oc exec test-cephfs-pod -- df -h /mnt/cephfs
```

### 8.3 Verificar aislamiento entre tenants

Desde el cluster central (toolbox), confirmar que cada tenant solo accede a su pool:

```bash
# Verificar permisos del usuario de producción
ceph auth get client.csi-rbd-provisioner-produccion
# Debe mostrar acceso SOLO a pool-tenant-produccion

# Verificar permisos del usuario de desarrollo
ceph auth get client.csi-rbd-provisioner-desarrollo
# Debe mostrar acceso SOLO a pool-tenant-desarrollo

# Verificar que no hay cross-access
# (un tenant NO debe poder listar imágenes RBD de otro pool)
```

-----

## 9. Encryption In-Transit (Recomendado para producción)

Para cifrar la comunicación entre los clusters consumer y el Ceph central:

```bash
# En el cluster central, dentro del toolbox
ceph config set global ms_client_mode secure
ceph config set global ms_cluster_mode secure
ceph config set global ms_service_mode secure
ceph config set global rbd_default_map_options ms_mode=secure

# Verificar
ceph config dump | grep ms_

# Esperar el reinicio progresivo de todos los daemons
ceph orch ps  # Verificar que todos los daemons reiniciaron
```

> **⚠️ Planificar una ventana de mantenimiento.** El reinicio de daemons puede causar latencia temporal en los consumers existentes.

-----

## 10. Operaciones Day-2 Multi-Tenant

### 10.1 Agregar un nuevo cluster consumer

1. En el cluster central, crear un nuevo pool para el tenant:
   
   ```bash
   ceph osd pool create pool-tenant-nuevo 64 64 replicated
   ceph osd pool application enable pool-tenant-nuevo rbd
   rbd pool init pool-tenant-nuevo
   ```
1. Ejecutar el script de exportación con los parámetros del nuevo tenant:
   
   ```bash
   python3 ceph-external-cluster-details-exporter.py \
     --rbd-data-pool-name pool-tenant-nuevo \
     --restricted-auth-permission true \
     --k8s-cluster-name nuevo-cluster \
     --cluster-name ceph-central \
     --run-as-user client.nuevo \
     > tenant-nuevo.json
   ```
1. En el nuevo cluster consumer, instalar ODF y crear el StorageSystem con el JSON generado (secciones 6.1 a 6.3).

### 10.2 Eliminar un cluster consumer

1. En el cluster consumer a eliminar:
   
   ```bash
   # Eliminar todas las PVCs y PVs primero
   oc delete pvc --all -A
   
   # Eliminar el StorageSystem
   # Seguir la documentación de uninstall de ODF external
   ```
1. En el cluster central, limpiar los usuarios y (opcionalmente) el pool:
   
   ```bash
   # Eliminar usuarios Ceph del tenant
   ceph auth del client.csi-rbd-node-<k8s-cluster-name>
   ceph auth del client.csi-rbd-provisioner-<k8s-cluster-name>
   ceph auth del client.csi-cephfs-node-<k8s-cluster-name>
   ceph auth del client.csi-cephfs-provisioner-<k8s-cluster-name>
   ceph auth del client.<run-as-user>
   
   # Eliminar el pool (CUIDADO: esto borra todos los datos)
   ceph osd pool delete pool-tenant-<nombre> pool-tenant-<nombre> \
     --yes-i-really-really-mean-it
   ```

### 10.3 Escalar storage del cluster central

El escalado se hace **solo en el cluster central**. Los consumers no necesitan cambios.

```bash
# Agregar OSDs adicionales (varía según plataforma)
# Los consumers verán automáticamente la capacidad adicional
```

### 10.4 Actualizar credenciales si cambian IPs de MON

Si los MONs del cluster central cambian de IP:

```bash
# En el cluster central, re-ejecutar para cada tenant
python3 ceph-external-cluster-details-exporter.py \
  --rbd-data-pool-name pool-tenant-produccion \
  --restricted-auth-permission true \
  --k8s-cluster-name produccion \
  --cluster-name ceph-central \
  --upgrade \
  > tenant-produccion-updated.json

# En cada cluster consumer, actualizar los secrets con el nuevo JSON
# (vía consola web o actualizando manualmente los Secrets en openshift-storage)
```

### 10.5 Monitoreo centralizado

Desde el cluster central se puede monitorear el uso de todos los tenants:

```bash
# Uso por pool
ceph osd pool stats
rados df

# Detalle por pool
ceph osd pool stats pool-tenant-produccion
ceph osd pool stats pool-tenant-desarrollo
ceph osd pool stats pool-tenant-qa

# IOPS y throughput por pool
ceph osd pool stats pool-tenant-produccion

# Quotas
ceph osd pool get-quota pool-tenant-produccion
```

-----

## 11. Consideraciones de Producción

### 11.1 Suscripciones

|Componente                 |Suscripción requerida          |
|---------------------------|-------------------------------|
|Cluster central (provider) |OCP + ODF (Premium recomendado)|
|Cada cluster consumer      |OCP + ODF Advanced             |
|RHCS standalone (si aplica)|RHCS Premium                   |

### 11.2 Alta disponibilidad del cluster central

El cluster central es un **punto único de fallo** para el storage de todos los consumers. Recomendaciones:

- Desplegar en al menos **3 zonas de disponibilidad** (si la infraestructura lo permite)
- Usar réplica 3 (estándar) para los pools
- Configurar **alertas proactivas** en Ceph para capacidad (70%, 85%, 95%)
- Tener un **plan de DR** para el cluster central
- Considerar **stretched cluster** de Ceph para HA geográfica
- Monitoreo 24x7 del cluster central

### 11.3 Rendimiento

- La latencia de red entre clusters impacta directamente el rendimiento de I/O
- Para cargas de trabajo sensibles a latencia (bases de datos), considerar si el modo externo es adecuado o si un ODF interno local es mejor opción
- Red dedicada de storage (VLAN/subred separada) es altamente recomendada
- Multus/Macvlan para aislar tráfico storage del tráfico de aplicación

### 11.4 Limitaciones del modo externo

- Los consumers **no gestionan** el cluster Ceph
- El escalado de storage se hace **solo en el cluster central**
- Auto-scaling de capacidad **no está disponible** en modo externo
- El monitoreo completo de Ceph es solo desde el cluster central
- Los parámetros del JSON (endpoints, pools) deben mantenerse estables post-configuración
- La **topología del cluster Ceph** solo se visualiza desde el modo interno

### 11.5 Seguridad Multi-Tenant

- Usar **siempre** `--restricted-auth-permission` en producción
- Cada JSON contiene keyrings Ceph: tratarlos como **secretos críticos**
- Rotar credenciales periódicamente regenerando los JSONs con `--upgrade`
- Encryption in-transit (`ms_*_mode secure`) es altamente recomendado
- Considerar encryption at-rest a nivel de OSD si los datos lo requieren

-----

## 12. Resumen del flujo completo Multi-Tenant

```
CLUSTER CENTRAL (una sola vez):
  1. Instalar OCP 4.20
  2. Instalar ODF 4.20 en modo interno
  3. Verificar Ceph saludable (HEALTH_OK)
  4. Habilitar toolbox de Rook-Ceph
  5. Crear pools RBD por tenant
  6. (Opcional) Configurar quotas por pool
  7. (Opcional) Configurar CephFS aislado por tenant
  8. (Opcional) Habilitar encryption in-transit

POR CADA CLUSTER CONSUMER (repetir N veces):
  9.  En el cluster central: ejecutar el script de exportación
      con --restricted-auth-permission y --k8s-cluster-name único
  10. Guardar el JSON generado de forma segura
  11. En el cluster consumer: instalar OCP 4.20
  12. En el cluster consumer: instalar ODF Operator 4.20
  13. En el cluster consumer: crear StorageSystem externo con el JSON
  14. Verificar: pods, conexión, storage classes
  15. Crear PVCs de prueba para validar
  16. Verificar aislamiento de permisos en Ceph
```

-----

## 13. Troubleshooting

### El CephCluster no conecta desde un consumer

```bash
# Verificar logs del operador Rook en el consumer
oc logs -n openshift-storage $(oc get pods -n openshift-storage \
  -l app=rook-ceph-operator -o name) --tail=100

# Verificar conectividad de red desde un nodo worker del consumer
oc debug node/<worker-node> -- chroot /host \
  curl -v telnet://<mon-ip>:6789

# Verificar que los secrets tienen las credenciales correctas
oc get secret rook-csi-rbd-provisioner -n openshift-storage \
  -o jsonpath='{.data.userID}' | base64 -d
```

### PVCs quedan en Pending

```bash
# Revisar eventos del PVC
oc describe pvc <nombre-pvc>

# Revisar logs del CSI provisioner
oc logs -n openshift-storage \
  $(oc get pods -n openshift-storage \
  -l app=openshift-storage.rbd.csi.ceph.com-ctrlplugin \
  -o name | head -1) -c csi-rbdplugin --tail=100

# Verificar desde el toolbox del cluster central que el pool existe
ceph osd pool ls | grep <pool-del-tenant>
```

### Un tenant ve datos de otro (problema de aislamiento)

```bash
# Verificar los permisos del usuario CSI
ceph auth get client.csi-rbd-provisioner-<k8s-cluster-name>

# Los caps deben incluir SOLO el pool asignado, ejemplo:
# caps: [osd] allow rwx pool=pool-tenant-produccion
# NO debe incluir otros pools
```

### Regenerar configuración de un tenant

```bash
# Re-ejecutar en el cluster central
python3 ceph-external-cluster-details-exporter.py \
  --rbd-data-pool-name pool-tenant-<nombre> \
  --restricted-auth-permission true \
  --k8s-cluster-name <nombre> \
  --cluster-name ceph-central \
  --upgrade \
  > tenant-<nombre>-updated.json

# Actualizar secrets en el consumer correspondiente
```

-----

*Documento generado basado en la documentación oficial de Red Hat OpenShift Data Foundation 4.20*
*Referencia: https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20/*
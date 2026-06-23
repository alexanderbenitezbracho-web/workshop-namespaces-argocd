# Argo CD + Kustomize: namespaces del workshop

GitOps para provisionar **múltiples proyectos por entorno** (desarrollo, qa, producción) en OpenShift con **Kustomize** en tres capas y sincronización automática vía **ApplicationSet** de Argo CD.

## Ramas del repositorio

| Rama | Estado | Uso |
|------|--------|-----|
| **`v1`** | Estructura activa — múltiples proyectos por entorno | ApplicationSet, overlays y documentación actual |

Remoto: [workshop-namespaces-argocd](https://github.com/alexanderbenitezbracho-web/workshop-namespaces-argocd.git)

```bash
git clone https://github.com/alexanderbenitezbracho-web/workshop-namespaces-argocd.git
cd workshop-namespaces-argocd
git checkout v1
```

---

## Guía rápida: implementar desde cero

| Paso | Acción | Comando |
|------|--------|---------|
| 0 | Clonar rama `v1` | `git checkout v1` |
| 1 | Verificar OpenShift GitOps | `oc get pods -n openshift-gitops` |
| 2 | Crear AppProject | `oc apply -f bootstrap/appproject-seguridad.yml` |
| 3 | Validar Kustomize (opcional) | `kubectl kustomize overlays/produccion/demo-app` |
| 4 | Aplicar ApplicationSet | `oc apply -f bootstrap/applicationset-workshop-namespaces.yaml` |
| 5 | Verificar Applications | `oc get applications.argoproj.io -n openshift-gitops` |
| 6 | Verificar namespaces | `oc get ns -l workshop.openshift.io/etapa=automatizacion-namespace` |

Detalle de cada paso, explicación **campo por campo** del ApplicationSet y procedimiento de teardown en [bootstrap/README.md](bootstrap/README.md).

---

## Jerarquía Kustomize (3 capas)

```
base/                          ← capa 1: recursos comunes a todos los entornos
overlays/<entorno>/env/        ← capa 2: cuotas, límites, NP por entorno
overlays/<entorno>/<proyecto>/ ← capa 3: namespace del proyecto (Application Argo CD)
```

```
                    ┌─────────────┐
                    │    base/    │  NetworkPolicy deny-all, RoleBinding
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        desarrollo/env  qa/env   produccion/env
        LimitRange      ...       ResourceQuota
        ResourceQuota             NP allow-same-entorno
        NP allow-same-entorno
        env/labels/ (Component)
              │            │            │
              ▼            ▼            ▼
           demo-app     demo-app     demo-app   ← ApplicationSet crea 1 Application c/u
```

Cada proyecto referencia `../env` (recursos del entorno) y `../env/labels` (labels heredados).

---

## Estructura del repositorio (rama `v1`)

```
workshop-namespaces-argocd/
├── README.md
├── bootstrap/
│   ├── README.md                              ← paso a paso + ApplicationSet explicado
│   ├── appproject-seguridad.yml
│   ├── applicationset-workshop-namespaces.yaml
│   └── clusterrole-base.yaml
├── cluster/
│   ├── kustomization.yaml                     # resources: [] (referencia canónica)
│   └── networkpolicy-deny-all-traffic.yaml
├── base/
│   ├── kustomization.yaml
│   ├── networkpolicy-deny-all-traffic.yaml
│   └── rolebinding-seguridad.yaml
└── overlays/
    ├── desarrollo/
    │   ├── env/
    │   │   ├── kustomization.yaml
    │   │   ├── labels/kustomization.yaml      # Component: labels al Namespace
    │   │   ├── limitrange.yaml
    │   │   ├── resourcequota.yaml
    │   │   └── networkpolicy-allow-same-entorno.yaml
    │   └── demo-app/
    │       ├── kustomization.yaml
    │       └── namespace.yaml
    ├── qa/
    │   ├── env/ ...
    │   └── demo-app/ ...
    └── produccion/
        ├── env/ ...
        └── demo-app/ ...
```

| Capa | Ruta | Contenido |
|------|------|-----------|
| `cluster/` | Definición canónica de `NetworkPolicy` deny-all. No se despliega sola. |
| `base/` | `NetworkPolicy` deny-all y `RoleBinding` compartidos por todos los entornos. |
| `overlays/<entorno>/env/` | `LimitRange`, `ResourceQuota`, NP de comunicación interna del entorno. |
| `overlays/<entorno>/env/labels/` | Kustomize Component que inyecta labels de entorno al Namespace. |
| `overlays/<entorno>/<proyecto>/` | Namespace del proyecto. Genera una Application de Argo CD. |

---

## Proyectos actuales (rama `v1`)

| Entorno | Proyecto | Namespace | Application Argo CD |
|---------|----------|-----------|---------------------|
| `desarrollo` | `demo-app` | `desarrollo-demo-app` | `ns-desarrollo-demo-app` |
| `qa` | `demo-app` | `qa-demo-app` | `ns-qa-demo-app` |
| `produccion` | `demo-app` | `produccion-demo-app` | `ns-produccion-demo-app` |

Recursos desplegados **por namespace**:

- `Namespace` (con labels)
- `ResourceQuota` `workshop-quota`
- `LimitRange` `workshop-default-limits`
- `NetworkPolicy` `deny-all-traffic`
- `NetworkPolicy` `allow-same-entorno`
- `RoleBinding` `view-seguridad`

---

## Convención de nombres

| Elemento | Convención | Ejemplo |
|----------|------------|---------|
| Carpeta del proyecto | `<proyecto>` | `demo-app`, `digital` |
| Namespace en OpenShift | `<entorno>-<proyecto>` | `produccion-demo-app` |
| Application Argo CD | `ns-<entorno>-<proyecto>` | `ns-produccion-demo-app` |

El namespace debe coincidir en:

1. `namespace.yaml` → `metadata.name`
2. `kustomization.yaml` del proyecto → `namespace:`
3. ApplicationSet → `destination.namespace` (derivado automáticamente del path)

---

## Labels y NetworkPolicy interna

Los labels de **entorno** se heredan desde `overlays/<entorno>/env/labels/` (Kustomize Component). Cada proyecto solo declara el label de **proyecto**.

| Label | Origen |
|-------|--------|
| `workshop.openshift.io/entorno` | `env/labels/` |
| `workshop.openshift.io/etapa` | `env/labels/` |
| `app.kubernetes.io/managed-by` | `env/labels/` |
| `workshop.openshift.io/proyecto` | `commonLabels` del proyecto |

Política de red resultante:

1. **`deny-all-traffic`** (`base/`) — bloquea todo ingress por defecto.
2. **`allow-same-entorno`** (`env/`) — permite ingress desde namespaces con el mismo label `workshop.openshift.io/entorno`.
3. Tráfico entre entornos distintos permanece bloqueado.

---

## Plantilla: añadir un proyecto nuevo

Crear `overlays/<entorno>/<proyecto>/` con **dos archivos obligatorios**:

**`namespace.yaml`** — solo nombre y anotaciones (sin labels de entorno):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: produccion-mi-app
  annotations:
    openshift.io/description: Descripción del proyecto
    openshift.io/display-name: Mi App (Producción)
```

**`kustomization.yaml`** — referencia env, hereda labels, declara proyecto:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

components:
  - ../env/labels

resources:
  - namespace.yaml
  - ../env

namespace: produccion-mi-app

commonLabels:
  workshop.openshift.io/proyecto: mi-app
```

Validar y publicar:

```bash
kubectl kustomize overlays/produccion/mi-app
git add overlays/produccion/mi-app/
git commit -m "Agregar proyecto mi-app en produccion"
git push origin v1
```

El ApplicationSet crea `ns-produccion-mi-app` automáticamente (~3 min).

> **Atención:** un `kustomization.yaml` sin `namespace.yaml` produce un namespace vacío de labels (Argo CD lo crea con `CreateNamespace=true` antes de aplicar el manifiesto completo).

---

## ApplicationSet (resumen)

Archivo: `bootstrap/applicationset-workshop-namespaces.yaml`

Puntos clave:

- **`revision: v1`** y **`targetRevision: v1`** — solo la rama `v1`.
- **`directories: overlays/*/*`** — detecta proyectos; **`overlays/*/env` excluido** — no trata bases de entorno como proyectos.
- **`goTemplate: true`** — nombres dinámicos con `{{index .path.segments 1}}` (entorno) y `{{.path.basename}}` (proyecto).
- **`automated.prune/selfHeal`** — Git es la fuente de verdad.
- **`CreateNamespace=true`** — crea el namespace si no existe.

Explicación detallada de cada campo: [bootstrap/README.md](bootstrap/README.md#applicationset-explicación-campo-por-campo).

---

## ResourceQuota por entorno

Valores de `overlays/<entorno>/env/resourcequota.yaml`. Se aplican a **cada proyecto** del entorno por igual.

### desarrollo

| Recurso | Límite |
|---------|--------|
| `requests.cpu` | 600m |
| `limits.cpu` | 1 |
| `requests.memory` | 1Gi |
| `limits.memory` | 2Gi |
| `requests.storage` | 2Gi |

### qa

| Recurso | Límite |
|---------|--------|
| `requests.cpu` | 600m |
| `limits.cpu` | 1 |
| `requests.memory` | 1Gi |
| `limits.memory` | 2Gi |
| `requests.storage` | 2Gi |

### produccion

| Recurso | Límite |
|---------|--------|
| `requests.cpu` | 3500m |
| `limits.cpu` | 4 |
| `requests.memory` | 10Gi |
| `limits.memory` | 16Gi |
| `requests.storage` | 20Gi |

---

## LimitRange por entorno

Valores de `overlays/<entorno>/env/limitrange.yaml`. Solo límites por **Container** (sin Pod ni PVC).

### desarrollo

| Tipo | defaultRequest | default | max | min |
|------|----------------|---------|-----|-----|
| Container | 100m CPU, 128Mi mem | 250m CPU, 400Mi mem | 250m CPU, 500Mi mem | 50m CPU, 64Mi mem |

### qa

| Tipo | defaultRequest | default | max | min |
|------|----------------|---------|-----|-----|
| Container | 100m CPU, 128Mi mem | 250m CPU, 400Mi mem | 250m CPU, 500Mi mem | 50m CPU, 64Mi mem |

### produccion

| Tipo | defaultRequest | default | max | min |
|------|----------------|---------|-----|-----|
| Container | 100m CPU, 128Mi mem | 250m CPU, 400Mi mem | 550m CPU, 1500Mi mem | 50m CPU, 64Mi mem |

---

## Validación local

```bash
kubectl kustomize overlays/desarrollo/demo-app
kubectl kustomize overlays/qa/demo-app
kubectl kustomize overlays/produccion/demo-app
```

Comprobar kinds generados:

```bash
kubectl kustomize overlays/produccion/demo-app | grep -E '^kind:'
# Namespace, ResourceQuota, LimitRange, RoleBinding, NetworkPolicy (×2)
```

Comprobar labels del Namespace:

```bash
kubectl kustomize overlays/produccion/demo-app | grep -A10 'kind: Namespace'
```

---

## Teardown (repetir el workshop)

```bash
oc delete applicationset workshop-namespaces -n openshift-gitops
oc delete ns desarrollo-demo-app qa-demo-app produccion-demo-app --wait=false
# opcional: oc delete appproject seguridad -n openshift-gitops
```

Volver a aplicar desde el [paso a paso](#guía-rápida-implementar-desde-cero).

---

## Personalización

| Objetivo | Dónde editar |
|----------|----------------|
| Deny-all en todos los entornos | `cluster/networkpolicy-deny-all-traffic.yaml` y `base/` |
| Cuota de un entorno | `overlays/<entorno>/env/resourcequota.yaml` |
| LimitRange de un entorno | `overlays/<entorno>/env/limitrange.yaml` |
| Comunicación interna del entorno | `overlays/<entorno>/env/networkpolicy-allow-same-entorno.yaml` |
| Labels de entorno | `overlays/<entorno>/env/labels/kustomization.yaml` |
| Recursos solo de un proyecto | YAML extra en `overlays/<entorno>/<proyecto>/` |
| Rama de despliegue | `bootstrap/applicationset-workshop-namespaces.yaml` → `v1` |

---

## Dependencias

- OpenShift GitOps (Argo CD) instalado en `openshift-gitops`.
- CNI con soporte de `NetworkPolicy` (OVN en OpenShift 4.x).
- Acceso al repositorio Git (público o credenciales registradas en Argo CD).
- Usuario con permisos para crear `AppProject`, `ApplicationSet` y namespaces.

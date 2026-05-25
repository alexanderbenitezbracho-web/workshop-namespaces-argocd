# Bootstrap Argo CD

Manifiestos que viven en el **clúster** (namespace `openshift-gitops`), no en los namespaces `desarrollo` / `qa` / `produccion`.

## AppProject

Se usa el proyecto existente **`seguridad`** (`openshift-gitops`), con acceso amplio a repos y destinos. No hay `appproject.yaml` en este directorio.

## ApplicationSet

| Fichero | Recurso | Efecto |
|---------|---------|--------|
| `applicationset-workshop-namespaces.yaml` | `ApplicationSet` `workshop-namespaces` | Una `Application` por carpeta en `overlays/*` del repo GitHub |

Repositorio: [workshop-namespaces-argocd](https://github.com/alexanderbenitezbracho-web/workshop-namespaces-argocd.git), rama `main`.

Applications generadas:

- `ns-desarrollo` → `overlays/desarrollo`
- `ns-qa` → `overlays/qa`
- `ns-produccion` → `overlays/produccion`

## Despliegue

```bash
oc apply -f /opt/workshop/automatizacion-namespace/argocd-namespaces/bootstrap/applicationset-workshop-namespaces.yaml \
  -n openshift-gitops
```

Comprobar:

```bash
oc get applicationset workshop-namespaces -n openshift-gitops
oc get applications -n openshift-gitops -l workshop.openshift.io/etapa=automatizacion-namespace
```

Si el repo es privado, registra credenciales en Argo CD antes del sync.

## Estructura del repo Git

```
workshop-namespaces-argocd/   (raíz en GitHub)
├── bootstrap/                ← este directorio (no lo lista overlays/*)
├── base/
├── cluster/
├── overlays/
└── README.md
```

El generador `directories` solo incluye `overlays/*`; `bootstrap/` no crea una Application extra.

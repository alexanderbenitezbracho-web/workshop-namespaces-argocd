# Bootstrap Argo CD

Manifiestos que viven en el **clúster** (namespace `openshift-gitops`), no en los namespaces `desarrollo` / `qa` / `produccion`.

## Recursos

| Fichero | Recurso | Efecto |
|---------|---------|--------|
| `appproject-seguridad.yml` | `AppProject` `seguridad` | Proyecto Argo CD con acceso a repos y destinos del workshop |
| `applicationset-workshop-namespaces.yaml` | `ApplicationSet` `workshop-namespaces` | Una `Application` por carpeta en `overlays/*` del repo GitHub |

Repositorio: [workshop-namespaces-argocd](https://github.com/alexanderbenitezbracho-web/workshop-namespaces-argocd.git), rama `main`.

Applications generadas:

- `ns-desarrollo` → `overlays/desarrollo`
- `ns-qa` → `overlays/qa`
- `ns-produccion` → `overlays/produccion`

## Despliegue

```bash
cd /opt/workshop/automatizacion-namespace/workshop-namespaces-argocd

# 1. AppProject (omitir si ya existe)
oc get appproject seguridad -n openshift-gitops 2>/dev/null \
  || oc apply -f bootstrap/appproject-seguridad.yml

# 2. ApplicationSet
oc apply -f bootstrap/applicationset-workshop-namespaces.yaml -n openshift-gitops
```

Comprobar:

```bash
oc get applicationset workshop-namespaces -n openshift-gitops
oc get applications.argoproj.io -n openshift-gitops \
  -l workshop.openshift.io/etapa=automatizacion-namespace
```

Si el repo es privado, registra credenciales en Argo CD antes del sync.

## Eliminación

```bash
oc delete applicationset workshop-namespaces -n openshift-gitops

# Namespaces residuales (si aplica)
for ns in desarrollo qa produccion; do
  oc delete ns "$ns" --ignore-not-found --wait=false
done

# AppProject (solo si no lo usa otra Application)
oc delete appproject seguridad -n openshift-gitops --ignore-not-found
```

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

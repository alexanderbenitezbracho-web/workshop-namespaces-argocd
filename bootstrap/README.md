# Bootstrap Argo CD

Manifiestos que viven en el **clúster** (namespace `openshift-gitops`), no en los namespaces de los proyectos.

## AppProject

Se usa el proyecto existente **`seguridad`** (`openshift-gitops`), con acceso amplio a repos y destinos. No hay `appproject.yaml` en este directorio.

## ApplicationSet

| Fichero | Recurso | Efecto |
|---------|---------|--------|
| `applicationset-workshop-namespaces.yaml` | `ApplicationSet` `workshop-namespaces` | Una `Application` por carpeta de proyecto en `overlays/<entorno>/<proyecto>/` |

Repositorio: [workshop-namespaces-argocd](https://github.com/alexanderbenitezbracho-web/workshop-namespaces-argocd.git), rama `main`.

### Generador de directorios

```yaml
directories:
  - path: overlays/*/*
  - path: overlays/*/env
    exclude: true
```

- **Incluye**: `overlays/desarrollo/demo-app`, `overlays/qa/demo-app`, etc.
- **Excluye**: `overlays/desarrollo/env`, `overlays/qa/env`, etc. (bases de entorno, no proyectos).

### Applications generadas (ejemplos)

| Application | Path Git | Namespace destino |
|-------------|----------|-------------------|
| `ns-desarrollo-demo-app` | `overlays/desarrollo/demo-app` | `desarrollo-demo-app` |
| `ns-desarrollo-portal` | `overlays/desarrollo/portal` | `desarrollo-portal` |
| `ns-qa-demo-app` | `overlays/qa/demo-app` | `qa-demo-app` |
| `ns-produccion-demo-app` | `overlays/produccion/demo-app` | `produccion-demo-app` |

Convención de nombres: `ns-<entorno>-<proyecto>` → namespace `<entorno>-<proyecto>`.

## Despliegue

```bash
oc apply -f bootstrap/applicationset-workshop-namespaces.yaml -n openshift-gitops
```

Comprobar:

```bash
oc get applicationset workshop-namespaces -n openshift-gitops
oc get applications -n openshift-gitops -l workshop.openshift.io/etapa=automatizacion-namespace
```

Si el repo es privado, registra credenciales en Argo CD antes del sync.

## Estructura del repo Git

```
workshop-namespaces-argocd/
├── bootstrap/                ← este directorio (no lo lista overlays/*/*)
├── base/
├── cluster/
├── overlays/
│   ├── desarrollo/
│   │   ├── env/              ← excluido del ApplicationSet
│   │   ├── demo-app/         ← Application ns-desarrollo-demo-app
│   │   └── portal/
│   └── ...
└── README.md
```

Al añadir una carpeta bajo `overlays/<entorno>/<proyecto>/`, el ApplicationSet crea la Application correspondiente en el próximo refresh (sin editar este manifiesto).

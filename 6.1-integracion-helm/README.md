# 6.1 Integración con Helm

En esta sección se mostrará la manera en la que Flux utiliza recursos para integrarse con [Helm](https://helm.sh/).

Vídeo de la explicación y la demo completa en este [enlace](https://www.youtube.com/watch?v=wQZ01-3vXBI&list=PLuQL-CB_D1E7gRzUGlchvvmGDF1rIiWkj&index=6).

## Requisitos

* Acceso para administrar un cluster de Kubernetes >=v1.19
* Tener instalado cliente Flux >=0.13.2

## Exportar token de GitHub

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Instalar Flux en el cluster

Utilice el comando `bootstrap` para instalar los componentes de flux en el cluster, crear el repositorio en GitHub y mucho más:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=gitops-flux-series-demo \
  --branch=main \
  --private=false \
  --path=./clusters/demo
```

<details>
  <summary>Resultado</summary>

  ```bash
  ► connecting to github.com
  ✔ repository "https://github.com/sngular/gitops-flux-series-demo" created
  ► cloning branch "main" from Git repository "https://github.com/sngular/gitops-flux-series-demo.git"
  ✔ cloned repository
  ► generating component manifests
  ✔ generated component manifests
  ✔ committed sync manifests to "main" ("f20fb16201be4cedc86860139c4c30a7a5569bf3")
  ► pushing component manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► installing components in "flux-system" namespace
  ✔ installed components
  ✔ reconciled components
  ► determining if source secret "flux-system/flux-system" exists
  ► generating source secret
  ✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC42KfDLo5DDDJU+KcLtT155hVQ3Gtd/IQLO2RRqshtRcnGmNebupSzea9CRi2sEzk+cNStXYpci0DWXY7joRnInMg+K/YwPYQGDfL373UNOi7pW6KqnlPmgxvqKXRHIh2/N4PWm+lG43Iq625xHKF1ITzEHPrdRULKB1uF1qHHOJFDTCJKPJrkZBrBspkJc4O/eKzloEjXuBlFwoWm/YvFo04kk3MRqKGGcOB/euxN5xeHgtq2nIS8m1qdJxHvkSA2zgVw3URYWEX+x5qz2zsM9w7Kj9TghmrquICnGkpF6Q7OcDh1MmX+1mrTjkvW//Nlua2x91y/4LVpsWAJDEHL
  ✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/sngular/gitops-flux-series-demo"
  ► applying source secret "flux-system/flux-system"
  ✔ reconciled source secret
  ► generating sync manifests
  ✔ generated sync manifests
  ✔ committed sync manifests to "main" ("53202cc8bd759a3e32e6dcc8e8c9b5968c7112e2")
  ► pushing sync manifests to "https://github.com/sngular/gitops-flux-series-demo.git"
  ► applying sync manifests
  ✔ reconciled sync configuration
  ◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
  ✔ Kustomization reconciled successfully
  ► confirming components are healthy
  ✔ source-controller: deployment ready
  ✔ kustomize-controller: deployment ready
  ✔ helm-controller: deployment ready
  ✔ notification-controller: deployment ready
  ✔ all components are healthy
  ```
</details>

Comprobar que los componentes han sido instalados:

```bash
  kubectl get pods --namespace flux-system
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                                       READY   STATUS    RESTARTS   AGE
  source-controller-85fb864746-4x4s2         1/1     Running   0          65s
  helm-controller-85bfd4959d-lsshl           1/1     Running   0          66s
  notification-controller-5c4d48f476-qltpw   1/1     Running   0          65s
  kustomize-controller-6977b8cdd4-qq482      1/1     Running   0          66s
  ```
</details>

## Clonar repositorio creado

```bash
{
  git clone git@github.com:$GITHUB_USER/gitops-flux-series-demo.git
  cd gitops-flux-series-demo
}
```

## Añadir al cluster un repositorio de Helm charts como fuente

Crear carpeta `sources`:
```bash
mkdir -p ./clusters/demo/sources
```

```bash
flux create source helm sngular \
  --url=https://sngular.github.io/gitops-helmrepository/ \
  --interval=5m \
  --namespace=flux-system \
  --export > clusters/demo/sources/sngular-helmrepository.yaml
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: source.toolkit.fluxcd.io/v1beta1
  kind: HelmRepository
  metadata:
    name: sngular
    namespace: flux-system
  spec:
    interval: 5m0s
    url: https://sngular.github.io/gitops-helmrepository/
  ```
</details>

Compruebe la nueva estructura del repositorio:

```bash
tree
.
└── clusters
    └── demo
        ├── flux-system
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        └── sources
            └── sngular-helmrepository.yaml

4 directories, 4 files
```

Realice un commit con los cambios al repositorio de código:

```bash
{
  git add .
  git commit -m 'Add Sngular Helm chart repository'
  git push origin main
}
```

Esperar a que se realice la sincronización del repositorio o indicarle a Flux que realice el ciclo de reconciliación de manera inmediata:

```bash
flux reconcile kustomization flux-system --with-source
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating GitRepository flux-system in flux-system namespace
  ✔ GitRepository annotated
  ◎ waiting for GitRepository reconciliation
  ✔ GitRepository reconciliation completed
  ✔ fetched revision main/0fe4240274ab1c54d6f8178635a63fe0d33e9685
  ► annotating Kustomization flux-system in flux-system namespace
  ✔ Kustomization annotated
  ◎ waiting for Kustomization reconciliation
  ✔ Kustomization reconciliation completed
  ✔ applied revision main/0fe4240274ab1c54d6f8178635a63fe0d33e9685
  ```
</details>

Comprobar que el estado del objeto `HelmRepository` en el campo `READY` sea `True`:

```bash
flux get source helm --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE  	NAME   	READY	MESSAGE                                                   	REVISION                                	SUSPENDED
  flux-system	sngular	True 	Fetched revision: 3f33f697ef0499ad9d54052b1e791c271df1dffd	3f33f697ef0499ad9d54052b1e791c271df1dffd	False
  ```
</details>

## Desplegar una release de Helm

Crear la carpeta `gitops-series` para almacenar los manifiestos de despliegue:

```bash
mkdir -p ./clusters/demo/gitops-series
```

Crear el fichero del namespace:

```bash
cat <<EOF > ./clusters/demo/gitops-series/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-series
EOF
```

Crear el fichero `helmrelease` a través del comando `flux create`:

```bash
flux create helmrelease echobot \
    --interval=1m \
    --source=HelmRepository/sngular.flux-system \
    --chart=echobot \
    --chart-version="0.2.1" \
    --namespace=gitops-series \
    --export > clusters/demo/gitops-series/echobot-helmrelease.yaml
```

<details>
  <summary>Resultado</summary>

  ```
  ---
  apiVersion: helm.toolkit.fluxcd.io/v2beta1
  kind: HelmRelease
  metadata:
    name: echobot
    namespace: gitops-series
  spec:
    chart:
      spec:
        chart: echobot
        sourceRef:
          kind: HelmRepository
          name: sngular
          namespace: flux-system
        version: 0.2.1
    interval: 10m0s
  ```
</details>

Añadir los cambios en el repositorio:

```bash
{
  git add .
  git commit -m 'Add echobot helmrelease file'
  git push origin main
}
```

Sincronizar la información sin esperara al ciclo de reconciliación:

```bash
flux reconcile source git flux-system
```

```bash
flux reconcile helmrelease echobot --namespace=gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  ► annotating HelmRelease echobot in gitops-series namespace
  ✔ HelmRelease annotated
  ◎ waiting for HelmRelease reconciliation
  ✔ HelmRelease reconciliation completed
  ✔ applied revision 0.2.1
  ```
</details>

Listar los chart registrados por flux:

```bash
flux get sources chart --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE  	NAME                 	READY	MESSAGE                	REVISION	SUSPENDED
  flux-system	gitops-series-echobot	True 	Fetched revision: 0.2.1	0.2.1   	False
  ```
</details>

Listar los objetos `helmreleases` desplegados:

```bash
flux get helmrelease --all-namespaces
```

<details>
  <summary>Resultado</summary>

  ```
  NAMESPACE    	NAME   	READY	MESSAGE                         	REVISION	SUSPENDED
  gitops-series	echobot	True 	Release reconciliation succeeded	0.2.1   	False
  ```
</details>

Listar los pods del servicio desplegado:

```bash
kubectl get pods --namespace gitops-series
```

<details>
  <summary>Resultado</summary>

  ```
  NAME                      READY   STATUS    RESTARTS   AGE
  echobot-bcfb77fcd-cqnqj   1/1     Running   0          7m42s
  ```
</details>

## (Opcional) Desintalar Flux

Utilice el siguiente comando para desintalar flux del cluster:

```bash
flux uninstall
```

> Compruebe que el repositorio en GitHub no ha sido eliminado.

<details>
  <summary>Resultado</summary>

  ```bash
  Are you sure you want to delete Flux and its custom resource definitions: y█
  ► deleting components in flux-system namespace
  ✔ Deployment/flux-system/helm-controller deleted
  ✔ Deployment/flux-system/kustomize-controller deleted
  ✔ Deployment/flux-system/notification-controller deleted
  ✔ Deployment/flux-system/source-controller deleted
  ✔ Service/flux-system/notification-controller deleted
  ✔ Service/flux-system/source-controller deleted
  ✔ Service/flux-system/webhook-receiver deleted
  ✔ NetworkPolicy/flux-system/allow-egress deleted
  ✔ NetworkPolicy/flux-system/allow-scraping deleted
  ✔ NetworkPolicy/flux-system/allow-webhooks deleted
  ✔ ServiceAccount/flux-system/helm-controller deleted
  ✔ ServiceAccount/flux-system/kustomize-controller deleted
  ✔ ServiceAccount/flux-system/notification-controller deleted
  ✔ ServiceAccount/flux-system/source-controller deleted
  ✔ ClusterRole/crd-controller-flux-system deleted
  ✔ ClusterRoleBinding/cluster-reconciler-flux-system deleted
  ✔ ClusterRoleBinding/crd-controller-flux-system deleted
  ► deleting toolkit.fluxcd.io finalizers in all namespaces
  ✔ GitRepository/flux-system/flux-system finalizers deleted
  ✔ Kustomization/flux-system/flux-system finalizers deleted
  ► deleting toolkit.fluxcd.io custom resource definitions
  ✔ CustomResourceDefinition/alerts.notification.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/buckets.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/gitrepositories.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/helmcharts.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/helmreleases.helm.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/helmrepositories.source.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/kustomizations.kustomize.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/providers.notification.toolkit.fluxcd.io deleted
  ✔ CustomResourceDefinition/receivers.notification.toolkit.fluxcd.io deleted
  ✔ Namespace/flux-system deleted
  ✔ uninstall finished
  ```
</details>

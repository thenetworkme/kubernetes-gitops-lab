# Kubernetes + Helm + Argo CD: práctica GitOps

Chart de Nginx para practicar despliegue local con Minikube, versionado en Git y sincronización automática mediante Argo CD.

## Estructura

```text
kubernetes-gitops-lab/
├── argocd/
│   └── application.yaml
└── charts/
    └── nginx-gitops/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── _helpers.tpl
            ├── configmap.yaml
            ├── deployment.yaml
            ├── service.yaml
            └── NOTES.txt
```

## Requisitos para Windows

- Docker Desktop ejecutándose con contenedores Linux.
- `kubectl`, Minikube, Helm y Git disponibles en PowerShell.
- Una cuenta GitHub y un repositorio público para que Argo CD pueda leerlo sin credenciales adicionales.

Instaladores sugeridos desde PowerShell:

```powershell
winget install Kubernetes.minikube
winget install Helm.Helm
```

Docker Desktop suele instalar `kubectl`. Comprueba las herramientas y abre una terminal nueva tras instalarlas:

```powershell
docker version
kubectl version --client
minikube version
helm version
git --version
```

## Parte 1 - Iniciar Minikube

Este proyecto usa Minikube con el driver de Docker. Inicia Docker Desktop primero:

```powershell
minikube start --driver=docker --cpus=2 --memory=4096
kubectl config current-context
kubectl get nodes
```

La salida de `kubectl get nodes` debe mostrar el nodo `minikube` como `Ready`. Si muestra un error `EOF`, comprueba el contexto actual, que Docker Desktop esté activo y vuelve a correr `minikube start`. No instales Argo CD hasta que la API de Kubernetes responda y el nodo esté `Ready`.

## Parte 2 - Probar el Chart localmente

Desde la carpeta raíz del repositorio:

```powershell
helm lint .\charts\nginx-gitops
helm template nginx-gitops .\charts\nginx-gitops
helm install nginx-gitops .\charts\nginx-gitops --namespace gitops-demo --create-namespace
kubectl -n gitops-demo get deploy,svc,pods
```

Espera a que el Pod esté `Running` y `READY 1/1`. Para abrir la página localmente:

```powershell
kubectl port-forward -n gitops-demo svc/nginx-gitops 8081:80
```

Mantén esa terminal abierta y visita `http://localhost:8081` en el navegador.

Captura sugerida: la terminal con `helm lint`, `helm install` y `kubectl get deploy,svc,pods`; otra captura de la página en el navegador.

Retira la instalación manual antes de que Argo CD gestione el mismo Chart:

```powershell
helm uninstall nginx-gitops --namespace gitops-demo
kubectl delete namespace gitops-demo
```

## Parte 3 - Crear el repositorio GitHub

1. En GitHub, crea un repositorio público llamado `kubernetes-gitops-lab`, sin README, `.gitignore` ni licencia precreados.
2. En PowerShell, entra a la carpeta de este proyecto y configura Git si aún no lo has hecho:

```powershell
git config --global user.name "Rayner Jose Rodriguez Jimenez"
git config --global user.email "TU_CORREO_DE_GITHUB"
```

3. Inicializa y publica la rama `main` (reemplaza el usuario):

```powershell
git init -b main
git add .
git commit -m "feat: add nginx Helm chart and Argo CD application"
git remote add origin https://github.com/TU_USUARIO/kubernetes-gitops-lab.git
git push -u origin main
```

4. Abre en el navegador `https://github.com/TU_USUARIO/kubernetes-gitops-lab` y verifica que se vean `charts/nginx-gitops/Chart.yaml`, `charts/nginx-gitops/values.yaml`, `charts/nginx-gitops/templates/` y `argocd/application.yaml`.

Captura sugerida: la página del repositorio mostrando el Chart y la carpeta `templates`.

## Parte 4 - Argo CD y aplicación

Realiza esta parte únicamente cuando `kubectl get nodes` confirme que el nodo de tu clúster está `Ready`.

1. Instala Argo CD siguiendo estos comandos oficiales para el clúster de práctica:

```powershell
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

2. Actualiza `repoURL` en `argocd/application.yaml` con la URL de tu repositorio, guarda el cambio, haz `git add`, `git commit` y `git push`.
3. Aplica la Application:

```powershell
kubectl apply -n argocd -f .\argocd\application.yaml
kubectl get applications -n argocd
```

4. En otra terminal, abre el túnel y visita `https://localhost:8080`:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

El usuario inicial es `admin`. Recupera la contraseña en PowerShell:

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
```

La aplicación debe quedar `Healthy` y `Synced`.

Captura sugerida: la UI de Argo CD mostrando `nginx-gitops`, `Healthy` y `Synced`; captura también el árbol de recursos Deployment, Service y ConfigMap.

## Parte 5 - Demostrar GitOps y rollback

1. En `charts/nginx-gitops/values.yaml`, cambia `replicaCount: 2` por `replicaCount: 3`.
2. Guarda y publica el cambio:

```powershell
git add .\charts\nginx-gitops\values.yaml
git commit -m "feat: scale nginx to three replicas"
git push
```

3. Espera a que Argo CD muestre la sincronización automática. Comprueba:

```powershell
kubectl get deploy,svc -n gitops-demo
kubectl get pods -n gitops-demo
```

El Deployment debe indicar tres réplicas disponibles. Captura la UI de Argo CD y la terminal mostrando `3/3`.

4. Para demostrar rollback por Git, revierte el cambio de réplicas:

```powershell
git revert HEAD
git push
```

Argo CD volverá a sincronizar `replicaCount: 2`. Valídalo otra vez con `kubectl get deploy,svc,pods -n gitops-demo` y captura `2/2`.

## Explicación breve del flujo GitOps

El repositorio Git contiene el estado deseado del Chart. Argo CD observa la rama `main`, compara sus manifiestos con los recursos del clúster y sincroniza automáticamente los cambios. Al revertir el commit que cambió las réplicas, Git vuelve a declarar dos Pods y Argo CD reconcilia el clúster con ese estado.

## Evidencias para entregar

- `kubectl get nodes` con el nodo `Ready`.
- `helm lint` satisfactorio y `kubectl get deploy,svc,pods` tras la instalación manual.
- Repositorio GitHub con `Chart.yaml`, `values.yaml` y `templates/`.
- UI de Argo CD en estado `Healthy` y `Synced`.
- Deployment actualizado a `3/3` y restaurado a `2/2` tras el `git revert`.

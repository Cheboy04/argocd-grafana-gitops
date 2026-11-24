# argocd-grafana-gitops

Despliegue GitOps de **Grafana** en Kubernetes con **Argo CD**: Deployment, Service NodePort y Horizontal Pod Autoscaler. Este repositorio es la fuente de verdad, y Argo CD sincroniza automáticamente todo lo que está en `k8s/`.

## Estructura

```
k8s/
├── deployment.yml   # Grafana 12.1.0, 2 réplicas, requests/limits de CPU y memoria
├── service.yml      # NodePort 30300 -> 3000
└── hpa.yml          # 2–5 réplicas, objetivo 50 % de CPU
argocd/
└── application.yml  # Application de Argo CD (auto-sync, prune, self-heal)
```

## Requisitos

- Clúster Kubernetes (probado en Minikube con Kubernetes v1.35) con **Argo CD** instalado en el namespace `argocd`.
- **Metrics Server**, que el HPA necesita: `minikube addons enable metrics-server`

## Despliegue

1. Crear el Secret con las credenciales de Grafana. El `.env` no se versiona:

   ```bash
   cat > .env <<'EOF'
   GF_SECURITY_ADMIN_USER=admin
   GF_SECURITY_ADMIN_PASSWORD=cambia-esta-clave
   EOF
   kubectl create namespace grafana-gitops
   kubectl create secret generic grafana-admin -n grafana-gitops --from-env-file=.env
   ```

2. Registrar la aplicación en Argo CD:

   ```bash
   kubectl apply -f argocd/application.yml
   ```

3. Verificar:

   ```bash
   kubectl get applications -n argocd          # Synced / Healthy
   kubectl get all,hpa -n grafana-gitops
   minikube service grafana -n grafana-gitops --url
   ```

Desde ese momento, cualquier cambio que llegue a `main` en `k8s/` lo aplica Argo CD solo.

## Prueba de autoescalado

```bash
kubectl run carga -n grafana-gitops --rm -it --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c 'for i in 1 2 3 4 5 6 7 8 9 10; do (while true; do wget -q -O- http://grafana:3000/login >/dev/null 2>&1; done) & done; wait'

kubectl get hpa grafana -n grafana-gitops -w   # en otra terminal
```

Con la carga, el HPA sube hasta 5 réplicas. Al detenerla con `Ctrl+C`, vuelve a 2 en unos minutos.

## Nota sobre HPA y Argo CD

La Application ignora `/spec/replicas` del Deployment (`ignoreDifferences` y `RespectIgnoreDifferences=true`). Sin esto, Argo CD devolvería las réplicas al valor de Git cada vez que el HPA escalara.

# Killercoda Argocd and Prometheus & Grafana Setup

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge -p '{"data":{"server.insecure":"true"}}'
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitor prometheus-community/kube-prometheus-stack -n monitoring
kubectl patch svc monitor-grafana -n monitoring -p '{"spec":{"type":"NodePort"}}'
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
kubectl get svc monitor-grafana -n monitoring
kubectl get svc argocd-server -n argocd

```

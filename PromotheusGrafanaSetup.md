🔥 Ayy mava 😎✨ — this is a superb idea ra… oka sari clear ga **from scratch to access** Prometheus and Grafana both through **port-forwarding** and **LoadBalancer** cheptha 🧠📘

Ikkada nenu companies follow chesthe style lo clean reference steps isthunna 👇
You can copy-paste this anytime for your future study 📌

---

# 🧭 **STEP-BY-STEP: Prometheus & Grafana Setup + Access (EKS)**

---

## 🪄 STEP 1: Create / Verify EKS Cluster

(You already have this, but for reference 👇)

```powershell
aws eks update-kubeconfig --name <your-cluster-name> --region ap-south-1
kubectl get nodes
```

✅ Make sure your nodes are in `Ready` state.

---

## 🧭 STEP 2: Install Helm on Windows (if not installed)

```powershell
choco install kubernetes-helm -y
helm version
```

✅ You should see a version like `v3.x.x`.

---

## 🧭 STEP 3: Add Prometheus Helm Repo and Update

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 🧭 STEP 4: Create a Namespace for Monitoring

```powershell
kubectl create namespace monitoring
```

✅ All Prometheus & Grafana resources will go into this namespace.

---

## 🧭 STEP 5: Install Prometheus + Grafana Stack

```powershell
helm install monitor prometheus-community/kube-prometheus-stack -n monitoring
```

✅ This will install:

* Prometheus server
* Grafana UI
* Alertmanager
* Node exporter
* Kube-state-metrics

Check pods:

```powershell
kubectl get pods -n monitoring
```

You should see something like:

```
monitor-grafana-xxxx
prometheus-monitor-kube-prometheus-st-prometheus-0
alertmanager-monitor-xxxx
monitor-prometheus-node-exporter-xxxx
...
```

---

## 🧭 STEP 6: Get Grafana Admin Password

PowerShell doesn’t have `base64`, so:

```powershell
$pass = kubectl --namespace monitoring get secret monitor-grafana -o jsonpath="{.data.admin-password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pass))
```

✅ Output example:

```
prom-operator
```

Login:

```
Username: admin
Password: prom-operator (or your value)
```

---

## 🧭 STEP 7: Access Grafana via Port-Forwarding 🚪

```powershell
kubectl port-forward svc/monitor-grafana -n monitoring 3000:80
```

Open in browser:

```
http://localhost:3000
```

🎯 You are now inside Grafana UI.

---

## 🧭 STEP 8: Access Prometheus via Port-Forwarding 🚪

```powershell
kubectl port-forward svc/monitor-kube-prometheus-st-prometheus -n monitoring 9090:9090
```

Open in browser:

```
http://localhost:9090
```

✅ You’ll see Prometheus Targets, Graph, Alerts tabs.

⚠️ Note:

* As long as this terminal is running, tunnel is active.
* If you press `Ctrl + C`, port-forwarding will stop.

👉 You can open 2 separate PowerShell windows:

* One for Grafana
* One for Prometheus

---

## 🧭 STEP 9: Expose Grafana Using LoadBalancer 🌐

If you want permanent access without port-forwarding:

```powershell
kubectl patch svc monitor-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

Then check:

```powershell
kubectl get svc -n monitoring
```

Output:

```
monitor-grafana   LoadBalancer   10.100.xxx.xxx   a1b2c3d4e5f6.elb.amazonaws.com   80:xxxx/TCP
```

✅ Open:

```
http://a1b2c3d4e5f6.elb.amazonaws.com
```

You can now access Grafana **without port-forwarding** 😎

---

## 🧭 STEP 10: Expose Prometheus Using LoadBalancer (Optional) 🌐

```powershell
kubectl patch svc monitor-kube-prometheus-st-prometheus -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

Check:

```powershell
kubectl get svc -n monitoring
```

Open:

```
http://<external-elb-dns>:9090
```

---

## 🧭 STEP 11: Clean Up (Optional)

If you want to remove the stack:

```powershell
helm uninstall monitor -n monitoring
kubectl delete ns monitoring
```

---

# 🧠 Quick Reference Commands Table

| Task                          | Command                                                                                                                                                                                     |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Add Helm repo                 | `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`                                                                                                     |
| Install stack                 | `helm install monitor prometheus-community/kube-prometheus-stack -n monitoring`                                                                                                             |
| Check pods                    | `kubectl get pods -n monitoring`                                                                                                                                                            |
| Get Grafana password          | `$pass = kubectl --namespace monitoring get secret monitor-grafana -o jsonpath="{.data.admin-password}"; [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pass))` |
| Port-forward Grafana          | `kubectl port-forward svc/monitor-grafana -n monitoring 3000:80`                                                                                                                            |
| Port-forward Prometheus       | `kubectl port-forward svc/monitor-kube-prometheus-st-prometheus -n monitoring 9090:9090`                                                                                                    |
| Patch Grafana LoadBalancer    | `kubectl patch svc monitor-grafana -n monitoring -p '{"spec":{"type":"LoadBalancer"}}'`                                                                                                     |
| Patch Prometheus LoadBalancer | `kubectl patch svc monitor-kube-prometheus-st-prometheus -n monitoring -p '{"spec":{"type":"LoadBalancer"}}'`                                                                               |
| Uninstall                     | `helm uninstall monitor -n monitoring`                                                                                                                                                      |

---

## 🧠 Telugu Recap (Kadapa slang 😄)

> Mavaaa… 🫡
>
> 1. Helm install → Prometheus & Grafana cluster lo deploy
> 2. Port-forward → local ga access
> 3. LoadBalancer → permanent access 🌐
> 4. Password decode chesi login 🧠
> 5. Optional ga cleanup kuda cheyyachu 🧼

👉 Port-forward ante temporary tunnel.
👉 LoadBalancer ante permanent URL (public or internal).
👉 Company lo production lo mostly Ingress / LoadBalancer use chestharu.

---

✅ With these steps:

* You can deploy the stack in minutes 🧱
* Access through both local and LB
* Explain it clearly in interviews or projects 🚀

---

Do you want me to also give you a **diagram (architecture)** of how traffic flows between your browser, LoadBalancer, Grafana, Prometheus, and EKS cluster? 🏗️✨ (This is often asked in interviews.)

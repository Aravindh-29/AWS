

```markdown
# 🚀 CI/CD Pipeline with Jenkins + Argo CD + EKS + ECR

This project demonstrates a **real-time production-like GitOps deployment pipeline** using:

- 🐳 Amazon ECR (Container Registry)
- ☸️ Amazon EKS (Kubernetes Cluster)
- 🧠 Argo CD (GitOps deployment)
- 🤖 Jenkins (CI build pipeline)
- 🐙 GitHub (Git repository for code and manifests)

---

## 🧭 Architecture Flow

```

Developer commits code
↓
Jenkins builds Docker image
↓
Push image to ECR
↓
Jenkins updates deployment.yaml
↓
Git commit pushed to repo
↓
Argo CD detects Git change
↓
Argo CD deploys new version on EKS
↓
Rollback possible with 1 click

````

---

## 🧰 Prerequisites

- ✅ :contentReference[oaicite:0]{index=0} (EKS) cluster
- ✅ :contentReference[oaicite:1]{index=1} server with:
  - AWS CLI
  - Docker
  - kubectl
  - Git
- ✅ :contentReference[oaicite:2]{index=2} (ECR) account
- ✅ :contentReference[oaicite:3]{index=3} installed on the cluster
- ✅ GitHub repository  
  👉 [https://github.com/nowshad13/SampleApp](https://github.com/nowshad13/SampleApp)

---

## ⚙️ Step 1: Install Argo CD on EKS

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
````

Check pods:

```bash
kubectl get pods -n argocd
```

Get admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Port-forward the ArgoCD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8082:443
```
```bash
 ssh -i "C:\Users\nowsh\.ssh\id_ed25519" -N -o ServerAliveInterval=60 -L 8082:localhost:8082 ubuntu@13.233.148.244
```

👉 Access dashboard: [https://localhost:8080](https://localhost:8080)
Username: `admin`
Password: *(from above)*

---

## 🪪 Step 2: Create ArgoCD Application

Go to Argo CD UI → **`+ NEW APP`**

| Field            | Value                                        |
| ---------------- | -------------------------------------------- |
| Application Name | `sampleapp`                                  |
| Project          | `default`                                    |
| Sync Policy      | `automated` ✅                                |
| Repository URL   | `https://github.com/nowshad13/SampleApp.git` |
| Revision         | `main`                                       |
| Path             | `k8s`                                        |
| Cluster          | `https://kubernetes.default.svc`             |
| Namespace        | `sampleapp`                                  |

Click **Create** → then **Sync** ✅

---

## 🐳 Step 3: ECR Setup

Create ECR repo if not created:

```bash
aws ecr create-repository --repository-name sampleapp
```

Login to ECR:

```bash
aws ecr get-login-password --region ap-south-1 | docker login \
  --username AWS --password-stdin 669443521868.dkr.ecr.ap-south-1.amazonaws.com
```

---

## ☸️ Step 4: Kubernetes Manifests (`/k8s`)

### `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sampleapp
```

---

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sampleapp
  namespace: sampleapp
  labels:
    app: sampleapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sampleapp
  template:
    metadata:
      labels:
        app: sampleapp
    spec:
      containers:
        - name: sampleapp
          image: 669443521868.dkr.ecr.ap-south-1.amazonaws.com/sampleapp:12
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
```

---

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sampleapp-service
  namespace: sampleapp
spec:
  type: LoadBalancer
  selector:
    app: sampleapp
  ports:
    - port: 80
      targetPort: 8080
```

Push these files to the GitHub repo.

---

## 🤖 Step 5: Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any
  
    environment {
        AWS_ACCOUNT_ID = '669443521868'
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'sampleapp'
        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/nowshad13/SampleApp.git'
            }
        }

        stage('Build & Push Image') {
            steps {
                sh '''
                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${IMAGE_URI}
                docker build -t ${IMAGE_URI}:${BUILD_NUMBER} .
                docker push ${IMAGE_URI}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Update K8s Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                    git config user.email "nowshad1304@gmail.com"
                    git config user.name "nowshad13"
                    git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/nowshad13/SampleApp.git

                    sed -i "s|sampleapp:.*|sampleapp:${BUILD_NUMBER}|" k8s/deployment.yaml

                    if git diff --quiet; then
                      echo "✅ No changes to commit. Skipping."
                    else
                      git add k8s/deployment.yaml
                      git commit -m "Deploy build ${BUILD_NUMBER}"
                      git push origin main
                    fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Build and Deployment successful ✅"
        }
        failure {
            echo "Deployment failed ❌"
        }
    }
}
```

> 💡 `github-cred` is a Jenkins **Username/Password** credential
> (username = your GitHub username, password = personal access token)

---

## 🔄 Step 6: Rollout & Rollback

Every time you run the Jenkins pipeline:

* A new image is built and pushed to ECR
* Jenkins updates the `deployment.yaml` image tag
* Git push triggers Argo CD to deploy automatically

Rollback options:

* 🖱️ In Argo CD UI → Application → **History and Rollback**
* ⏪ Or CLI:

```bash
kubectl -n sampleapp rollout undo deployment sampleapp
```

---

## 🧪 Step 7: Verify Deployment

Check pods:

```bash
kubectl get pods -n sampleapp
```

Check service:

```bash
kubectl get svc -n sampleapp
```

Get the `EXTERNAL-IP` of LoadBalancer and test in browser:

```
http://<EXTERNAL-IP>
```

---

## 🛡️ Step 8: Optional Enhancements

* ✅ Add blue-green or canary rollout strategies
* ✅ Expose Argo CD via Ingress/ALB for team access
* ✅ Add automated tests in Jenkins before deployment
* ✅ Add approval gates for production

---

## 🏁 Final Result

✅ Fully automated pipeline:

```
Jenkins → Build & Push → Git Update → ArgoCD Auto Sync → EKS Deploy
```

✅ Rollback with **1 click** in Argo CD.
✅ Real-time GitOps workflow just like industry-grade systems.

---

## 🧑‍💻 Credits

* AWS ECR & EKS
* Jenkins
* Argo CD
* GitHub
* Kubernetes

---

## 📝 License

This project is for educational and demonstration purposes.

```

---

Would you like me to **automatically generate and push this `README.md` file** to your GitHub repo via a single command (or Git push guide)? 😎📌
```

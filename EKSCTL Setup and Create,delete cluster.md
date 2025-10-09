# Powershell commands to install EKCTL

# 📥 Download eksctl
Invoke-WebRequest "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Windows_amd64.zip" -OutFile "eksctl.zip"

# 📂 Extract the binary
Expand-Archive -Path .\eksctl.zip -DestinationPath .

# 🏗 Create folder under Program Files (needs admin)
New-Item -ItemType Directory -Force -Path "C:\Program Files\Amazon\eksctl" | Out-Null

# 🚚 Move the binary to Program Files
Move-Item -Path .\eksctl.exe -Destination "C:\Program Files\Amazon\eksctl\eksctl.exe" -Force

# ➕ Add to PATH for the current session
$env:Path += ";C:\Program Files\Amazon\eksctl"

# ✅ Test installation
eksctl version


# Creating cluster using ekctl


```
eksctl create cluster \
--name DbApp \
--region ap-south-1 \
--version 1.31 \
--nodegroup-name dbnodegroup \
--node-type t3.small \
--nodes 2 \
--nodes-min 2 \
--nodes-max 2 \
--managed

```

# 🧪 After Cluster Creation, Verify It:

```
aws eks list-clusters
eksctl get cluster
kubectl get nodes
kubectl get pods -A
```


✅ You should see your worker nodes in Ready state.


# ⚡ Bonus: Quick Delete Command (if needed)

To clean everything created by eksctl:
```
eksctl delete cluster --name my-eks-cluster --region ap-south-1

```
This will:

Delete control plane

Node groups

ENIs created by EKS

VPC (if eksctl created it)

✅ Much cleaner than manual deletion.

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

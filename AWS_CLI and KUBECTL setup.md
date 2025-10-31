




# Kubectl setup

```bash
# 1️⃣ Download the latest kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 2️⃣ Verify the binary (optional but recommended)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# 3️⃣ Make it executable
chmod +x kubectl

# 4️⃣ Move it to your user’s local bin (no sudo required)
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/

# 5️⃣ Add ~/.local/bin to your PATH permanently
echo 'export PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc

# 6️⃣ (Optional) Make it system-wide
# sudo cp ~/.local/bin/kubectl /usr/local/bin/

# 7️⃣ Verify installation
kubectl version --client
```

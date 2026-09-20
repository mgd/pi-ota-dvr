# Pi 4B OTA DVR — k3s + ArgoCD GitOps Stack

TVHeadend (HDHomeRun tuner backend) + Plex (playback/UI), deployed to a
single-node k3s cluster on your Raspberry Pi 4B, managed via ArgoCD so all
future changes are just `git push`.

## 0. Prerequisites

- Raspberry Pi OS Lite (64-bit) on the 4B, updated
- NVMe drive in a USB 3.0 enclosure, connected to the Pi
- HDHomeRun on the same LAN, already receiving your antenna feed
- A git repo (GitHub, GitLab, self-hosted — anything ArgoCD can reach) to push
  this folder into

## 1. Mount the NVMe permanently

```bash
lsblk                          # find your device, e.g. /dev/sda1
sudo mkfs.ext4 /dev/sda1       # only if it's not already formatted
sudo mkdir -p /mnt/nvme
sudo blkid /dev/sda1           # copy the UUID
sudo nano /etc/fstab
# add this line (replace UUID):
# UUID=xxxx-xxxx  /mnt/nvme  ext4  defaults,noatime  0  2
sudo mount -a
sudo mkdir -p /mnt/nvme/tvheadend-config /mnt/nvme/recordings /mnt/nvme/plex-config
```

## 2. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
sudo cat /var/lib/rancher/k3s/server/node-token   # save if adding more nodes later
```

k3s ships with the `local-path` StorageClass already enabled — that's what
the PVCs here use, pointed at your NVMe mount via a hostPath-backed
provisioner path.

Point local-path at your NVMe instead of the default SD-card path:

```bash
sudo kubectl -n kube-system edit configmap local-path-config
# change "path" under "nodePathMap" to ["/mnt/nvme/k3s-local-path"]
sudo mkdir -p /mnt/nvme/k3s-local-path
sudo kubectl -n kube-system rollout restart deployment local-path-provisioner
```

## 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd patch svc argocd-server -p '{"spec": {"type": "NodePort"}}'
```

Get the initial admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Find the NodePort ArgoCD landed on and browse to `https://<pi-ip>:<port>`.

## 4. Push this repo, then register the root Application

```bash
cd pi-ota-dvr
git init && git add . && git commit -m "initial OTA DVR stack"
git remote add origin <your-repo-url>
git push -u origin main
```

Edit `bootstrap/root-app.yaml` — replace `REPLACE_WITH_YOUR_GIT_REPO_URL`
with your repo URL — then apply it:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

This is the "app of apps" pattern: this one Application tells ArgoCD to go
watch `apps/` in your repo and deploy everything it finds there
(TVHeadend, Plex). From now on, editing a YAML file and pushing is all you
need — ArgoCD syncs automatically.

## 5. Configure TVHeadend

Once the pod is running (`kubectl get pods -n media`):

- Browse to `http://<pi-ip>:9981`
- Configuration → DVB Inputs → Networks → it should auto-discover the
  HDHomeRun (hostNetwork mode is what makes this discovery work)
- Run a channel scan, confirm your local ABC/CBS/FOX/NBC affiliates show up
- Configuration → Users → set a password (it's wide open by default)
- Enable "HDHomeRun emulation" under Configuration → General → Base if you
  want Plex to see TVHeadend as a virtual HDHomeRun tuner (recommended —
  simplest way to get Plex DVR/Live TV working against it)

## 6. Configure Plex

- Browse to `http://<pi-ip>:32400/web`
- Sign in, then Settings → Live TV & DVR → it should detect the emulated
  HDHomeRun from TVHeadend
- Point recordings storage at `/recordings` (mounted from your NVMe PVC)

## Notes on the Pi 4B / k3s fit

- Single-node k3s on a 4B handles this workload comfortably — TVHeadend and
  Plex are both lightweight when Plex isn't transcoding. Avoid enabling
  Plex hardware transcoding assumptions; the 4B's ARM CPU will struggle if
  multiple clients need different bitrates simultaneously. Direct Play to
  your TV app avoids this entirely.
- `hostNetwork: true` is required for both containers (HDHomeRun discovery
  needs multicast/SSDP, which doesn't cross a container bridge network) —
  this means no per-pod IP isolation, but it's a single-node home cluster,
  so that's a non-issue.
- If you ever add more Pis to the cluster, add `nodeSelector` so these pods
  stay pinned to the 4B with the NVMe attached.

# GPU AMD (VAAPI / ROCm) sur Proxmox VE pour LXC

## Contexte & périmètre

Ce document résume **la configuration du GPU AMD Radeon RX 6700 XT** sur le serveur Proxmox VE et son exposition à plusieurs conteneurs LXC :

- **Ollama LXC** (CT `21211434`) – IA / LLM avec ROCm + Vulkan
- **Plex LXC** (CT `20232400`) – transcodage vidéo matériel via VAAPI
- **Jellyfin LXC** (CT `20308096`) – transcodage vidéo matériel via VAAPI
- **ComfyUI LXC** (CT `21608188`) – VAAPI fonctionnel (ROCm/PyTorch pour Stable Diff non couvert ici)

> ⚠️ **Important**
>
> - Ce README couvre **l’exposition du GPU** et les **tests VAAPI/ROCm** à l’intérieur des conteneurs.
> - La configuration applicative détaillée (UI Plex/Jellyfin/ComfyUI, modèles Ollama, workflows ComfyUI, etc.) n’est pas décrite ici.
> - Pour ComfyUI, seule la partie **VAAPI** est validée. La stack **ROCm + PyTorch** pour Stable Diff sera documentée séparément.

---

## 1. Prérequis côté Proxmox VE

### 1.1 Hôte Proxmox

- Proxmox VE 8.x avec noyau :

```bash
uname -a
Linux pve 6.8.12-16-pve ...
```

- GPU AMD visible :

```bash
lspci -nnk | grep -A2 'VGA compatible controller'
# ... AMD Radeon RX 6700 XT ...
# Kernel driver in use: amdgpu
```

- Périphériques GPU sur l’hôte :

```bash
ls -l /dev/kfd /dev/dri
crw-rw---- 1 root render 234, 0 ... /dev/kfd
total 0
drwxr-xr-x 2 root root         80 ... by-path
crw-rw---- 1 root video  226,   0 ... card0
crw-rw---- 1 root render 226, 128 ... renderD128
```

### 1.2 Groupes importants sur l’hôte

```bash
grep -w 'video\|render' /etc/group
video:x:44:
render:x:993:
```

Ces GIDs sont réutilisés dans les configs LXC pour mapper correctement les droits sur les devices.

---

## 2. Règles générales LXC

Pour tous les conteneurs utilisant le GPU :

1. **Ne pas utiliser `pct restart`**Toujours faire :

   ```bash
   pct stop <CTID>
   pct start <CTID>
   ```
2. LXC en mode **unprivileged** (`unprivileged: 1`) quand c’est supporté par le template (c’est le cas ici pour Ollama & ComfyUI).
3. Exposition du GPU via `devX:` dans la conf LXC, en réutilisant les GID de l’hôte :

   - `/dev/dri/renderD128` → GID **44** (groupe `video`)
   - `/dev/kfd` → GID **993** (groupe `render` sur l’hôte / `kvm` ou équivalent dans le LXC)

---

## 3. Ollama LXC – ROCm + Vulkan (CT 21211434)

### 3.1 Configuration LXC (`/etc/pve/lxc/21211434.conf`)

```ini
# ROCm / AMD GPU pour Ollama
arch: amd64
cmode: shell
cores: 8
memory: 32768
swap: 0
hostname: ollama-lxc
ostype: ubuntu
unprivileged: 1
nameserver: 192.168.1.3
searchdomain: z-server.me
net0: name=eth0,bridge=vmbr0,gw=192.168.1.1,hwaddr=BC:24:11:3D:C3:0A,ip=192.168.1.212/24,ip6=auto,type=veth
rootfs: vm-docker:vm-21211434-disk-0,size=200G
startup: order=15
tags: community-script;linux;lxc;marechal;remote;server;ssh;webserver

# GPU
dev0: /dev/kfd,gid=993,uid=0
dev1: /dev/dri/renderD128,gid=44
```

> 💡 Ici, on **n’utilise pas** de `lxc.mount.entry` pour `/dev/dri`, uniquement les lignes `dev0/dev1`.

### 3.2 Installation ROCm (dans le LXC)

Ollama LXC est basé sur **Ubuntu 24.04**, avec ROCm 6.2.4 :

```bash
# Dans le CT 21211434
rocminfo | grep -i gfx | head
# Name: gfx1030
# amdgcn-amd-amdhsa--gfx1030
```

Variables globales pour le GPU :

```bash
cat <<'EOF' >> /etc/environment
HSA_OVERRIDE_GFX_VERSION=10.3.0
ROCR_VISIBLE_DEVICES=0
EOF
```

Puis installation d’Ollama via le script officiel :

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 3.3 Activation de Vulkan et ROCm pour Ollama

Override systemd :

```bash
mkdir -p /etc/systemd/system/ollama.service.d

cat <<'EOF' >/etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="HSA_OVERRIDE_GFX_VERSION=10.3.0"
Environment="ROCR_VISIBLE_DEVICES=0"
EOF

systemctl daemon-reload
systemctl restart ollama
```

### 3.4 Vérifications

- GPU détecté par ROCm :

```bash
rocminfo | grep -i gfx | head
# gfx1030 ...
```

- Journal Ollama, détection ROCm :

```bash
journalctl -u ollama -n 40 --no-pager | grep -E 'inference compute|ROCm0'
# inference compute id=0 library=ROCm compute=gfx1030 name=ROCm0 description="AMD Radeon RX 6700 XT" ...
```

Ollama utilise alors le GPU via ROCm/Vulkan.

---

## 4. Plex LXC – VAAPI (CT 20232400)

### 4.1 Configuration LXC (`/etc/pve/lxc/20232400.conf`)

```ini
# Plex LXC – GPU AMD pour VAAPI
arch: amd64
cmode: shell
cores: 12
memory: 4096
swap: 2048
hostname: plex
ostype: ubuntu
unprivileged: 1
nameserver: 192.168.1.3
searchdomain: z-server.me
net0: name=eth0,bridge=vmbr0,gw=192.168.1.1,hwaddr=BC:24:11:2C:BF:A3,ip=192.168.1.202/24,type=veth
rootfs: vm-docker:vm-20232400-disk-0,size=300G
startup: order=4
tags: community-script;linux;lxc;marechal;media;server;share;ssh;webserver

# GPU VAAPI
dev0: /dev/dri/renderD128,gid=44
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

### 4.2 Groupes dans le conteneur

```bash
grep -w 'video\|render' /etc/group
# video:x:44:plex
# render:x:118:plex
```

> Vérifier que l’utilisateur **`plex`** est bien dans `video` **et** `render`.

### 4.3 Paquets VAAPI + test FFmpeg

```bash
apt update
apt install -y vainfo mesa-va-drivers ffmpeg
export LIBVA_DRIVER_NAME=radeonsi

vainfo | egrep 'Driver version|VAProfile'
# Driver version: Mesa Gallium ... for AMD Radeon RX 6700 XT ...
# VAProfileH264* / HEVC* / VP9* / AV1* ...
```

Test d’encodage matériel :

```bash
ffmpeg -v verbose \
  -init_hw_device vaapi=va:/dev/dri/renderD128 \
  -filter_hw_device va \
  -f lavfi -i testsrc2=size=1920x1080:rate=30 \
  -t 5 \
  -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi \
  -f null -
```

Si le test passe, VAAPI est fonctionnel pour Plex (à activer ensuite dans l’UI Plex).

---

## 5. Jellyfin LXC – VAAPI (CT 20308096)

### 5.1 Configuration LXC (`/etc/pve/lxc/20308096.conf`)

```ini
# Jellyfin LXC – GPU AMD pour VAAPI
arch: amd64
cmode: shell
cores: 12
memory: 6144
swap: 2048
hostname: jellyfin
ostype: ubuntu
unprivileged: 1
nameserver: 192.168.1.3
searchdomain: z-server.me
net0: name=eth0,bridge=vmbr0,gw=192.168.1.1,hwaddr=BC:24:11:E7:9C:CC,ip=192.168.1.203/24,type=veth
rootfs: vm-docker:vm-20308096-disk-0,size=100G
startup: order=5
tags: community-script;linux;lxc;marechal;media;server;share;ssh;webserver

# GPU VAAPI
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

(Ici on n’utilise que `/dev/dri` pour VAAPI, pas `/dev/kfd`.)

### 5.2 Groupes dans le conteneur

```bash
grep -w 'video\|render' /etc/group
# video:x:44:jellyfin
# render:x:104:jellyfin
```

### 5.3 Paquets VAAPI + test FFmpeg

Même principe que Plex :

```bash
apt update
apt install -y vainfo mesa-va-drivers ffmpeg
export LIBVA_DRIVER_NAME=radeonsi

vainfo | egrep 'Driver version|VAProfile'
ffmpeg -v verbose \
  -init_hw_device vaapi=va:/dev/dri/renderD128 \
  -filter_hw_device va \
  -f lavfi -i testsrc2=size=1920x1080:rate=30 \
  -t 5 \
  -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi \
  -f null -
```

Une fois validé, on active VAAPI dans l’UI Jellyfin (section **Lecture / Transcodage**).

---

## 6. ComfyUI LXC – VAAPI (CT 21608188)

### 6.1 Configuration LXC (`/etc/pve/lxc/21608188.conf`)

```ini
# ComfyUI LXC – GPU AMD (VAAPI)
arch: amd64
cmode: shell
cores: 4
memory: 32768
swap: 512
hostname: comfyui
ostype: debian
unprivileged: 1
nameserver: 192.168.1.3
searchdomain: z-server.me
net0: name=eth0,bridge=vmbr0,gw=192.168.1.1,hwaddr=BC:24:11:40:40:37,ip=192.168.1.216/24,type=veth
rootfs: vm-docker:vm-21608188-disk-0,size=100G
startup: order=15
tags: ai;community-script

# GPU
dev0: /dev/kfd,gid=993,uid=0
dev1: /dev/dri/renderD128,gid=44
```

> On réplique ici le même principe que pour Ollama (ROCm possible plus tard), mais on valide surtout **VAAPI**.

### 6.2 Groupes & devices dans le LXC

```bash
ls -l /dev/kfd /dev/dri/renderD128
# /dev/dri/renderD128 → root video
# /dev/kfd          → root kvm (ou groupe équivalent)

grep -w 'video\|render' /etc/group
# video:x:44:root
# render:x:992:root
```

### 6.3 Paquets VAAPI + test FFmpeg

```bash
apt update
apt install -y vainfo mesa-va-drivers mesa-vulkan-drivers ffmpeg
export LIBVA_DRIVER_NAME=radeonsi

vainfo --display drm --device /dev/dri/renderD128 \
  | egrep 'Driver version|VAProfile'
# Driver version: Mesa Gallium driver 25.0.7-2 for AMD Radeon RX 6700 XT ...
```

Test encode :

```bash
ffmpeg -v verbose \
  -init_hw_device vaapi=va:/dev/dri/renderD128 \
  -filter_hw_device va \
  -f lavfi -i testsrc2=size=1920x1080:rate=30 \
  -t 5 \
  -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi \
  -f null -
```

Si ce test passe, l’accélération VAAPI fonctionne et ComfyUI peut en tirer parti pour certaines opérations vidéo (selon les nodes utilisés).

---

## 7. Récapitulatif rapide

|       CT | Service  | OS     | GPU exposé                   | Usage principal        | Test de validation                     |
| -------: | -------- | ------ | ----------------------------- | ---------------------- | -------------------------------------- |
| 21211434 | Ollama   | Ubuntu | `/dev/kfd` + `renderD128` | LLM via ROCm/Vulkan    | `rocminfo`, `journalctl -u ollama` |
| 20232400 | Plex     | Ubuntu | `/dev/dri` (VAAPI)          | Transcodage vidéo     | `vainfo` + `ffmpeg h264_vaapi`     |
| 20308096 | Jellyfin | Ubuntu | `/dev/dri` (VAAPI)          | Transcodage vidéo     | `vainfo` + `ffmpeg h264_vaapi`     |
| 21608188 | ComfyUI  | Debian | `/dev/kfd` + `renderD128` | IA / vidéo (VAAPI ok) | `vainfo` + `ffmpeg h264_vaapi`     |

Ce README sert de **référence unique** pour l’exposition du GPU AMD RX 6700 XT aux conteneurs LXC sur Proxmox VE.
Toute nouvelle LXC GPU (autre service) pourra se baser sur ces modèles de configuration.

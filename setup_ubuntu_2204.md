```bash
#Install Ubuntu 22.04 or 22.10 then..

#Basic Tools (optional)
apt-get install curl wget vim
#Inspection Tools (optional)
apt-get install htop iotop iftop lm-sensors 
#Network/Disk diagnoses tools (optional)
apt-get install iputils-ping net-tools mtr-tiny locate ncdu
#Disk/Binary debugging tools (optional)
apt-get install smartmontools linux-tools-common linux-tools-generic strace

#GPUX Dependencies
apt-get install ca-certificates file git zip

#install podman + all dependencies
apt-get install podman
#upgrade podman >=4.2.0
wget http://ftp.us.debian.org/debian/pool/main/libp/libpod/podman_4.2.0+ds1-3_amd64.deb
dpkg -i podman_4.2.0+ds1-3_amd64.deb
rm podman_4.2.0+ds1-3_amd64.deb

#add a regular user (no sudo)
#deluser --remove-home user
useradd -m -s /bin/bash user
loginctl enable-linger user

#allow rootless podman CPU quota for user
mkdir -p /etc/systemd/system/user@.service.d/
cat <<EOT >> /etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=memory pids io cpu cpuset
EOT

#block lan access (recommended if node on LAN without smartswitch)
apt-get install iptables-persistent
cat <<EOT > /etc/iptables/rules.v4
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A OUTPUT -m state --state ESTABLISHED -j ACCEPT
-A OUTPUT -d 10.0.0.0/8 -j DROP
-A OUTPUT -d 172.16.0.0/12 -j DROP
-A OUTPUT -d 192.168.0.0/16 -j DROP
COMMIT
EOT

#config system files for higher performance
#file handles
cat <<EOT > /etc/security/limits.conf
root hard nofile 1048576
root soft nofile 1048576
* hard nofile 1048576
* soft nofile 1048576
root hard nproc unlimited
root soft nproc unlimited
* hard nproc unlimited
* soft nproc unlimited
root hard memlock unlimited
root soft memlock unlimited
* hard memlock unlimited
* soft memlock unlimited
EOT

#systemctl
cat <<EOT > /etc/sysctl.conf
net.ipv4.ip_unprivileged_port_start = 22
net.core.netdev_max_backlog = 307200

net.ipv4.tcp_rmem = 8192 262144 536870912
net.ipv4.tcp_wmem = 4096 16384 536870912
net.ipv4.tcp_adv_win_scale = -2
net.ipv4.tcp_notsent_lowat = 131072
EOT

#systemd
cat <<EOT > /etc/systemd/user.conf
[Manager]
DefaultTasksMax=infinity
DefaultLimitNOFILE=infinity
DefaultLimitNPROC=infinity
DefaultLimitMEMLOCK=infinity
DefaultLimitLOCKS=infinity
EOT
cat <<EOT > /etc/systemd/system.conf
[Manager]
DefaultTasksMax=infinity
DefaultLimitNOFILE=infinity
DefaultLimitNPROC=infinity
DefaultLimitMEMLOCK=infinity
DefaultLimitLOCKS=infinity
EOT


#install nvidia driver + CUDA + container runtime(if have NVIDIA GPUs)
apt-get install --no-install-recommends nvidia-driver-515

wget https://developer.download.nvidia.com/compute/cuda/11.7.1/local_installers/cuda_11.7.1_515.65.01_linux.run
sh cuda_11.7.1_515.65.01_linux.run --silent --toolkit --no-drm --no-man-page --override
rm cuda_11.7.1_515.65.01_linux.run
echo "PATH=\"\$PATH:/usr/local/cuda-11.7/bin\"" > /etc/environment && \
echo "CUDA_HOME=\"/usr/local/cuda-11.7\"" >> /etc/environment && \
echo "CUDA_PATH=\"/usr/local/cuda-11.7\"" >> /etc/environment

#install nvidia-container-runtime + setup OCI hook
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | apt-key add -
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | apt-key add -
mkdir -p /etc/apt/sources.list.d/
cat <<EOT >> /etc/apt/sources.list.d/nvidia-container-runtime.list
deb https://nvidia.github.io/libnvidia-container/experimental/ubuntu22.04/\$(ARCH) /
deb https://nvidia.github.io/nvidia-container-runtime/experimental/ubuntu22.04/\$(ARCH) /
deb https://nvidia.github.io/nvidia-docker/ubuntu22.04/\$(ARCH) /
EOT
apt-get update
apt-get install -y nvidia-container-runtime
mkdir -p /usr/share/containers/oci/hooks.d
cat <<EOT >> /usr/share/containers/oci/hooks.d/oci-nvidia-hook.json
{
    "version": "1.0.0",
    "hook": {
        "path": "/usr/bin/nvidia-container-toolkit",
        "args": ["nvidia-container-toolkit", "prestart"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ]
    },
    "when": {
        "always": true,
        "commands": [".*"]
    },
    "stages": ["prestart"]
}
EOT
#TODO: remove this once cgroupsV2 support is stable (likely the next major release)
sed -i 's/^#no-cgroups = false/no-cgroups = true/;' /etc/nvidia-container-runtime/config.toml
```

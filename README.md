# farm_image
Provisioning instances of farm

```elixir
drive = "sdb"
hostname = "node1"
{block_size,0} = System.shell("cat /sys/block/#{drive}/queue/physical_block_size")
block_size = :erlang.binary_to_integer(String.trim(block_size))
{io_size,0} = System.shell("cat /sys/block/#{drive}/queue/optimal_io_size")
io_size = :erlang.binary_to_integer(String.trim(io_size))
{align,0} = System.shell("cat /sys/block/#{drive}/alignment_offset")
align = :erlang.binary_to_integer(String.trim(align))

sector = trunc(io_size / block_size)
first_sector = trunc((align + io_size) / block_size)
size_boot = trunc(trunc((1024*1024*1024) / io_size) * sector)

{"",0} = System.shell("parted -s /dev/#{drive} mklabel gpt", [stderr_to_stdout: true])
{"",0} = System.shell("parted -s -a optimal /dev/#{drive} mkpart primary #{first_sector}s #{first_sector+size_boot}s", [stderr_to_stdout: true])
{"",0} = System.shell("parted -s -a optimal /dev/#{drive} mkpart primary #{first_sector+sector+size_boot}s 100%", [stderr_to_stdout: true])
{"",0} = System.shell("parted -s /dev/#{drive} set 1 esp on", [stderr_to_stdout: true])
{"",0} = System.shell("parted -s /dev/#{drive} set 1 boot on", [stderr_to_stdout: true])

{_,0} = System.shell("mkfs.vfat /dev/#{drive}1", [stderr_to_stdout: true])
{_,0} = System.shell("mkfs.btrfs -f -R free-space-tree /dev/#{drive}2", [stderr_to_stdout: true])

{_,0} = System.shell("mount -o discard=async,space_cache=v2,compress-force=zstd:2,ssd,noatime /dev/#{drive}2 /mnt", [stderr_to_stdout: true])
{_,0} = System.shell("tar --same-owner -xf jammy-base-amd64.tar.gz -C /mnt", [stderr_to_stdout: true])
File.mkdir_p!("/mnt/boot/efi")
{_,0} = System.shell("mount -o umask=0077 /dev/#{drive}1 /mnt/boot/efi", [stderr_to_stdout: true])
#File.rm!("/mnt/boot/lost+found")

#write fstab
{uuid_vfat,0} = System.shell("blkid -s UUID -o value /dev/#{drive}1", [stderr_to_stdout: true])
uuid_vfat = String.trim(uuid_vfat)
{uuid_btrfs,0} = System.shell("blkid -s UUID -o value /dev/#{drive}2", [stderr_to_stdout: true])
uuid_btrfs = String.trim(uuid_btrfs)

proc = "proc /proc proc defaults 0 0"
btrfs = "UUID=#{uuid_btrfs} / btrfs defaults,discard=async,space_cache=v2,compress-force=zstd:2,ssd,noatime 0 0"
vfat = "UUID=#{uuid_vfat} /boot/efi vfat umask=0077 0 0"
File.write!("/mnt/etc/fstab", "#{proc}\n#{btrfs}\n#{vfat}")

#write misc
locale = """
LC_CTYPE="en_US.UTF-8"
LC_ALL="en_US.UTF-8"
LANG="en_US.UTF-8"
LANGUAGE="en_US.UTF-8"
"""
File.write!("/mnt/etc/default/locale", String.trim(locale))

hosts = """
127.0.0.1 localhost
127.0.0.1 #{hostname}
"""
File.write!("/mnt/etc/hosts", String.trim(hosts))
File.write!("/mnt/etc/hostname", hostname)

#install via systemd-nspawn
nspawn = """
sudo systemd-nspawn --pipe --capability=CAP_SYS_ADMIN,CAP_SYS_RAWIO --bind=/dev/sdb1 --bind=/dev/sdb2 --bind=/dev/sdb -D /mnt/ /bin/bash << EOF
apt-get update
apt-get -y dist-upgrade
#add a regular user (no sudo)
useradd -m -s /bin/bash user
#install basics + timezone + locales
apt-get install -y dialog apt-utils debconf-utils
DEBIAN_FRONTEND=noninteractive TZ=Etc/UTC apt-get install -y tzdata
apt-get install -y locales
locale-gen en_US.UTF-8
DEBIAN_FRONTEND=noninteractive apt-get install -y keyboard-configuration console-setup
#install systemd
apt-get install -y init
#Install kernel + grub
apt-get install -y linux-{headers,image}-generic
apt-get install -y grub-efi
apt-get install -y initramfs
apt-get install -y initramfs-tools btrfs-progs
#Install generic apps
apt-get install -y --no-install-recommends ca-certificates nano vim git wget curl zip ncdu iftop iotop htop \
net-tools locate lm-sensors mtr-tiny openssh-server python-is-python3 \
smartmontools linux-tools-common linux-tools-generic fdisk iputils-ping strace screen
#Install netplan + iptables persistent
DEBIAN_FRONTEND=noninteractive apt-get install -y netplan.io iptables-persistent
cat <<EOT > /etc/netplan/config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enwild:
      match:
        name: enp*
      dhcp4: true
EOT
#block lan access
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
#Configure grub
cat <<EOT > /etc/default/grub
GRUB_DEFAULT=1
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=0 nomodeset transparent_hugepage=never"
GRUB_CMDLINE_LINUX=""
GRUB_GFXPAYLOAD_LINUX="text"
EOT
#install openssh-server
apt-get install -y openssh-server
#install podman + nvidia
apt-get install -y podman
apt-get install -y --no-install-recommends nvidia-driver-510
wget https://developer.download.nvidia.com/compute/cuda/11.6.2/local_installers/cuda_11.6.2_510.47.03_linux.run
sh cuda_11.6.2_510.47.03_linux.run --silent --toolkit --no-drm --no-man-page
rm cuda_11.6.2_510.47.03_linux.run
echo "PATH=\"\$PATH:/usr/local/cuda-11.6/bin\"" > /etc/environment && \
echo "CUDA_HOME=\"/usr/local/cuda-11.6\"" >> /etc/environment && \
echo "CUDA_PATH=\"/usr/local/cuda-11.6\"" >> /etc/environment
#install nvidia-container-runtime + setup OCI hook
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | apt-key add -
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | apt-key add -
mkdir -p /etc/apt/sources.list.d/
cat <<EOT >> /etc/apt/sources.list.d/nvidia-container-runtime.list
deb https://nvidia.github.io/libnvidia-container/experimental/ubuntu18.04/\\\$(ARCH) /
deb https://nvidia.github.io/nvidia-container-runtime/experimental/ubuntu18.04/\\\$(ARCH) /
deb https://nvidia.github.io/nvidia-docker/ubuntu18.04/\\\$(ARCH) /
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
#allow rootless podman CPU quota
mkdir -p /etc/systemd/system/user@.service.d/
cat <<EOT >> /etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=memory pids io cpu cpuset
EOT
#set podman storage driver as btrfs
#cat <<EOT >> /etc/containers/storage.conf
#[storage]
#driver = "btrfs"
#EOT
#add linger
touch /var/lib/systemd/linger/user
#grub
update-grub
grub-install --removable --no-floppy --no-nvram --no-uefi-secure-boot --target=x86_64-efi --efi-directory="/boot/efi" --recheck
source /etc/default/locale && update-initramfs -c -u -k all
 history -c
EOF
"""

#write sysconfig
limits = """
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
"""
File.write!("/mnt/etc/security/limits.conf", String.trim(limits))

sysctl = """
net.ipv4.ip_unprivileged_port_start = 22
net.core.netdev_max_backlog = 307200

net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.ipv4.tcp_rmem = 4096 1048576 8388608
"""
File.write!("/mnt/etc/sysctl.conf", String.trim(sysctl))

sysd = """
[Manager]
DefaultTasksMax=infinity
DefaultLimitNOFILE=infinity
DefaultLimitNPROC=infinity
DefaultLimitMEMLOCK=infinity
DefaultLimitLOCKS=infinity
"""
File.write!("/mnt/etc/systemd/user.conf", String.trim(sysd))
File.write!("/mnt/etc/systemd/system.conf", String.trim(sysd))

#fix MOTD
File.rm!("/mnt/etc/update-motd.d/10-help-text")
File.rm!("/mnt/etc/update-motd.d/60-unminimize")
gpux = """
#!/bin/sh
printf "\\n\\n"
printf "  __________________ ____ _______  ___"
printf " /  _____/\\______   \\    |   \\   \\/  /"
printf "/   \\  ___ |     ___/    |   /\\     / "
printf "\\    \\_\\  \\|    |   |    |  / /     \\ "
printf " \\______  /|____|   |______/ /___/\\  \\"
printf "        \\/                         \\_/"
printf "\\n\\n"
"""
File.write!("/mnt/etc/update-motd.d/10-gpux", String.trim(gpux))

#dont forgot to replace instances in /boot/grub/grub.cfg of /dev/sdaX with UUID

#add authorized_keys
File.mkdir_p!("/mnt/root/.ssh")
File.touch!("/mnt/root/.ssh/authorized_keys")
{"", 0} = System.shell("chown -R root:root /mnt/root/.ssh") 
File.mkdir_p!("/mnt/home/user/.ssh")
File.touch!("/mnt/home/user/.ssh/authorized_keys")
{"", 0} = System.shell("chown -R user:user /mnt/home/user/.ssh") 

#umount
{_,0} = System.shell("umount /mnt/boot/efi", [stderr_to_stdout: true])
{_,0} = System.shell("umount /mnt", [stderr_to_stdout: true])

```

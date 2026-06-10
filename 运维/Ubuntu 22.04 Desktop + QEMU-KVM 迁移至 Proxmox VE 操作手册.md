# Ubuntu 22.04 Desktop + QEMU/KVM 迁移至 Proxmox VE 操作手册

> 适用场景：当前物理服务器运行 Ubuntu 22.04 Desktop + QEMU/KVM/libvirt，多台虚拟机需要迁移到全新 Proxmox VE 平台。  
> 重点虚拟机：`win11-2`，Windows 11 Pro，提供远程桌面连接，当前存在卡死问题。  
> 说明：Proxmox VE 应安装在 Debian/Proxmox 官方环境上，不建议、也不支持从 Ubuntu Desktop 直接“原地改造成 Proxmox VE”。推荐做法是：备份现有虚拟机 → 安装 Proxmox VE → 导入虚拟机磁盘 → 重建虚拟机配置 → 验证业务。

---

## 一、 迁移目标

1. 将现有 Ubuntu/KVM 上的所有虚拟机迁移到 Proxmox VE。
2. 保留虚拟机系统盘、数据盘、MAC 地址、网络配置和关键业务数据。
3. 尽量降低 `win11-2` 卡死导致的数据损坏风险。
4. 在 Proxmox VE 上完成：
   - VM 管理
   - 快照/备份
   - 监控
   - 自动启动
   - 后续扩展和维护

---

## 二、 重要风险说明

### 2.1 不建议原地安装 Proxmox VE

当前系统是 Ubuntu 22.04 Desktop，Proxmox VE 官方环境基于 Debian。不要尝试在 Ubuntu 上直接安装 Proxmox VE 软件包作为生产方案。

推荐两种迁移方式：

| 方案 | 说明 | 推荐程度 |
| --- | --- | --- |
| 方案 A：新服务器安装 Proxmox VE 后迁移 | 最安全，可并行验证 | 强烈推荐 |
| 方案 B：同一台服务器备份后重装 Proxmox VE | 需要完整停机和可靠备份 | 可行 |
| 方案 C：Ubuntu 原地改造为 Proxmox VE | 风险高，不推荐 | 不推荐 |

---

## 三、 迁移前准备

### 3.1 准备外部备份介质

至少准备一个容量足够的外部磁盘、NAS、NFS、SMB 或临时服务器。

建议备份内容包括：

```bash
/var/lib/libvirt/images/
/etc/libvirt/qemu/
/etc/netplan/
/etc/libvirt/
/etc/qemu/
/etc/fstab
````

如果虚拟机磁盘不在 `/var/lib/libvirt/images/`，需要先找出真实路径。

---

### 3.2 查看当前虚拟机列表

在 Ubuntu 宿主机执行：

```bash
virsh list --all
```

示例输出：

```text
 Id   Name       State
--------------------------
 1    win11-2    running
 -    ubuntu-1   shut off
 -    debian-1   shut off
```

记录所有虚拟机名称。

---

### 3.3 导出现有虚拟机 XML 配置

创建备份目录：

```bash
sudo mkdir -p /backup/libvirt-xml
sudo mkdir -p /backup/vm-disks
```

导出所有虚拟机 XML：

```bash
for vm in $(virsh list --all --name); do
  sudo virsh dumpxml "$vm" > "/backup/libvirt-xml/${vm}.xml"
done
```

确认导出成功：

```bash
ls -lh /backup/libvirt-xml/
```

---

### 3.4 查看每台虚拟机磁盘路径

逐台查看：

```bash
virsh domblklist win11-2 --details
```

示例输出：

```text
Type   Device   Target   Source
------------------------------------------------
file   disk     vda      /var/lib/libvirt/images/win11-2.qcow2
file   cdrom    sda      -
```

也可以批量查看：

```bash
for vm in $(virsh list --all --name); do
  echo "===== $vm ====="
  virsh domblklist "$vm" --details
done
```

记录每台 VM 的：

- 系统盘路径

- 数据盘路径

- 磁盘格式：qcow2 / raw / lvm

- 网卡 MAC 地址

- CPU、内存、磁盘控制器类型

- 是否使用 UEFI

- 是否使用 TPM

- 是否使用 PCI 直通、USB 直通、显卡直通

---

### 3.5 查看虚拟机磁盘格式

例如：

```bash
qemu-img info /var/lib/libvirt/images/win11-2.qcow2
```

示例：

```text
file format: qcow2
virtual size: 200 GiB
disk size: 95 GiB
```

---

## 四、 重点处理 win11-2 卡死问题

`win11-2` 当前存在卡死，迁移前不要在运行状态下直接复制磁盘，否则容易得到不一致的磁盘镜像。

### 4.1 优先尝试正常关机

```bash
virsh shutdown win11-2
```

等待系统关闭后确认：

```bash
virsh list --all
```

如果长时间无法关机，再执行强制关闭：

```bash
virsh destroy win11-2
```

注意：`virsh destroy` 等价于强制断电，可能导致 Windows 文件系统损坏。执行后建议在迁移前做一次磁盘检查或至少保留原始备份。

---

### 4.2 建议迁移前在 Windows 内执行的操作

如果 `win11-2` 还能进入系统，建议先完成：

1. 安装或更新 VirtIO 驱动。

2. 安装 QEMU Guest Agent。

3. 关闭快速启动。

4. 设置固定 IP 或记录当前 IP。

5. 记录网卡 MAC 地址。

6. 执行磁盘检查。

Windows 内以管理员执行：

```cmd
chkdsk C: /scan
```

关闭快速启动：

```cmd
powercfg /h off
```

---

### 4.3 RDP Wrapper 风险提示

`RDP Wrapper` 不是微软官方支持的并发远程桌面方案，Windows 更新、termsrv.dll 变化、补丁不兼容、授权问题都可能导致异常。

如果该虚拟机承担多人并发桌面业务，建议后续改为：

- Windows Server + Remote Desktop Services

- 正规 RDS CAL 授权

- 或其他 VDI / 远程应用方案

迁移到 Proxmox VE 可以改善虚拟化管理、备份和监控，但不能从根本上保证 RDP Wrapper 稳定。

---

## 五、 迁移前完整备份

### 5.1 停止所有虚拟机

```bash
for vm in $(virsh list --name); do
  sudo virsh shutdown "$vm"
done
```

确认全部关闭：

```bash
virsh list --all
```

如有无法关闭的 VM：

```bash
sudo virsh destroy VM名称
```

---

### 5.2 复制虚拟机磁盘

假设虚拟机磁盘在 `/var/lib/libvirt/images/`：

```bash
sudo rsync -aHAXS --numeric-ids --info=progress2 \
  /var/lib/libvirt/images/ \
  /backup/vm-disks/
```

如果磁盘分散在多个目录，需要逐个复制。

例如：

```bash
sudo rsync -aHAXS --numeric-ids --info=progress2 \
  /data/kvm-images/ \
  /backup/vm-disks/data-kvm-images/
```

---

### 5.3 复制 libvirt 配置

```bash
sudo rsync -aHAXS --numeric-ids /etc/libvirt/ /backup/etc-libvirt/
sudo rsync -aHAXS --numeric-ids /etc/netplan/ /backup/etc-netplan/
```

---

### 5.4 校验备份文件

生成校验文件：

```bash
cd /backup
sudo find vm-disks -type f -exec sha256sum {} \; > SHA256SUMS.txt
```

后续可校验：

```bash
cd /backup
sha256sum -c SHA256SUMS.txt
```

---

## 六、 安装 Proxmox VE

### 6.1 下载 Proxmox VE ISO

从 Proxmox 官方网站下载当前最新 Proxmox VE ISO。

由于当前环境无法联网确认最新小版本，建议实际操作时以官网最新 ISO 为准。

---

### 6.2 安装建议

安装时建议：

|项目|建议|
|---|---|
|文件系统|ZFS 或 ext4 + LVM-thin|
|单盘服务器|ext4 + LVM-thin 较简单|
|多盘服务器|ZFS mirror / raidz|
|系统盘|尽量使用 SSD|
|VM 存储盘|SSD / NVMe 优先|
|网卡|配置 Linux Bridge：`vmbr0`|
|管理 IP|使用固定 IP|
|主机名|例如 `pve01.example.local`|

---

### 6.3 安装后的基础更新

登录 Proxmox VE Shell：

```bash
apt update
apt full-upgrade -y
reboot
```

查看版本：

```bash
pveversion -v
```

---

### 6.4 配置无订阅源

如果没有企业订阅，使用 no-subscription 源。

编辑：

```bash
nano /etc/apt/sources.list
```

确保包含类似内容，具体版本代号以当前 Proxmox VE 对应 Debian 版本为准：

```text
deb http://deb.debian.org/debian bookworm main contrib
deb http://deb.debian.org/debian bookworm-updates main contrib
deb http://security.debian.org/debian-security bookworm-security main contrib
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

禁用企业源：

```bash
sed -i 's/^deb/#deb/g' /etc/apt/sources.list.d/pve-enterprise.list 2>/dev/null || true
```

更新：

```bash
apt update
```

---

## 七、 Proxmox VE 网络规划

### 7.1 推荐网络结构

如果虚拟机需要被局域网直接访问，推荐桥接模式：

```text
物理网卡 eno1
   |
   +-- vmbr0
          |
          +-- Proxmox 管理 IP
          +-- VM1
          +-- VM2
          +-- win11-2
```

典型 `/etc/network/interfaces`：

```text
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

应用网络配置：

```bash
ifreload -a
```

如果没有 `ifreload`：

```bash
apt install ifupdown2 -y
ifreload -a
```

---

### 7.2 保留原 VM MAC 地址

迁移时建议保留原虚拟机 MAC 地址，尤其是 Windows VM。这样可以减少：

- DHCP 地址变化

- Windows 激活变化

- 应用授权绑定变化

- RDP 访问地址变化

从 XML 中查看 MAC：

```bash
grep -i "mac address" /backup/libvirt-xml/win11-2.xml
```

示例：

```xml
<mac address='52:54:00:12:34:56'/>
```

在 Proxmox VE 创建 VM 时指定：

```bash
qm set 101 --net0 virtio=52:54:00:12:34:56,bridge=vmbr0
```

---

## 八、 将备份数据传到 Proxmox VE

### 8.1 使用 rsync 从旧 Ubuntu 拷贝到 Proxmox

在 Proxmox VE 上创建目录：

```bash
mkdir -p /mnt/migration/libvirt-xml
mkdir -p /mnt/migration/vm-disks
```

从旧服务器拉取：

```bash
rsync -aHAXS --numeric-ids --info=progress2 \
  root@旧服务器IP:/backup/libvirt-xml/ \
  /mnt/migration/libvirt-xml/

rsync -aHAXS --numeric-ids --info=progress2 \
  root@旧服务器IP:/backup/vm-disks/ \
  /mnt/migration/vm-disks/
```

如果是同一台服务器重装，则从外置硬盘或 NAS 挂载后复制。

---

## 九、 Proxmox VE 中创建虚拟机

迁移原则：

1. 不直接导入 libvirt XML。

2. 在 Proxmox VE 中重新创建 VM 配置。

3. 导入原磁盘。

4. 按原 VM 的 CPU、内存、BIOS、UEFI、TPM、网卡重新设置。

5. 首次启动前确认启动顺序。

---

## 十、 迁移 Windows 11 虚拟机 win11-2

以下示例假设：

|项目|值|
|---|---|
|VM 名称|`win11-2`|
|Proxmox VMID|`101`|
|原磁盘|`/mnt/migration/vm-disks/win11-2.qcow2`|
|存储池|`local-lvm`|
|网桥|`vmbr0`|
|MAC|`52:54:00:12:34:56`|

请根据实际情况修改。

---

### 10.1 创建空 VM

```bash
qm create 101 \
  --name win11-2 \
  --memory 16384 \
  --cores 8 \
  --cpu host \
  --machine q35 \
  --bios ovmf \
  --scsihw virtio-scsi-single \
  --net0 virtio=52:54:00:12:34:56,bridge=vmbr0 \
  --agent enabled=1
```

说明：

|参数|说明|
|---|---|
|`--machine q35`|Windows 11 推荐|
|`--bios ovmf`|UEFI|
|`--cpu host`|使用宿主机 CPU 特性|
|`virtio-scsi-single`|推荐的 VirtIO SCSI 控制器|
|`--agent enabled=1`|启用 QEMU Guest Agent|

---

### 10.2 添加 EFI Disk

```bash
qm set 101 --efidisk0 local-lvm:0,efitype=4m,pre-enrolled-keys=1
```

如果原 Windows 11 使用 Secure Boot，可以保留 `pre-enrolled-keys=1`。  
如果启动异常，可以后续在 Proxmox GUI 中关闭 Secure Boot 相关选项测试。

---

### 10.3 添加 TPM 2.0

Windows 11 通常需要 TPM 2.0：

```bash
qm set 101 --tpmstate0 local-lvm:0,version=v2.0
```

---

### 10.4 导入原 Windows 磁盘

查看磁盘信息：

```bash
qemu-img info /mnt/migration/vm-disks/win11-2.qcow2
```

导入到 Proxmox 存储：

```bash
qm importdisk 101 /mnt/migration/vm-disks/win11-2.qcow2 local-lvm --format raw
```

导入后查看 VM 配置：

```bash
qm config 101
```

你会看到类似：

```text
unused0: local-lvm:vm-101-disk-0
```

将导入的磁盘挂载为系统盘：

```bash
qm set 101 --scsi0 local-lvm:vm-101-disk-0,discard=on,iothread=1
```

如果实际显示为 `vm-101-disk-1`，则使用实际名称。

---

### 10.5 设置启动顺序

```bash
qm set 101 --boot order=scsi0
```

---

### 10.6 挂载 VirtIO 驱动 ISO

如果 Windows 原本已经使用 VirtIO，通常可以直接启动。

如果不确定，建议挂载 VirtIO ISO：

```bash
qm set 101 --ide2 local:iso/virtio-win.iso,media=cdrom
```

如果 Proxmox 里还没有 `virtio-win.iso`，需要先下载并上传到：

```text
local -> ISO Images
```

---

### 10.7 首次启动 win11-2

```bash
qm start 101
```

打开控制台：

```bash
qm terminal 101
```

或在 Web UI 中进入：

```text
VM 101 -> Console
```

---

### 10.8 Windows 首次启动后检查

进入 Windows 后执行：

1. 检查设备管理器是否有未知设备。

2. 安装或修复 VirtIO 驱动。

3. 安装 QEMU Guest Agent。

4. 检查 IP 地址。

5. 检查远程桌面服务。

6. 检查 Windows 激活状态。

7. 检查事件查看器。

Windows 内确认 Guest Agent 服务：

```cmd
sc query qemu-ga
```

---

### 10.9 如果 Windows 11 蓝屏或无法启动

#### 10.9.1 情况 A：提示找不到启动盘

检查启动顺序：

```bash
qm config 101
qm set 101 --boot order=scsi0
```

进入 OVMF BIOS 检查启动项：

```text
Console -> ESC -> Boot Manager
```

---

#### 10.9.2 情况 B：VirtIO SCSI 驱动缺失

如果原 VM 使用 SATA/IDE，Windows 可能没有 VirtIO SCSI 驱动。

可以临时将磁盘挂为 SATA：

```bash
qm set 101 --delete scsi0
qm set 101 --sata0 local-lvm:vm-101-disk-0
qm set 101 --boot order=sata0
qm start 101
```

进入系统后安装 VirtIO 驱动，再关机切换回 SCSI：

```bash
qm shutdown 101
qm set 101 --delete sata0
qm set 101 --scsi0 local-lvm:vm-101-disk-0,discard=on,iothread=1
qm set 101 --boot order=scsi0
```

---

#### 10.9.3 情况 C：UEFI 启动项丢失

从 Windows 安装 ISO 进入修复环境，执行：

```cmd
diskpart
list vol
```

找到 EFI 分区后，例如 EFI 分区为 `S:`，Windows 分区为 `C:`：

```cmd
bcdboot C:\Windows /s S: /f UEFI
```

---

## 十一、 迁移 Linux 虚拟机

以下示例假设：

|项目|值|
|---|---|
|VM 名称|`ubuntu-1`|
|VMID|`102`|
|磁盘|`/mnt/migration/vm-disks/ubuntu-1.qcow2`|
|存储池|`local-lvm`|

---

### 11.1 创建 Linux VM

```bash
qm create 102 \
  --name ubuntu-1 \
  --memory 4096 \
  --cores 4 \
  --cpu host \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0 \
  --agent enabled=1
```

如果原 Linux 是 UEFI 启动：

```bash
qm set 102 --bios ovmf
qm set 102 --efidisk0 local-lvm:0,efitype=4m
```

如果原 Linux 是传统 BIOS，不需要设置 OVMF。

---

### 11.2 导入磁盘

```bash
qm importdisk 102 /mnt/migration/vm-disks/ubuntu-1.qcow2 local-lvm --format raw
qm config 102
```

假设导入后为：

```text
unused0: local-lvm:vm-102-disk-0
```

挂载磁盘：

```bash
qm set 102 --scsi0 local-lvm:vm-102-disk-0,discard=on,iothread=1
qm set 102 --boot order=scsi0
```

启动：

```bash
qm start 102
```

---

### 11.3 Linux 启动后安装 Guest Agent

Debian/Ubuntu：

```bash
sudo apt update
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```

CentOS/RHEL/Rocky/Alma：

```bash
sudo dnf install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```

Proxmox 中确认：

```bash
qm agent 102 ping
```

---

## 十二、 批量迁移流程模板

每台虚拟机按以下流程执行：

```text
1. 查看原 VM XML
2. 查看原磁盘路径
3. 查看 BIOS/UEFI
4. 查看网卡 MAC
5. 在 Proxmox 创建空 VM
6. 导入磁盘
7. 挂载磁盘
8. 设置启动顺序
9. 启动测试
10. 安装 Guest Agent
11. 验证业务
12. 设置开机自启和备份
```

---

## 十三、 libvirt XML 与 Proxmox 配置对应关系

|libvirt/QEMU 项|Proxmox VE 配置|
|---|---|
|VM 名称|`--name`|
|UUID|一般无需迁移|
|vCPU|`--cores` / `--sockets`|
|内存|`--memory`|
|CPU mode host-passthrough|`--cpu host`|
|UEFI/OVMF|`--bios ovmf` + `efidisk0`|
|TPM|`tpmstate0`|
|qcow2/raw 磁盘|`qm importdisk`|
|virtio 网卡|`--net0 virtio=MAC,bridge=vmbr0`|
|SATA 磁盘|`--sata0`|
|VirtIO SCSI|`--scsihw virtio-scsi-single` + `--scsi0`|
|CDROM|`--ide2 local:iso/xxx.iso,media=cdrom`|
|自动启动|`--onboot 1`|
|QEMU Guest Agent|`--agent enabled=1`|

---

## 十四、 配置虚拟机开机自启

例如 `win11-2`：

```bash
qm set 101 --onboot 1 --startup order=1,up=60
```

Linux VM：

```bash
qm set 102 --onboot 1 --startup order=2,up=30
```

说明：

|参数|说明|
|---|---|
|`onboot 1`|宿主机启动时自动启动 VM|
|`startup order`|启动顺序|
|`up`|启动后等待秒数|

---

## 十五、 配置备份

### 15.1 创建备份目录

如果使用本地目录：

```bash
mkdir -p /backup/pve
```

在 Proxmox Web UI 添加：

```text
Datacenter -> Storage -> Add -> Directory
```

参数示例：

|项目|值|
|---|---|
|ID|`backup-local`|
|Directory|`/backup/pve`|
|Content|`VZDump backup file`|

---

### 15.2 配置定时备份

Web UI：

```text
Datacenter -> Backup -> Add
```

建议：

|项目|建议|
|---|---|
|Schedule|每日或每周|
|Selection mode|Include selected VMs|
|Mode|Snapshot|
|Compression|ZSTD|
|Retention|保留 7 天或更多|
|Storage|backup-local / NAS / PBS|

生产环境强烈建议使用 Proxmox Backup Server。

---

### 15.3 手动备份测试

```bash
vzdump 101 --storage backup-local --mode snapshot --compress zstd
```

检查备份文件：

```bash
ls -lh /backup/pve/dump/
```

---

## 十六、 win11-2 稳定性优化建议

迁移完成后，针对 `win11-2` 建议做以下优化。

### 16.1 Proxmox VM 配置建议

```bash
qm set 101 --cpu host
qm set 101 --scsihw virtio-scsi-single
qm set 101 --agent enabled=1
qm set 101 --balloon 0
```

说明：

|项目|建议|
|---|---|
|CPU|`host`|
|磁盘控制器|VirtIO SCSI Single|
|磁盘缓存|默认或谨慎使用 write back|
|Balloon|对 Windows 桌面 VM 可考虑关闭|
|内存|不要过度超分|
|存储|SSD/NVMe 优先|
|Guest Agent|必须安装|
|VirtIO 驱动|使用较新稳定版本|

---

### 16.2 Windows 内部优化

管理员 CMD：

```cmd
powercfg /h off
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
chkdsk C: /scan
```

建议检查：

```text
事件查看器 -> Windows 日志 -> 系统
事件查看器 -> Windows 日志 -> 应用程序
可靠性监视器
任务管理器 -> 性能
```

重点关注：

- Disk timeout

- Display driver reset

- RDP 服务异常

- termsrv 相关错误

- 内存不足

- 虚拟磁盘 I/O 卡死

- Windows Update 后 RDP Wrapper 不兼容

---

### 16.3 宿主机硬件检查

在 Proxmox VE 上检查磁盘健康：

```bash
apt install smartmontools -y
smartctl -a /dev/sda
smartctl -a /dev/nvme0n1
```

查看内核错误：

```bash
dmesg -T | egrep -i "error|fail|reset|timeout|nvme|ata|i/o"
```

查看 Proxmox 日志：

```bash
journalctl -p warning..alert -b
```

查看 VM 任务日志：

```bash
cat /var/log/syslog | grep "101"
```

---

## 十七、 防止再次强制断电导致 Windows 自动修复

### 17.1 启用 Guest Agent 后优雅关机

```bash
qm shutdown 101
```

如果 Guest Agent 正常，Proxmox 可以更可靠地通知 Windows 关机。

---

### 17.2 避免直接 Stop

不要优先使用：

```bash
qm stop 101
```

`qm stop` 类似强制断电，容易导致 Windows 下次自动修复。

---

### 17.3 设置 UPS

如果服务器用于多人远程桌面，建议配置 UPS，并让 Proxmox 在断电时自动关机。

---

### 17.4 迁移验证清单

每台 VM 完成迁移后检查：

```text
[ ] VM 可以正常启动
[ ] 系统盘识别正常
[ ] 数据盘识别正常
[ ] IP 地址正确
[ ] MAC 地址符合预期
[ ] DNS 正常
[ ] 网关正常
[ ] 业务端口正常
[ ] Windows 远程桌面正常
[ ] Linux SSH 正常
[ ] Guest Agent 正常
[ ] 时间同步正常
[ ] 自动启动已配置
[ ] 备份任务已配置
[ ] 手动备份成功
[ ] 重启 VM 后业务仍正常
[ ] 重启 Proxmox 后 VM 自动启动正常
```

---

## 十八、 回滚方案

在正式切换业务前，必须保留旧环境备份。

### 18.1 不删除旧虚拟机磁盘

至少保留：

```text
原始 qcow2/raw 磁盘
libvirt XML
网络配置
备份校验文件
```

---

### 18.2 回滚到 Ubuntu/KVM

如果 Proxmox 上启动失败，可以将原磁盘恢复到 Ubuntu/KVM：

```bash
virsh define /backup/libvirt-xml/win11-2.xml
virsh start win11-2
```

前提是原 Ubuntu/KVM 环境仍保留或已经可恢复。

---

### 18.3 回滚到 Proxmox 迁移前状态

如果 Proxmox VM 已启动但系统异常，不要覆盖原始备份。

可以删除 Proxmox 中导入失败的 VM 后重新导入：

```bash
qm stop 101
qm destroy 101 --purge
```

然后重新执行创建和导入流程。

---

## 十九、 推荐最终架构

```text
Proxmox VE Host
├── vmbr0：桥接物理网卡，提供 VM 局域网访问
├── local-lvm / ZFS：虚拟机磁盘存储
├── backup-local / NAS / PBS：备份存储
├── VM 101：win11-2
│   ├── OVMF UEFI
│   ├── TPM 2.0
│   ├── VirtIO 网卡
│   ├── VirtIO SCSI 磁盘
│   └── QEMU Guest Agent
├── VM 102：Linux VM
└── VM 103：其他业务 VM
```

---

## 二十、 建议迁移顺序

1. 先迁移非关键 Linux VM。

2. 再迁移普通 Windows VM。

3. 最后迁移 `win11-2`。

4. `win11-2` 迁移后不要马上开放所有用户并发登录。

5. 先单用户 RDP 测试。

6. 再小范围并发测试。

7. 最后正式切换。

---

## 二十一、 常用命令汇总

## 二十二、 Ubuntu/KVM 侧

```bash
virsh list --all
virsh dumpxml VM名称 > VM名称.xml
virsh domblklist VM名称 --details
qemu-img info /path/to/disk.qcow2
virsh shutdown VM名称
virsh destroy VM名称
```

---

## 二十三、 Proxmox VE 侧

```bash
pveversion -v
qm list
qm config VMID
qm create VMID --name VM名称
qm importdisk VMID 磁盘路径 存储池 --format raw
qm set VMID --scsi0 存储池:vm-VMID-disk-0
qm set VMID --boot order=scsi0
qm start VMID
qm shutdown VMID
qm stop VMID
qm destroy VMID --purge
vzdump VMID --storage 备份存储 --mode snapshot --compress zstd
```

---

## 二十四、 win11-2 示例完整命令

请按实际 CPU、内存、MAC、磁盘路径修改。

```bash
qm create 101 \
  --name win11-2 \
  --memory 16384 \
  --cores 8 \
  --cpu host \
  --machine q35 \
  --bios ovmf \
  --scsihw virtio-scsi-single \
  --net0 virtio=52:54:00:12:34:56,bridge=vmbr0 \
  --agent enabled=1

qm set 101 --efidisk0 local-lvm:0,efitype=4m,pre-enrolled-keys=1
qm set 101 --tpmstate0 local-lvm:0,version=v2.0

qm importdisk 101 /mnt/migration/vm-disks/win11-2.qcow2 local-lvm --format raw

qm config 101

qm set 101 --scsi0 local-lvm:vm-101-disk-0,discard=on,iothread=1
qm set 101 --boot order=scsi0
qm set 101 --ide2 local:iso/virtio-win.iso,media=cdrom
qm set 101 --onboot 1 --startup order=1,up=60

qm start 101
```

---

## 二十五、 结论

本次迁移的核心不是简单复制磁盘，而是：

1. 先完整备份 Ubuntu/KVM 的 VM 磁盘和 XML。

2. 全新安装 Proxmox VE。

3. 在 Proxmox VE 中重建 VM 配置。

4. 导入原 VM 磁盘。

5. 对 Windows 11 VM 保留 UEFI、TPM、MAC、VirtIO 驱动和 Guest Agent。

6. 对 `win11-2` 重点处理卡死、强制断电、RDP Wrapper、磁盘一致性和备份问题。

7. 迁移完成后建立 Proxmox 原生备份机制，避免以后只能强制关机和等待自动修复。

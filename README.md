# 在PVE中安裝macOS Mojave 並設置直通顯卡
## 先決條件：
+ 8G或以上內存
+ 支持虛擬化以及SSE4.2的cpu

> *包括AMD與INTEL*

## 1 安裝PVE
此處安裝方法與一般的方法相同，推薦將pve系統安裝到usb設備

## 2 創建macos安裝鏡像
在linux系統下運行以下命令
```bash
wget  https://raw.githubusercontent.com/thenickdude/OSX-KVM/master/fetch-macOS.py 
chmod +x fetch-macOS.py
./fetch-macOS.py
```
在選單中選擇最新的iso，等待下載完成後，運行以下命令，將dmg包轉換成iso包
```
# 安裝dmg2img 若你是deb包管理系以外的系統，可以自行查找如何安裝dmg2img
apt-get install dmg2img -y 
# 利用dmg2img將基本系統dmg鏡像轉為iso鏡像
dmg2img BaseSystem.dmg Mojave-installer.iso
```
然後下載Clover
```
# Ubuntu & Debian
apt-get install unzip -y
# Centos & RHEL
yum install unzip -y
# 取得clover
wget https://github.com/thenickdude/OSX-KVM/releases/download/clover-r4920/clover-r4920.iso.zip && unzip clover-r4920.iso.zip
```
然後將clover-r4920.iso與Mojave-installer.iso上傳到PVE中

## 3 在網頁中創建虛擬機

1. OS頁面選擇Clover ISO進行引導
2. 系統選擇其他（other）
3. System頁面的顯示卡選擇VMware兼容
4. BIOS選擇OVMF
5. Machine選擇q35
6. 硬盤選擇SATA，緩存設置成Write back（不安全）
7. CPU的類型設置成Penryn
8. 網卡設置成Vmware vmxnet3

點開虛擬機的硬件選項卡，添加Mojave-installer.iso，選擇ide通道的cd-ram

先不要啟動虛擬機，在ssh中打開/etc/pve/qemu-server/你的VMID.conf
```
nano /etc/pve/qemu-server/你的VMID.conf
```
然後輸入以下代碼
```
args: -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" -smbios type=2 -cpu Penryn,kvm=on,vendor=GenuineIntel,+invtsc,vmware-cpuid-freq=on,+pcid,+ssse3,+sse4.2,+popcnt,+avx,+aes,+xsave,+xsaveopt,check -device usb-kbd,bus=ehci.0,port=2
```
將兩個驅動器的配置中的cdrom刪除，加入cache=unsafe
最後的文檔應該看起來像這樣
```
args: -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" -smbios type=2 -cpu Penryn,kvm=on,vendor=GenuineIntel,+invtsc,vmware-cpuid-freq=on,+pcid,+ssse3,+sse4.2,+popcnt,+avx,+aes,+xsave,+xsaveopt,check -device usb-kbd,bus=ehci.0,port=2
balloon: 0
bios: ovmf
boot: cdn
bootdisk: ide2
cores: 4
cpu: Penryn
efidisk0: vms:vm-144-disk-1,size=128K
ide0: isos:iso/Mojave.iso,cache=unsafe
ide2: isos:iso/clover-r4920.iso,cache=unsafe
machine: q35
memory: 8192
name: mojave
net0: vmxnet3=xx:xx:xx:xx:xx:xx,bridge=vmbr0,firewall=1
numa: 0
ostype: other
sata0: vms:vm-144-disk-0,cache=unsafe,size=64G
smbios1: uuid=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx
sockets: 1
vga: vmware
```
設置完畢後，根據[patch-ovmf-to-support-macos-in-proxmox-5-1 ](https://www.nicksherlock.com/2018/04/patch-ovmf-to-support-macos-in-proxmox-5-1/)來安裝OVMF庫

## 4 設置直通
首先，編輯grub
```
vim /etc/default/grub
```
將
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```
改為
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on video=efifb:off"
```
***（若是amd則改為amd_iommu=on）***

執行以下命令更新grub信息
```
update-grub
```
最後輸入以下命令檢查是否有錯
```
dmesg | grep -e DMAR -e IOMMU
```
然後編輯/etc/modules
```
nano /etc/modules
```
在最尾加入以下四行
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
再次輸入以下命令檢查設備是否支持iommu
```
dmesg | grep ecap
```
然後執行以下命令將驅動加入黑名單
```
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
```
執行以下命令更新信息
```
update-initramfs -u
```
執行lspci，找出顯卡的代號（例如01：00），然後執行
```
lspci -n -s 01:00
```
得到類似以下的輸出
```
01:00.0 0300: 10de:1d01 (rev a1)
01:00.1 0403: 10de:0fb8 (rev a1)
```
其中10de:1d01與10de:0fb8是vendor IDs
將vendor IDs指定到VFIO模塊
```
echo "options vfio-pci ids=10de:1d01,10de:0fb8" > /etc/modprobe.d/vfio.conf
```
進入web管理介面
+ 在vm的硬件選項卡中編輯添加PCI設備
+ 選定顯卡對應的01:00.0與01:00.1
+ 在01:00.0（顯示設備）中勾選
	+ PCI-Express
	+ All Functions
	+ 主GPU
+ 檢查文件/etc/pve/qemu-server/YOUR-VM-ID.conf
	+ 確保01:00後的參數正確

**e.g.**
```
hostpci0: 01:00,x-vga=1,pcie=1
```

然後，直通鼠標鍵盤到VM中，添加USB設備，選擇鼠標鍵盤，然後添加到VM

## 5 安裝Mojave
現在啟動你的虛擬機，若你設置了直通，啟動的一瞬間會發生以下事情
* 連接在你顯卡的顯示器應該會從PVE的ttl介面變為黑屏，然後變成OVMF UEFI啟動介面。
* 你的鍵盤鼠標這時候也可以直接控制虛擬機。

在啟動的時候趕快按下F2以進入OVMF設置畫面。
1. 進入Device Manager
2. 選擇OVMF platform configuration
3. 設置分辨率為1920x1080
4. 保存設置
5. 在根菜單下選擇Reset（不是continue）

然後你應該進入了Clover，接下來跟著指引安裝系統
全部安裝完畢之後，再次啟動時
1. 手動按下F2
2. 選定Clover啟動項

然後才會看到磁盤內的Mac系統。
進入Mac系統後，打開終端，輸入
```
diskutil list
```
以檢查設備
然後輸入
```
sudo dd if=<Clover CD的EFI分區> of=<硬盤的EFI分區> 
```
將Clover安裝到硬盤中
> 你也可以忽略這一步，將Clover CD永久掛載在虛擬機下

關機，移除CLover CD，從硬盤啟動。



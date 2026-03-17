# 磁盘占用检查，服务器无损扩容

---

## 一、学习目标

1. 复现一线运维场景的磁盘故障

2. 掌握磁盘故障排查思路与解决方案（判断何时需要扩容）

3. 故障根治方法：理论 + 实战应用

---

## 二、典型故障场景：空间充足却无法写入

### 故障现象

- 业务高峰期数据库崩溃，提示：`No space left on device`

- 监控无报警，`df -h` 显示剩余空间 50G

- 根本原因：**inode 耗尽**，监控未监控 inode

### 错误处理方式

- A：重启服务器 → 文件在内存未落盘，重启可能无法写入、无法启动

- B：直接删文件 → 未分析盲目删除有风险

- C：申请加硬盘 → 远水解不了近渴

---

## 三、inode 核心原理

- 类比：磁盘 = 公寓楼，文件 = 房间，**inode = 房号**

- ext4：初始化时 inode 数量固定，用完即无法新建文件

- xfs：inode 动态分配，几乎不会出现空间够但 inode 满的问题

### 磁盘满的两种情况

1. 存储空间满：`df -h`

2. inode 满：`df -i`

```Bash
df -i  # 查看 inode 使用情况
```

---

## 四、inode 耗尽排查与清理

### 1. 排查步骤

1. 先用 `df -i` 确认是否 inode 满

2. 再用 `df -h` 确认是否空间满

3. 定位占用 inode 最多的目录

### 2. 批量删除小文件（解决 rm 参数过长）

```Bash
# 先预览再删除
find /root/inode-test -type f -print
# 直接删除
find /root/inode-test -type f -delete
```

### 3. 定位高 inode 占用目录

```Bash
# 查看一级子目录 inode 使用量
du --inodes -s /* 2>/dev/null | sort -rh | head -n 10
```
1. du --inodes
du = 查看磁盘 / 文件占用情况
--inodes = 不看空间大小，只看 inode 数量
2. -s
-s = 只显示汇总总数，不展开子目录（干净）
3. /*
表示 根目录下所有一级文件夹
/bin /etc /var /usr /root 全部统计
4. 2>/dev/null
Linux 系统里有三个默认 “通道”：
0 = 标准输入（键盘输入）
1 = 标准输出（正常打印的内容）
2 = 标准错误（报错、警告、权限不足）
所以：
1> = 输出正常内容
2> = 输出错误内容
/dev/null 垃圾桶
把报错信息丢掉（比如权限不够的目录不弹错误）
让输出干净整洁
5. |
管道符 → 把前面命令的结果传给后面继续处理
6. sort -rh
sort = 排序
-r = 倒序（从大到小）
-h = 自动识别数字大小
7. head -n 10
只显示 前 10 行（inode 最多的前 10 个目录）

# 统计各目录文件数量并排序
find . -mindepth 1 -maxdepth 1 -type d | 
while read dir; do
count=$(find "$dir" -type f | wc -l)
echo -e "$count\t$dir"
done | sort -nr | head -n 10 | awk -F '\t' '{print $2 ":" $1 "个文件"}'
```
find . -mindepth 1 -maxdepth 1 -type d
作用：列出当前目录下的 第一层 子目录
. = 当前目录
-mindepth 1 = 不从当前目录开始
-maxdepth 1 = 只找第一层子目录
-type d = 只找目录，不找文件

| 管道就是把结果传给下一条命令

count=$(find "$dir" -type f | wc -l)
作用：统计这个目录里有多少个文件
find "$dir" -type f = 找这个目录下所有文件
wc -l = 统计行数 = 统计文件数量
count=$(...) = 把数量存到变量 count 里

echo -e "$count\t$dir"
输出：数量 + 目录名
\t = 按一下 Tab 键（对齐用）

| sort -nr
把结果按文件数量 从大到小排序
-n = 按数字排序
-r = 倒序（大 → 小）

| head -n 10
只保留 前 10 个文件最多的目录

| awk -F '\t' '{print $2 ":" $1 "个文件"}'
格式化输出，让人看得更舒服
-F '\t' = 按 Tab 分割
$2 = 目录名
$1 = 文件数量

### 4. find 常用参数

- `-type f`：普通文件

- `-type d`：目录

- `-mindepth 1`：最小深度

- `-maxdepth 1`：最大深度

- `-mtime -7`：7天内文件

### 5. 按条件保留/删除文件

```Bash
# 保留7天内 .log 或 .session 文件，其余删除
find /var/log -type f ! -mtime -7 ! -name "*.log" ! -name "*.session" -delete
```

---

## 五、LVM 在线扩容（无损业务）

### LVM 三层结构

- PV（Physical Volume）：物理卷 → 水桶

- VG（Volume Group）：卷组 → 水池

- LV（Logical Volume）：逻辑卷 → 水龙头

- 关系图：

  ![image-20260316194838360](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316194838360.png)

### 标准扩容四步流程

#### 1. 识别新硬盘

```Bash
lsblk  # 查看磁盘
# 未识别新盘时重新扫描
echo "- - -" > /sys/class/scsi_host/host0/scan
echo "- - -" > /sys/class/scsi_host/host1/scan
echo "- - -" > /sys/class/scsi_host/host2/scan
```

#### 2. 创建 PV（物理卷）

```Bash
pvcreate /dev/sdd /dev/sdc
```

#### 3. 扩展 VG（卷组）

```Bash
vgs  # 查看卷组名
 vgcreate vg_WSL /dev/sdd /dev/sdc
vgextend vg_WSL /dev/sdf  # 扩展卷组
```

#### 4. 扩展 LV（逻辑卷）

```Bash
lvcreate -l 100%FREE -n lv_data vg_wsl  #100%挂满

lvextend -l +100%FREE /dev/vg_WSL/lv_data
```
#### 5.创建挂载点,进行格式化并挂载

```Bash
# 自定义目录，名字随便改
sudo mkdir -p /mnt/lvm_storage
# 格式化为ext4
mkfs.ext4 /dev/vg_wsl/lv_data
# 挂载到你自己建的目录，和WSL默认路径无关
sudo mount /dev/vg_wsl/lv_data /mnt/lvm_storage
```
#### 6. 刷新文件系统（关键）

让文件系统 “识别并用上” 扩容后的磁盘空间

```Bash
# 查看文件系统类型
df -T

# ext4
resize2fs /dev/vg_WSL/lv_data

# xfs
xfs_growfs /dev/vg_WSL/lv_data
```

#### 7. 设置永久挂载
1. **先获取逻辑卷 UUID（WSL 内执行）**

  ```bash
  blkid /dev/vg_wsl/lv_data
  ```

  复制输出里的 UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx 这一串
2. **写入开机自动挂载（永久生效）**
    把命令里的 【你的 UUID】 替换成刚复制的字符串，执行：

  ```bash
  echo "UUID=【你的UUID】 /mnt/data ext4 defaults 0 0" >> /etc/fstab
  ```
3. **测试配置（无报错 = 成功）**

  ```bash
  # 1. 重载系统挂载配置（无需重启整机）
  systemctl daemon-reload
  # 2. 重新挂载所有分区（验证永久配置）
  mount -a
  ```

  

  #### WSL 原生永久挂载

  **注意：此实验需要Windows 永久挂载 VHDX**（跟传统linux在fstab挂载不一样）

 	1. ​	**Windows 磁盘永久加载**
 	 新建 `C:\Users\你的用户名\.wslconfig`：

  ```ini
  	[wsl2] mountVhd=D:\wsl_disk.vhdx,type=bare
  ```

2. ​	**WSL 开机自动挂载**

   ```
   # 创建WSL启动指令
   cat > /etc/wsl.conf << EOF
   [boot]
   command = vgchange -ay vg_wsl && mount /dev/vg_wsl/lv_data /mnt/data
   EOF
   ```

3.  **重启 WSL 服务生效**

  ```bash
  # 1. 关闭所有WSL进程 
  wsl --shutdown 
  # 2. 重启WSL系统服务（强制刷新挂载配置） 
  net stop LxssManager 
  net start LxssManager
  ```



---

## 六、日志目录迁移（预防 inode 爆满）

### 适用场景

ext4 日志目录产生海量小文件，迁移到 XFS 分区

### 操作步骤

1. 格式化新盘为 XFS

```Bash
mkfs.xfs /dev/sdc
```

1. 停止日志服务

```Bash
systemctl stop rsyslog systemd-journald
```

1. 安装迁移工具

```Bash
dnf install -y rsync
```

1. 迁移数据（保留权限/属性）

```Bash
rsync -aHAX /var/log/ /mnt/newlog/
```

1. 配置永久挂载

```Bash
UUID=$(blkid -s UUID -o value /dev/sdc)
echo "UUID=$UUID /var/log xfs defaults,noatime 0 0" >> /etc/fstab
```

1. 重新挂载 & 启动服务

```Bash
umount /mnt/newlog
mount -a
systemctl start rsyslog systemd-journald
systemctl daemon-reload
```

---

## 七、运维避坑指南

1. XFS 只支持扩容，**不支持缩容**

2. 删除文件先分析，用 find 精准清理

3. 磁盘监控必须双指标：`df -h` + `df -i`

4. 关键目录独立挂载：`/var/log`、`/var/lib/docker` 等

5. 生产环境避免所有目录都挂载到根分区

---

## 八、常用命令速查

|用途|命令|
|---|---|
|查看磁盘空间|df -h|
|查看 inode|df -i|
|查看磁盘结构|lsblk|
|创建物理卷|pvcreate|
|扩展卷组|vgextend|
|扩展逻辑卷|lvextend|
|刷新 ext4|resize2fs|
|刷新 xfs|xfs_growfs|
|批量删文件|find ... -delete|
|数据迁移|rsync -aHAX|
---



#### 实践部分（WSL准备工作）

##### 第一步：创建 2 块空白裸 VHDX（LVM 专用，无格式无分区）

以**管理员身份**打开 PowerShell，输入 `diskpart` 进入工具

##### 逐行复制执行（创建磁盘 1）

diskpart

```
create vdisk file="D:\WSL-Disks\lvm-disk1.vhdx" maximum=10240 type=expandable
select vdisk file="D:\WSL-Disks\lvm-disk1.vhdx"
detach vdisk #分离
```

##### 创建磁盘 2

```
create vdisk file="D:\WSL-Disks\lvm-disk2.vhdx" maximum=10240 type=expandable
select vdisk file="D:\WSL-Disks\lvm-disk2.vhdx"
detach vdisk
```

##### 创建磁盘 3（拓展lvm使用）
```
create vdisk file="D:\WSL-Disks\lvm-disk3.vhdx" maximum=10240 type=expandable
select vdisk file="D:\WSL-Disks\lvm-disk3.vhdx"
detach vdisk
exit
```
##### 第二步：把 2 块盘 裸挂载到 WSL（LVM 必须用裸设备）

powershell

```
wsl --shutdown
# 挂载第一块盘
wsl --mount D:\WSL-Disks\lvm-disk1.vhdx --vhd --bare
# 挂载第二块盘
wsl --mount D:\WSL-Disks\lvm-disk2.vhdx --vhd --bare
# 挂载第三块盘
wsl --mount D:\WSL-Disks\lvm-disk3.vhdx --vhd --bare
```

....

#### 展示成果：

##### lvm基本创建+临时挂载：

![wsl系统挂载识别：](C:\Users\Lesch\Pictures\Screenshots\Screenshot 2026-03-16 193416.png)



![检查磁盘](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316193702306.png)

![创建过程](C:\Users\Lesch\Pictures\Screenshots\Screenshot 2026-03-16 195455.png)

格式化+临时挂载![image-20260316195842920](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316195842920.png)

![image-20260316195856616](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316195856616.png)

![image-20260316195922777](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316195922777.png)

无损拓展并检查：（准备拓展分区）

![image-20260316200048313](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316200048313.png)

![image-20260316200124939](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316200124939.png)

![扩展](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316200447215.png)

拓展后需要对应格式类型做系统同步：

![image-20260316200615385](C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260316200615385.png)

------

### 永久挂载（WSL）

### 1. 先激活 LVM 卷组

```Bash
sudo vgscan
sudo vgchange -ay vg_wsl
```

你会看到提示：`1 logical volume(s) in volume group "vg_wsl" now active`，说明卷组已经在运行了。

### 2. 找到 LVM 设备的「设备号」（关键！）

```Bash
sudo dmsetup ls | grep vg_wsl
```

输出会是类似这样的格式：

```Plain
vg_wsl-lv_data (253:0)
```

这里的 `253:0` 就是 **主设备号:次设备号**，记下来（你的数字可能不一样）。

### 3. 手动创建缺失的设备节点（WSL 专属修复）

```Bash
# 先创建 mapper 目录（如果不存在）
sudo mkdir -p /dev/mapper
# 用你刚才查到的设备号创建设备节点（把 253:0 换成你的实际数字）
sudo mknod /dev/mapper/vg_wsl-lv_data b 253 0
```

### 4. 直接挂载！

```Bash
sudo mount /dev/mapper/vg_wsl-lv_data /mnt/data
```

## ✅ 验证成功

执行：

```Bash
df -h /mnt/data
```

你会看到类似输出，说明挂载成功：

```Plain
文件系统                     大小  已用  可用 已用% 挂载点
/dev/mapper/vg_wsl-lv_data   30G  1.2G   29G    4% /mnt/data
```

最后做个脚本mount_lvm.sh

```Bash
#!/bin/bash
sudo vgchange -ay vg_wsl
sudo mount /dev/mapper/vg_wsl-lv_data /mnt/data
echo "挂载完成！"
```

给脚本加权限：`chmod +x ~/mount_lvm.sh`，之后运行 `./mount_lvm.sh` 就能一键恢复。

**windows也要挂载！！！！**

```
# WSL 自动挂载 LVM 磁盘脚本
# 先关闭可能运行的 WSL 实例（避免挂载冲突）
wsl --shutdown
Start-Sleep -Seconds 3  # 等待 3 秒确保完全关闭

# 挂载 3 个 VHDX 磁盘到 WSL（--bare 表示仅挂载为块设备，不自动格式化）
wsl --mount D:\WSL-Disks\lvm-disk1.vhdx --vhd --bare
wsl --mount D:\WSL-Disks\lvm-disk2.vhdx --vhd --bare
wsl --mount D:\WSL-Disks\lvm-disk3.vhdx --vhd --bare

```

`nano ~/.bashrc`

```bash
# 自动激活 LVM 并挂载到 /mnt/data

if ! mountpoint -q /mnt/data; then
  sudo vgchange -ay vg_wsl
  sudo mount /dev/mapper/vg_wsl-lv_data /mnt/data
  echo "✅ LVM 已自动挂载到 /mnt/data"
fi
```

<img src="C:\Users\Lesch\AppData\Roaming\Typora\typora-user-images\image-20260317005017362.png" alt="在wsl虚拟机中创建挂载" style="zoom:50%;" />

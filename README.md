# 运维学习文档 | OPS Learning Notes

个人运维实战沉淀仓库，记录 Linux 系统、网络、虚拟化、数据库等方向的实操笔记与自动化脚本，用于技术复盘与面试展示。

## 📚 仓库目录

- `scripts/`：实用 Shell 脚本（系统巡检、数据库备份、日志清理、环境自动化部署）

- `docs/`：项目实操文档（WSL2 环境配置、NFS 存储、KVM 虚拟化、开源项目部署等）

- `notes/`：学习笔记（Linux 基础、网络原理、数据库运维等）

## 🎯 核心实践项目

1. **WSL2 环境优化 + Git 代理 + 双开源项目部署**

    - 配置 WSL 代理加速 GitHub 访问

    - 自动化编译安装 `fastfetch` 系统信息工具

    - 部署 Apple `ml-sharp` Python 项目，掌握虚拟环境与依赖管理

2. **KVM 虚拟化集群 + NFS 网络存储**

    - 搭建跨节点 KVM 集群，实现虚拟机热迁移

    - 配置 NFS 共享存储，保障数据高可用

3. **MySQL 数据库运维与自动化备份**

    - 主从架构部署、慢查询优化

    - Shell 脚本实现定时全量备份与过期数据清理

4. **企业级网络设备调试（网工基础）**

    - 华为/H3C 交换机/路由器配置实践

    - OSPF/VRRP/ACL/VLAN 等企业网核心功能部署

## 🛠️ 技术栈

- **系统**：Ubuntu / CentOS / EulerOS / WSL2

- **网络**：TCP/IP、华为/H3C 设备、NFS

- **虚拟化/容器**：KVM、Libvirt、Docker

- **自动化**：Shell、Ansible（基础）、GitHub Actions

- **数据库**：MySQL 8.0（备份、主从、慢查询优化）

- **语言**：Python（环境管理/脚本）、C（数据结构基础）

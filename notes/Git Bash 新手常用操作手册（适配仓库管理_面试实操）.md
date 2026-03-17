- # Git Bash 新手常用操作手册（适配仓库管理）

  ## 一、基础准备

  **逻辑：建库 →git clone到本地 →本地再从git bash里面操作内容（add commit push）**

  ### 1. 打开 Git Bash

  电脑左下角搜索「Git Bash」，点击打开（黑窗口即为操作界面）；

  若需操作指定仓库，右键仓库文件夹（如 OPS_Note）→ 选择「Git Bash Here」，可直接进入该仓库目录，无需额外定位。

  ### 2. 基础验证命令
  
  ```Bash
  # 查看 Git 版本（验证是否安装成功）
  git --version
  
  # 查看当前所在目录（确认是否在目标仓库）
  pwd
  
  # 查看仓库当前状态（是否有未提交文件、冲突等）
  git status
  ```

  ## 二、仓库核心操作（高频使用）

  ### 1. 进入/切换仓库目录
  
  ```Bash
  # 格式：cd 仓库本地路径（复制仓库文件夹地址替换即可）
  cd D:\OPS_Note  # 示例：进入本地 OPS_Note 仓库
  cd /d/OPS_Note  # 若路径显示为 Linux 格式，用此命令
  ```

  ### 2. 本地仓库初始化（新建仓库时用）
  
  ```Bash
  # 在当前目录初始化 Git 仓库（生成 .git 隐藏文件夹）
  git init
  
  # 关联远程 GitHub 仓库（首次新建仓库后关联）
  git remote add origin https://github.com/你的用户名/OPS_Note.git
  ```

  ### 3. 克隆远程仓库到本地（首次获取仓库时用）
  
  ```Bash
  # 格式：git clone 远程仓库地址
  git clone https://github.com/你的用户名/OPS_Note.git
  ```

  ## 三、文件提交与推送（最常用）

  核心流程：本地修改 → 暂存文件 → 本地提交 → 推送到远程 GitHub
  
  ```Bash
  # 1. 暂存所有修改的文件（. 表示所有文件）
  git add .
  
  # 2. 暂存指定文件（如只暂存 README.md）
  git add README.md
  
  # 3. 本地提交（必须写说明，简洁明了，面试友好）
  # 示例1：新增文件
  git commit -m "feat: 新增 NFS 配置文档"
  # 示例2：修改文件
  git commit -m "docs: 优化 README 内容"
  # 示例3：修复问题
  git commit -m "fix: 解决文件冲突"
  
  # 4. 推送到远程 GitHub（同步本地修改）
  git push origin main  # main 是你的主分支，默认即为 main
  ```

  ## 四、Commit 记录管理（你重点关注）

  ### 1. 查看 commit 历史
  
  ```Bash
  # 查看所有 commit 记录（简洁版）
  git log --oneline
  
  # 查看所有 commit 记录（详细版，包含时间、作者、说明）
  git log
  ```

  ### 2. 清空所有 commit 记录（保留文件，仅清历史）
  
  ```Bash
  # 1. 创建全新干净分支
  git checkout --orphan new_start
  # 2. 暂存所有文件
  git add .
  # 3. 新的初始化提交（面试友好版说明）
  git commit -m "init: 初始化运维实战文档仓库，用于技术复盘与面试展示"
  # 4. 删除原主分支
  git branch -D main
  # 5. 重命名新分支为主分支
  git branch -m main
  # 6. 强制推送，覆盖远程历史（必输，确认清空）
  git push -f origin main
  ```

  ### 3. 撤销最近一次 commit（未推送时用）
  
  ```Bash
  # 撤销最近一次 commit，文件保留在暂存区
  git reset --soft HEAD~1
  
  # 撤销最近一次 commit，文件恢复为未暂存状态（谨慎使用）
  git reset --hard HEAD~1
  ```

  ## 五、远程仓库操作（同步相关）

  ### 1. 拉取远程仓库最新内容（避免冲突）
  
  ```Bash
  # 拉取远程 main 分支的最新内容到本地
  git pull origin main
  ```

  ### 2. 查看远程仓库关联情况
  
  ```Bash
  git remote -v
  ```

  ### 3. 解除远程仓库关联（如需更换仓库时用）
  
  ```Bash
  git remote remove origin
  ```

  ## 六、常用辅助操作

  ### 1. 创建目录（适配你的 scripts/docs 目录）
  
  ```Bash
  # 创建 scripts 和 docs 目录
  mkdir scripts docs
  
  # 给空目录添加占位文件（Git 不识别空目录）
  touch scripts/.gitkeep docs/.gitkeep
  ```

  ### 2. 删除本地文件并提交到远程
  
  ```Bash
  # 删除本地文件（如删除无用的 test.txt）
  rm test.txt
  
  # 提交删除操作到远程
  git add .
  git commit -m "del: 删除无用测试文件"
  git push origin main
  ```

  ## 七、新手注意事项

  - 所有命令复制粘贴时，注意替换「你的用户名」「仓库路径」等占位内容；

  - 强制推送（git push -f）仅在清空 commit 历史时使用，日常推送用 git push 即可；

  - 提交说明（commit -m 后面的内容）尽量规范，避免写“修改”“更新”等模糊表述，面试时更显专业；
  
  - 若出现「冲突」，先解决冲突文件（删除冲突标记，保留需要的内容），再执行 add → commit → push。

### ✅ 命令行一键清空所有 commit 记录（最干净、最直接）

你现在的仓库有 4 条 commit，想让它变成**只有 1 条干净的初始化记录**，用下面这套命令，复制粘贴就能搞定，**不会删文件，只清历史**：

---

### 第一步：打开 Git Bash 并进入仓库

1. 找到你本地的 `OPS_Note` 文件夹

2. 右键空白处 → 选择 **`Git Bash Here`**

3. 确认在仓库里：输入 `git status`，看到 `On branch main` 就对了

---

### 第二步：复制粘贴这串命令（一键清空历史）

```Bash
# 1. 创建一个全新的干净分支
git checkout --orphan new_start

# 2. 把当前所有文件加入暂存
git add .

# 3. 提交一个新的初始化记录（写一句面试友好的说明）
git commit -m "init: 初始化运维实战文档仓库"

# 4. 删除原来的主分支
git branch -D main

# 5. 把新分支改名为 main
git branch -m main

# 6. 强制推送到 GitHub（覆盖所有旧记录）
git push -f origin main
```

---

### 第三步：验证效果

打开你的 GitHub 网页 → 进入 `OPS_Note` 仓库 → 点击 `Commits`

✅ 现在只会显示 **1 条 ** **`init: 初始化运维实战文档仓库`** 的记录，之前的 3 条杂乱记录全部消失！

---

### 📌 为什么 web 端不能删？

GitHub 网页端**故意隐藏了删除 commit 的功能**，就是为了防止误操作，所以必须用命令行的「强制推送」方式来清理历史，这是行业通用做法，面试官看了只会觉得你专业。

---

### 最终效果

- 你的 [README.md](README.md)、脚本、文档**一个都不会丢**

- 仓库历史瞬间变清爽，只有 1 条干净的初始化记录

- 完全符合你「展示实操文档、给面试看」的需求

要不要我帮你把**新的初始化 commit 说明**写得更贴合面试？比如：`init: 个人运维实战沉淀仓库，用于技术复盘与面试展示`，你直接替换就行。

> 
# Git 基础概念详解

## 1. Git 对象系统

### Git 对象类型
- **Blob对象**: 存储文件内容
- **Tree对象**: 存储目录结构
- **Commit对象**: 存储提交信息（作者、时间、消息等）
- **Tag对象**: 存储标签信息

### 对象存储位置
```
.git/objects/
├── 12/
│   └── 34567890abcdef...  （对象文件）
├── ab/
│   └── cdef1234567890...
```
- 对象按 SHA-1 哈希值的前两位分目录存储
- 例如：`c629483de0c86b7a8f6e23c7efbc3ef33e544e84` 存储在 `.git/objects/c6/29483de0c86b7a8f6e23c7efbc3ef33e544e84`

## 2. Git 垃圾回收 (gc - Garbage Collection)

### `git gc` 命令作用
- **整理对象**: 将松散对象打包成pack文件
- **删除无用对象**: 移除不再被引用的对象
- **优化性能**: 减少磁盘空间，提高访问速度

### 常用参数
```bash
git gc                    # 普通垃圾回收
git gc --aggressive      # 激进模式，更彻底的优化
git gc --prune=now       # 立即删除无用对象
```

## 3. Prune 概念

### `--prune` 的含义
- **Prune** = 修剪、删除
- 删除不再被任何分支或标签引用的对象
- 类似于"清理垃圾文件"

### Prune 的时间参数
```bash
--prune=now              # 立即删除所有无用对象
--prune=2.weeks.ago      # 删除2周前的无用对象
--prune=1.day.ago        # 删除1天前的无用对象
```

## 4. Git 状态检查

### `git fsck` (File System Check)
```bash
git fsck --full          # 完整检查仓库完整性
git fsck --dangling      # 只显示悬空对象
```

**检查内容**:
- 对象完整性
- 引用完整性
- 悬空对象（不被引用的对象）

## 5. Git 历史和引用

### Reflog（引用日志）
```bash
git reflog               # 查看引用变更历史
git reflog expire        # 清理reflog
```
- 记录分支指针的移动历史
- 即使提交被删除，reflog也可能保留引用

### HEAD 和分支
- **HEAD**: 当前检出的提交指针
- **分支**: 指向某个提交的可移动指针
- **远程分支**: 如 `origin/main`，跟踪远程仓库状态

## 6. 推送相关概念

### 推送过程
1. **打包本地对象**: 将本地新提交打包
2. **传输数据**: 发送到远程仓库
3. **远程解包**: 远程仓库解包并验证
4. **更新引用**: 更新远程分支指针

### 常见推送选项
```bash
git push                           # 推送当前分支
git push origin main              # 推送到指定远程分支
git push --force-with-lease       # 安全的强制推送
git push --force                  # 危险的强制推送
```

## 7. 对象损坏问题

### 损坏原因
- **存储设备问题**: 磁盘错误、断电等
- **并发访问**: 多个进程同时操作
- **网络传输错误**: 推送/拉取时的数据损坏

### 修复方法
```bash
# 1. 检查损坏
git fsck --full

# 2. 垃圾回收修复
git gc --aggressive --prune=now

# 3. 清理reflog
git reflog expire --expire=now --all

# 4. 删除损坏对象（危险操作）
rm .git/objects/xx/xxxxxxxxx...
```

## 8. 分支同步概念

### 本地 vs 远程
- **本地分支**: 你当前工作的分支
- **远程跟踪分支**: 本地对远程分支状态的记录
- **远程分支**: 实际存在于远程服务器上的分支

### 同步状态
```
ahead of 'origin/main' by X commits    # 本地领先远程X个提交
behind 'origin/main' by X commits      # 本地落后远程X个提交
diverged                               # 本地和远程分叉了
```

## 9. 认证方式

### SSH vs HTTPS
- **SSH**: 使用密钥对认证，更安全
- **HTTPS**: 使用用户名/密码或token认证

### 认证类型
- **Basic认证**: 用户名+密码的Base64编码
- **Bearer认证**: 使用token的Authorization头
- **SSH认证**: 使用公私钥对

## 10. 常用诊断命令

```bash
# 仓库状态
git status                    # 工作区状态
git log --oneline            # 提交历史
git remote -v                # 远程仓库配置

# 仓库健康
git fsck --full              # 完整性检查
git gc --aggressive          # 优化仓库

# 同步状态
git fetch                    # 获取远程更新
git log --graph --all        # 查看分支图

# 配置信息
git config --list           # 所有配置
git config user.name         # 特定配置
```

## 故障排除流程

1. **检查状态**: `git status`
2. **检查完整性**: `git fsck --full`
3. **清理优化**: `git gc --aggressive --prune=now`
4. **检查网络**: `ping github.com`
5. **检查认证**: `ssh -T git@github.com`
6. **重新尝试**: `git push`

记住：Git 是一个分布式版本控制系统，理解对象、引用和同步概念是掌握它的关键！

---
# 2. 获取远程变更（不会自动合并）
git fetch origin
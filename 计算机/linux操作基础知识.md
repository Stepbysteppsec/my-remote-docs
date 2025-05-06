
### Cmdlet
Cmdlet（读作 “command-let”，中文常称为 “命令行小工具”）是 PowerShell 的核心组成部分，是构建 PowerShell 环境的基本命令单元。

#### vim
ctrl-z 挂起 fg恢复
如果您想直接让一个命令从一开始就在后台运行，可以在命令末尾加上 `&` 符号，例如：

bash

```bash
apt-get download git &
```
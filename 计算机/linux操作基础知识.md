
### Cmdlet
Cmdlet（读作 “command-let”，中文常称为 “命令行小工具”）是 PowerShell 的核心组成部分，是构建 PowerShell 环境的基本命令单元。

#### vim
ctrl-z 挂起 fg恢复
一个命令从一开始就在后台运行，可以在命令末尾加上 `&` 符号，例如：
```bash
apt-get download git &
```



`systemctl` 是 Linux 中一个非常重要的命令行工具，它用于管理 `systemd` 系统和服务管理器。

在现代 Linux 发行版（如 Ubuntu、Debian、CentOS 7+、Fedora 等）中，`systemd` 已经取代了传统的 `SysVinit`或 `Upstart` 作为默认的 init 系统。`systemd` 负责在系统启动时启动各种服务、进程，并在系统运行期间管理它们。

`systemctl` 命令就是你与 `systemd` 交互的主要方式。它允许你执行各种任务，例如：

- **启动/停止/重启服务：**
    
    - `sudo systemctl start ssh`：立即启动 SSH 服务。
    - `sudo systemctl stop ssh`：立即停止 SSH 服务。
    - `sudo systemctl restart ssh`：重启 SSH 服务。
    - `sudo systemctl reload ssh`：如果服务支持，重新加载其配置文件而不完全重启。
- **启用/禁用服务（开机自启动）：**
    
    - `sudo systemctl enable ssh`：**设置 SSH 服务在系统启动时自动运行。** 这个命令通常会创建一个符号链接，将服务单元文件链接到 `systemd` 在启动时检查的目录中。它**不会**立即启动服务。
    - `sudo systemctl disable ssh`：**禁用 SSH 服务在系统启动时自动运行。** 这会删除之前创建的符号链接。
- **查看服务状态：**
    
    - `systemctl status ssh`：查看 SSH 服务的当前状态，包括是否正在运行、PID、内存使用情况、最近的日志等。这是一个非常有用的调试命令。
- **列出服务和单元：**
    
    - `systemctl list-units`：列出所有已加载的 `systemd` 单元（包括服务、挂载点、设备等）。
    - `systemctl list-unit-files --type=service`：列出所有可用的服务单元文件及其启用/禁用状态。
- **管理系统状态（targets）：**
    
    - `systemctl get-default`：获取默认的启动目标（runlevel 概念的 `systemd` 版本，例如 `multi-user.target` 代表命令行模式，`graphical.target` 代表图形界面模式）。
    - `sudo systemctl set-default graphical.target`：设置默认启动到图形界面。

**回到你提供的命令：**

1. `sudo apt update`：更新包列表。
    
2. `sudo apt install openssh-server`：安装 OpenSSH 服务器软件包。
    
3. `sudo systemctl enable ssh`：
    
    - **作用：** 将 OpenSSH 服务设置为在系统每次启动时自动启动。
    - **解释：** 这个命令告诉 `systemd` 在下次系统引导时启动 `ssh.service`。它通过在 `/etc/systemd/system/multi-user.target.wants/` 目录中为 `ssh.service` 创建一个符号链接来实现这一点。
    - **重要：** `enable` 命令本身不会立即启动服务。
4. `sudo systemctl start ssh`：
    
    - **作用：** 立即启动 OpenSSH 服务。
    - **解释：** 这个命令会立即启动 `ssh.service`，使其在当前会话中运行。

**总结 `enable` 和 `start` 的区别：**

- `systemctl enable <服务名称>`：**管开机自启动。** 确保服务在下次系统引导时自动启动。
- `systemctl start <服务名称>`：**管当前运行。** 立即启动服务，但在下次重启后可能不会自动启动（除非你同时使用了 `enable` 命令）。

因此，你通常会先 `enable` 一个你希望长期运行的服务，然后 `start` 它以使其立即生效，而无需重启系统。

为了更方便，`systemd` 也支持一个组合选项：`sudo systemctl enable --now ssh`，这个命令会同时启用服务并在当前会话中启动它。
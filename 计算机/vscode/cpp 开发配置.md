



1. `tasks.json`

- 主要负责编译任务
- 定义如何编译你的C++代码
- 指定编译器（如g++）
- 设置编译选项
- 决定输出的可执行文件名和位置

2. `launch.json`

- 负责调试配置
- 定义如何启动和调试程序
- 指定调试器（如gdb）
- 设置运行时的参数
- 配置断点和调试环境

具体步骤：

1. 打开项目文件夹
2. 按Ctrl+Shift+D打开"运行和调试"视图
3. 点击"创建launch.json文件"
4. 选择C++（GDB/LLDB）
5. VSCode会自动生成模板配置
6. 根据需要修改`launch.json`和`tasks.json`

示例`tasks.json`：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "shell",
            "label": "g++ build active file",
            "command": "/usr/bin/g++",
            "args": [
                "-g",
                "${file}",
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

示例`launch.json`：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++ - 生成和调试活动文件",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "g++ build active file"
        }
    ]
}
```

关于动态链接库：

- 在`tasks.json`中的`args`里可以添加`-l`参数链接库
- 例如 `-lpthread` 链接pthread库
- 库路径可以用 `-L` 选项指定

这两个文件的配置会帮你自动化编译和调试流程，让C++开发更便捷。




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



ctrl+shift +B 
- MSBuild：用于Visual Studio和C#/.NET项目
- Maven：Java项目的构建工具
- .NET Core：微软的跨平台开发框架
- Others：适合像C++这样使用g++/gcc编译的项

## tasks.json 常见选项解析

### 基本结构

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            // 任务配置...
        }
    ]
}
```

### 主要选项说明

1. **label** - 任务的名称，在 VS Code 界面中显示
    
    ```json
    "label": "build"
    ```
    
2. **type** - 任务类型，常见的有：
    
    - `shell`: 在shell中执行命令
    - `process`: 直接启动进程
    
    ```json
    "type": "shell"
    ```
    
3. **command** - 要执行的命令
    
    ```json
    "command": "g++"
    ```
    
4. **args** - 命令行参数，是一个字符串数组
    
    ```json
    "args": ["-g", "${file}", "-o", "${fileDirname}/${fileBasenameNoExtension}"]
    ```
    
5. **group** - 任务组，定义任务的类别和默认行为
    
    ```json
    "group": {
        "kind": "build",
        "isDefault": true
    }
    ```
    
    常见的组类型：
    
    - `build`: 构建任务
    - `test`: 测试任务
    - `none`: 不分类
6. **problemMatcher** - 用于从输出中解析问题（错误、警告）
    
    ```json
    "problemMatcher": ["$gcc"]
    ```
    
    常见的内置 matcher:
    
    - `$gcc`: GCC编译器
    - `$msCompile`: MSVC编译器
    - `$tsc`: TypeScript
    - `$eslint-compact`: ESLint
7. **presentation** - 控制任务输出的显示方式
    
    ```json
    "presentation": {
        "echo": true,
        "reveal": "always",
        "focus": false,
        "panel": "shared",
        "showReuseMessage": true,
        "clear": false
    }
    ```
    
8. **options** - 设置执行环境选项
    
    ```json
    "options": {
        "cwd": "${workspaceFolder}",
        "env": {
            "PATH": "${env:PATH}:/additional/path"
        }
    }
    ```
    
9. **dependsOn** - 定义任务依赖，会在当前任务执行前执行
    
    ```json
    "dependsOn": ["otherTaskLabel"]
    ```
    
10. **runOptions** - 运行选项
    
    ```json
    "runOptions": {
        "runOn": "folderOpen",
        "instanceLimit": 1
    }
    ```
    

### 常用变量

VS Code 提供了许多预定义变量用于任务配置：

- `${workspaceFolder}` - 工作区文件夹路径
- `${file}` - 当前打开的文件完整路径
- `${fileBasename}` - 当前文件的文件名（含扩展名）
- `${fileBasenameNoExtension}` - 当前文件名（不含扩展名）
- `${fileDirname}` - 当前文件所在的文件夹路径
- `${fileExtname}` - 当前文件的扩展名
- `${cwd}` - 任务启动时的工作目录
- `${env:NAME}` - 环境变量

### C++ 编译任务示例

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "编译C++",
            "type": "shell", 
            "command": "g++",
            "args": [
                "-std=c++17",
                "-Wall",
                "-g",
                "${file}",
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            },
            "problemMatcher": ["$gcc"]
        }
    ]
}
```

这个配置创建了一个默认构建任务，它使用g++编译器，带有C++17标准、调试信息和警告选项，编译当前打开的文件并输出到同一目录下。

希望这些信息对您有所帮助！如果您有任何特定的tasks.json配置需求，请随时告诉我。
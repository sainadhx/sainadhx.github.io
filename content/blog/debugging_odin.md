---
title: Debugging Odin programs using lldb
date: 2025-01-17
---


## Prerequisites
- Install [lldb](https://lldb.llvm.org/)

---

## Command Line Debugging
- Compile the odin program in debug mode.

`odin build . -debug -out:debug.bin`

- Start lldb on command line with the generated binary.

`lldb debug.bin`

---

## Using VSCode

## Setup
- Install the CodeLLDB VSCode extension from Extensions
- Create a `.vscode` folder at the root of your Odin project
- Copy `launch.json` and `tasks.json` provide below into it
- Goto Run (in menu) and click on **Start Debugging**

#### launch.json
```json
{
    "version": "1.0.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "preLaunchTask": "Build",
            "name": "Debug",
            "program": "${workspaceFolder}/build/debug.bin",
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

#### tasks.json
```json
{
    "version": "1.0.0",
    "command": "",
    "args": [],
    "tasks": [
        {
            "label": "clean_and_makedir",
            "type": "shell",
            "command": "rm -rf build/ && mkdir -p build",
        },
        {
            "label": "build",
            "type": "shell",
            "command": "odin build . -debug -out:build/debug.bin",
            "group": "build"
        },
        {
            "label": "Build",
            "dependsOn": [
                "clean_and_makedir",
                "build"
            ]
        }
    ]
}
```

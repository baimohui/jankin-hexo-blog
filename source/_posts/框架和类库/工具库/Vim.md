---
title: Vscode 搭配 Vim 指南
categories: 
- vim
tags:
- vim
---

参考文档：

1. [Vim 入坑指南](https://juejin.cn/post/6844903580952952845#heading-20)

2. [在 VSCode 里面配置 Vim 的正确姿势](https://zhuanlan.zhihu.com/p/188499395)


<!-- more -->

### settings.json 配置

```json
[
  "vim.insertModeKeyBindings": [
    // 输入模式下，将 esc 退出输入模式的功能绑定到双击 j 上
    {
      "before": [
        "j",
        "j"
      ],
      "after": [
        "<Esc>"
      ]
    }
  ],
  // 将空格键设置成 leader 键
  "vim.leader": "<space>",
  "vim.normalModeKeyBindings": [
    // 正常模式下，leader 键 + s 实现文件保存
    {
      "before": [
        "<leader>",
        "s"
      ],
      "commands": [
        ":w!"
      ]
    },
    // 正常模式下，leader 键 + q 实现退出
    {
      "before": [
        "<leader>",
        "q"
      ],
      "commands": [
        ":q!"
      ]
    },
    // 正常模式下，leader 键 + sq 实现文件保存并退出
    {
      "before": [
        "<leader>",
        "sq"
      ],
      "commands": [
        ":wq!"
      ]
    },
    // 正常模式下，leader 键 + u 实现 ctrl + u
    {
      "before": [
        "<leader>",
        "u"
      ],
      "after": [
        "<C-u>"
      ]
    },
    {
      "before": [
        "<leader>",
        "f"
      ],
      "after": [
        "<C-f>"
      ]
    },
    {
      "before": [
        "<leader>",
        "d"
      ],
      "after": [
        "<C-d>"
      ]
    },
    {
      "before": [
        "<leader>",
        "b"
      ],
      "after": [
        "<C-b>"
      ]
    },
  ],
  // 值设为 false，则取消 vim 对应点击事件的效果，默认使用 vscode 的
  "vim.handleKeys": {
    "<C-d>": true,
    "<C-s>": false,
    "<C-z>": false,
    "<C-a>": false,
    "<C-f>": false,
    "<C-n>": false,
    "<C-c>": false,
    "<C-l>": false,
    "<C-v>": false,
    "<C-x>": false,
  }
]
```

### keybindings.json
ctrl + shift + p / F1: 打开键盘快捷方式（JSON）
```json
// 将键绑定放在此文件中以覆盖默认值
[
  // 对 alt 键进行映射
  {
    "key": "alt+shift+down",
    "command": "editor.action.copyLinesDownAction",
    "when": "editorTextFocus && !editorReadonly"
  },
  {
    "key": "alt+shift+up",
    "command": "editor.action.copyLinesUpAction",
    "when": "editorTextFocus && !editorReadonly"
  },
]
```
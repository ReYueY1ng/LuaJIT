# LuaJIT for Mini World

[LuaJIT 2.1](https://luajit.org/) 的定制分支，用于《迷你世界》（Mini World）的 Lua 字节码逆向与
工具链。相对官方上游，唯一的功能性改动是**重排字节码操作码表（BCDEF）**，使本仓库的
`string.dump` / 字节码读写与游戏内嵌的 LuaJIT 保持一致。

> 《迷你世界》逆向工程主体项目是 **MNReverse**（私有仓库，不在 GitHub 公开）：世界存档、
> 区块 FlatBuffer、调色板编码、纹理/加密、Reflex 二进制协议，以及基于本仓库字节码布局的
> Lua 字节码 dump / 反编译 / 重编译。

## 这是什么

《迷你世界》客户端内嵌的 LuaJIT 并非原版：它带有私有导出（`lua_getAllocOwner` /
`lua_setAllocOwner` / `lua_GetStrByGCPtr` / `lua_myGetStack` 等），并且**重排了字节码操作码表**。
因此：

- 用官方 LuaJIT 生成的字节码（`.luac` / `string.dump`）无法被游戏加载；
- 游戏导出的字节码也无法被官方 LuaJIT 正确读取（操作码编号对不上）。

本分支在 `src/lj_bc.h` 中按游戏的实际顺序重排 `BCDEF`（opcode 编号随之改变），从而在
**字节码层面**与游戏兼容；除此之外其余代码与上游保持一致。

### 与上游的差异

`git diff upstream/<branch> <branch>` 只有一处：`src/lj_bc.h` 中 `BCDEF` 的条目顺序，
共 17 行位置调整：

- 常量指令 `KSTR / KCDATA / KSHORT / KNUM / KPRI / KNIL` 移到 `ISNUM` 之后、`MOV` 之前；
- 上值/函数指令 `UGET / USETV / USETS / USETN / USETP / UCLO / FNEW` 移到表指令之后。

该顺序不是推测：通过一个 Lua 脚本分别编译官方 LuaJIT 与游戏的字节码，逐项对比 `string.dump`
差异后确定。

## 分支说明

| 分支 | 上游基线 | 用途 |
|---|---|---|
| `v2.1` | 上游 `v2.1` 最新 | **默认分支**：持续跟随上游 + 迷你世界 BCDEF 补丁 |
| `old` | 上游 `631a45f7`（2023-08-28） | 与游戏内嵌版本基线对齐 + 同一 BCDEF 补丁 |
| `master` / `v2.0` | 上游同步 | 与上游一致，无迷你世界补丁 |

游戏内嵌 LuaJIT 版本鉴定结论：`LuaJIT 2.1.ROLLING`，`version_num = 20199`，
源码状态约等于上游 v2.1 提交 `748ab9d9`（2023-08-22）/ `631a45f7`（2023-08-28）——
即第一个 2.1 rolling 版本。详细报告见 MNReverse
`docs/RE/luajit_2.1_rolling_version.md`。`old` 分支正是以此为基线。

## 构建

### Windows（MSVC）

打开 “x64 Native Tools Command Prompt for VS”，然后：

```bat
cd src
msvcbuild.bat            :: 默认：x64 + GC64 动态发布版
msvcbuild.bat debug      :: 带调试符号
msvcbuild.bat amalg      :: amalgamated 单文件构建
msvcbuild.bat static     :: 静态库
```

产物：`src\luajit.exe`、`src\lua51.dll`、`src\lua51.lib`。

### Linux / macOS

```bash
make -j
```

## CI 与 Release

[`.github/workflows/build-release.yml`](.github/workflows/build-release.yml) 使用
GitHub Actions（`windows-latest` + MSVC），同时构建 `v2.1` 与 `old` 两个分支并发布为滚动
Release。每个 Release 同时提供打包 zip 与可直接下载的单个文件（含 **`lua51.dll`**、
`luajit.exe`、`lua51.lib`）：

| Release tag | 对应分支 |
|---|---|
| `v2.1-rolling` | `v2.1` |
| `old-rolling` | `old` |

触发方式：向 `v2.1` / `old` 分支 push，或在 Actions 页面手动 `workflow_dispatch`。

## 同步上游

```bash
git remote add upstream https://github.com/LuaJIT/LuaJIT.git   # 仅需一次
git fetch upstream
git checkout v2.1
git merge upstream/v2.1        # 与迷你世界 BCDEF 补丁合并
git push origin v2.1
```

## 许可

LuaJIT 版权归 Mike Pall 所有，以 MIT 许可证发布，详见 [COPYRIGHT](COPYRIGHT) 与
`src/luajit.h`。本仓库仅额外修改 `src/lj_bc.h` 的操作码顺序。

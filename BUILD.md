# SourceGit 编译指南

> 本文档由实际编译验证生成（2025-10-10），下次编译直接按此操作即可。

## 项目概况

- **SourceGit**：Avalonia UI + .NET 10 的开源 Git 图形客户端
- 目标框架：`net10.0`（`global.json` 锁定 SDK 10.0.0+）
- Release 配置默认启用 **Native AOT**（`PublishAot` + `PublishTrimmed`，见 `src/SourceGit.csproj`），产物为原生自包含 exe
- 官方 CI 参考：`.github/workflows/build.yml`

## 环境要求

| 依赖 | 要求 | 本机状态 |
|---|---|---|
| .NET SDK | 10.0.x 稳定版 | ✅ 10.0.401（`dotnet --list-sdks` 确认） |
| VS C++ 工具链 | link.exe（AOT 必需） | ✅ 已装 |
| git submodule | depends/AvaloniaEdit | ⚠️ 首次必须初始化 |

## 编译步骤

### 1. 初始化子模块（首次/拉取新代码后必须）

```bash
git submodule update --init --recursive
```

> `depends/AvaloniaEdit` 是子模块，为空时编译直接失败。

### 2. Release 编译（验证用，可跳过）

```bash
dotnet build -c Release -maxcpucount:4
```

### 3. 生成 Release 64 位 exe（官方 CI 同款命令）

```bash
dotnet publish src/SourceGit.csproj -c Release -r win-x64 -o publish
```

其他 RID：`win-arm64` / `osx-x64` / `osx-arm64` / `linux-x64` / `linux-arm64`。

## ⚠️ 资源占用限制（重要）

编译必须限制资源，两层措施：

1. **`-maxcpucount:4`**：MSBuild 最多 4 个并行任务
2. **BelowNormal 进程优先级**：编译链（msbuild/csc/AOT 编译器）只用空闲 CPU，不抢前台资源（Windows 下子进程继承父进程优先级）

**Git Bash 里的完整命令**（注意整个 PS 命令用单引号包住，防止 `$p` 被 bash 展开）：

```bash
# 编译
powershell -NoProfile -Command '$p = Start-Process -FilePath "dotnet" -ArgumentList "build","-c","Release","-maxcpucount:4","--nologo" -WorkingDirectory "<项目路径>" -NoNewWindow -PassThru; $p.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::BelowNormal; $p.WaitForExit(); exit $p.ExitCode'

# 发布（生成 exe）
powershell -NoProfile -Command '$p = Start-Process -FilePath "dotnet" -ArgumentList "publish","src/SourceGit.csproj","-c","Release","-r","win-x64","-o","publish","--nologo","-maxcpucount:4" -WorkingDirectory "<项目路径>" -NoNewWindow -PassThru; $p.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::BelowNormal; $p.WaitForExit(); exit $p.ExitCode'
```

## 产物

- 路径：`publish/SourceGit.exe`（PE32+ x86-64 原生程序，约 55 MB）
- 自包含，目标机器**无需安装 .NET 运行时**
- `av_libglesv2.dll`、`libSkiaSharp.dll`、`libHarfBuzzSharp.dll`、`libonigwrap.dll` 是 Avalonia 渲染依赖，分发时须与 exe 同目录
- `*.pdb` 仅调试用，可删除（官方打包脚本就是直接删 pdb 后压 zip）

## 常见问题

| 现象 | 处理 |
|---|---|
| `fatal: No names found, cannot describe anything` | 忽略。仓库无 git tag，仅影响"友好版本号"，csproj 已 `IgnoreExitCode`。想要完整版本号：`git fetch --tags` |
| AOT 编译失败（缺 VS C++ 工具链的机器） | 退回托管模式发布：追加 `-p:DisableAOT=true` |
| 找不到 AvaloniaEdit 项目 | 子模块没初始化，见步骤 1 |

## 实测数据（本机验证）

- 子模块：`ddf00a91`（love-linger/AvaloniaEdit）
- `dotnet build -c Release`：0 警告 0 错误，约 1 分 07 秒
- `dotnet publish -r win-x64`：成功，publish 目录约 221 MB（含 pdb）
- 产物校验：`PE32+ executable for MS Windows 6.00 (GUI), x86-64` ✅

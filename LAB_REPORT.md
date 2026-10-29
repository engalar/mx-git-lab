# mx-git-lab — mxcli git 与 Studio Pro 提交逻辑对照实验

目的：验证 `mxcli git commit` / `mxcli git doctor` 能否复现 Studio Pro 的提交逻辑，以及两支合并与原生的异同。
方法：每个场景先取 Studio Pro 的"金样"（golden），再以 mxcli 复刻同形变更，逐项对比文件集、mx_metadata note 结构、本地配置与远端 notes ref。Studio Pro 版本 **11.12.6**。

## S0 — Studio Pro 金样（已完成）

操作：`mxcli new mx-git-lab --version 11.12.6` 建项；推送 GitHub（`engalar/mx-git-lab`，HTTPS）；以 Studio Pro 打开（Version Control 指向该 remote），在 MyFirstModule 的 Domain Model 上作一改动并 Commit（`629ac8c commit by sp`）。

### 金样证据

初始上传提交（46c3836 `Initial app upload.`）的 note：

```json
{"BranchName":"","ModelerVersion":"11.12.6","ModelChanges":[],"RelatedStories":[],"SolutionVersion":"","MPRFormatVersion":"Version2","HasModelerVersion":true}
```

Studio Pro 提交（629ac8c `commit by sp`，DomainModel 被改）的 note：

```json
{"BranchName":"","ModelerVersion":"11.12.6","ModelChanges":[{"Status":"Modified","UnitID":"84776610-13b2-48cb-a265-d5e984d63129","UnitType":"DomainModels$DomainModel","UnitName":"","Module":"MyFirstModule"}],"RelatedStories":[],"SolutionVersion":"","MPRFormatVersion":"Version2","HasModelerVersion":true}
```

该提交的文件集（恰二件）：

```
M	mprcontents/84/77/84776610-13b2-48cb-a265-d5e984d63129.mxunit
M	mx-git-lab.mpr
```

本地 git 配置（Studio Pro 写入）：

```
core.autocrlf=false
mendix.commits-since-gc=2
mendix.lineendingresetdone=true
```

远端：`refs/notes/mx_metadata` **由 Studio Pro 自行推送**（ls-remote 可见）。

### S0 发现（已反哺 mxcli）

- **F1**：`mendix.commits-since-gc` 在健康仓库中是 Studio Pro 的计数器（=2 而非 0）——`git doctor` 不再判其值，仅缺失时补 0（mxcli#1670 之 6a5c942fa）。
- **F2**：`lineendingresetdone`/`autocrlf` 的值小写——doctor 的比较不区分大小写，实测通过。
- **F3**：初始上传提交的 note `ModelChanges:[]`（全空），而 `git note` 对根提交会列全部单元为 Added——差异记录在案（S5 前不影响日常提交场景）。

### doctor 复检（6a5c942fa）

```
$ mxcli git doctor -p mx-git-lab.mpr
Git local config
  [x] core.autocrlf = false
  [x] mendix.commits-since-gc = 2
  [x] mendix.lineEndingResetDone = true
Remote
  [x] origin: https://github.com/engalar/mx-git-lab.git (HTTPS)
  [x] origin has refs/notes/mx_metadata
mx_metadata notes
Nothing to fix.   (exit 0)
```

## 场景一览

| # | 场景 | 状态 |
|---|------|------|
| S0 | Studio Pro 金样 | ✅ 完成 |
| S1 | mxcli 复刻模型变更 + `git commit` | 进行中 |
| S2 | 杂物文件不被提交 | 待办 |
| S3 | 哈希漂移自修 | 待办 |
| S4 | 裸提交占位符 → doctor 补齐 | 待办 |
| S5 | 两支合并：Studio Pro 原生 vs git | 待办 |

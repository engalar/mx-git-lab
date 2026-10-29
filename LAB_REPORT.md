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
| S1 | mxcli 复刻模型变更 + `git commit` | ✅ 完成（note 与金样逐字节全同） |
| S2 | 杂物文件不被提交 | ✅ 完成（35 件皆剔，提交恰二件） |
| S3 | 哈希漂移自修 | ✅ 完成 |
| S4 | 裸提交占位符 → doctor 补齐 | ✅ 完成 |
| S5 | 两支合并：Studio Pro 原生 vs git | 进行中 |

## S4 — 裸提交与占位符（c749007 → doctor 补齐）

操作：裸 git 提交（`c749007`，无 note，仅改 LAB_REPORT.md）并推送；Studio Pro 11.12.6 开项 fetch 之后观其史。

**实测（11.12.6）**：

- Studio Pro **不写占位符 note**（notes 仍 7 条）——占位符乃 11.13 之行为；
- 历史面板对无 note 之提交显示 `Changes in model(0)`, `Changes on disk(1)`——11.12.6 径读提交树，不赖 note；
- `mxcli git doctor -p --fix` 检得 `c7490076eb without a note`，补以计算之 note（`ModelChanges:[]`，`ModelerVersion:"11.12.6"`——与 Studio Pro 自书之空变更 note 同构）。

**判定 S4：11.12.6 无占位符之患；note 缺失由 doctor 一令补齐。**（对 11.13+，占位符会被 `git note`/`doctor` 自动替换，此路径已有测试。）

**附**：lab 之 `.gitignore` 乃 Studio Pro 自生（无 `.mxcli/`）；`mxcli init` 模板固有之——已补入 lab（`ca4f3e7`）。

## S5 — 两支合并（git 侧已完成，原生侧待行）

布局：`feature/s5-attribute`（bd355da：`ALTER ENTITY CreatedByMxcli add Nickname`，note 之 `BranchName:"feature/s5-attribute"` 正确）vs `main`（480879a：新建 `MainOnly` 实体）。两侧皆改**同一** DomainModel 单元（`84776610`）。

### git 原生 merge 之实测

```
$ git merge feature/s5-attribute
warning: Cannot merge binary files: mprcontents/84/77/84776610-….mxunit (HEAD vs. feature/s5-attribute)
CONFLICT (content): Merge conflict in mprcontents/84/77/84776610-….mxunit
warning: Cannot merge binary files: mx-git-lab.mpr (HEAD vs. feature/s5-attribute)
CONFLICT (content): Merge conflict in mx-git-lab.mpr
Automatic merge failed; fix conflicts and then commit the result.
```

- git 将 `.mxunit`（BSON）与 `.mpr`（SQLite）皆视为**二进制**：凡两侧同触一单元，必 `UU` 冲突，无自动合并可言；
- 此时 `mxcli git commit` **拒**：`refusing to commit: the merge has unresolved conflicts …`（`git ls-files --unmerged` 可列）；
- 已 `git merge --abort`，`gitmerge` 试验支已除，main 完好（480879a）。

**git 侧判定：裸 git 不能合并 Mendix 模型——此非 mxcli 可补，乃格式之性。**

### 原生金样（c141f6f `Merge commit(auto) by sp`）

双亲（`e2adea6`+`bd355da`）真合并提交；note 列 DomainModel Modified 一条，与常例同构。领域模型双侧之变并存。

### mxcli CLI 模型感知合并（60a9e00，经 `mx git-merge`）

`mx.exe` 自带 `git-merge BASE MINE THEIRS BASE_COMMIT MINE_COMMIT THEIRS_COMMIT`（`-v` 尝试自解）。于同局重演（`gitmerge` 支自 480879a）：

```
$ mx git-merge -v <base>.mpr <mine>.mpr <theirs>.pr 63b65706 480879a3 bd355da4
[{"UnitId":"84776610-…","Type":"MergeContents","TreeConflict":"None","ConflictResolutionDetails":"AutoResolved"}]
Merge successful.  (exit 0)
```

工作树遂含双侧之变（`CreatedByMxcli(FullName,Nickname)` + `MainOnly`）。`git add` 后以 `mxcli git commit` 结之：双亲合并提交 `60a9e00`，note 与原生金样**同构**。

### 真冲突之实测（两侧同改 FullName 之长）

```
[{"…","TreeConflict":"None","ConflictResolutionDetails":"ManualResolutionRequired"}]
Conflicts found during merging. Please resolve them by opening the project in Studio Pro.
exit 2
```

- mx 之败不损我侧模型（featA 之 300 仍在），唯留 `.mpr` 索引 1 mismatch；
- 故 `mxcli git merge` 之 UX 定为：exit 2 即 `git merge --abort` + 列冲突单元 + 指向 Studio Pro 解之。

**S5 终判**：

| | git merge | Studio Pro 原生 | mxcli（`mx git-merge` 之助） |
|---|---|---|---|
| 可自动合并 | 二进制冲突，**不可** | ✅ 模型感知 | ✅ 同（AutoResolved） |
| 真冲突 | 二进制冲突 | 冲突 UI 手解 | 报冲突清单，转 Studio Pro |
| note/提交规范 | — | ✅ | ✅（与原生同构） |

`mxcli git merge` 之实现随后入 mxcli（#1670）。

## S1+S2 — mxcli 复刻金样 + 杂物剔除（53f08dd）

操作：`mxcli exec` 建 `MyFirstModule.CreatedByMxcli (FullName: String(200))`（与金样同形单元：DomainModel）；以 `git add -f` 强置 `project-settings.user.json` 入索引；`mxcli git commit -m … -p mx-git-lab.mpr`。

结果：

- 提交文件集**恰二件**，与金样同构：
  ```
  M	mprcontents/84/77/84776610-13b2-48cb-a265-d5e984d63129.mxunit
  M	mx-git-lab.mpr
  ```
- note 与金样（629ac8c）**逐字节全同**（同一 UnitID，同键序同紧凑）：
  ```json
  {"BranchName":"","ModelerVersion":"11.12.6","ModelChanges":[{"Status":"Modified","UnitID":"84776610-13b2-48cb-a265-d5e984d63129","UnitType":"DomainModels$DomainModel","UnitName":"","Module":"MyFirstModule"}],"RelatedStories":[],"SolutionVersion":"","MPRFormatVersion":"Version2","HasModelerVersion":true}
  ```
- `git add -f` 强置之杂物与 `.mxcli/` 等 35 件**皆未入提交**（仍在盘上）。

**判定 S1：完美复现。**（对同一变更，`mxcli git commit` 之产物与 Studio Pro 不可分辨。）

**F4（已修）**：陈列表误列已剔之件（显示之疵，提交无染）——aace3305c 修复，附测试。

## S3 — 哈希漂移（1054a91）

操作：`git checkout 629ac8c -- <DomainModel.mxunit>`（经典"checkout 还原单元文件"之漂：索引言新，文件言旧）→ `mxcli git commit`。

结果：

```
1 mismatch(es) …  [DomainModels$DomainModel]
Repaired 1 ContentsHash value(s) from the files on disk.
Committed 1054a910d6 … 1 model change(s)
```

- 提交自动包含：还原后之 `.mxunit` + 修复后之 `.mpr`（文件即真相，索引随之）。
- 提交后 `fix hashes` 复验：0 mismatch。
- 随后重建 `CreatedByMxcli` 实体（53944c4），模型归于齐整。

**判定 S3：漂移自修，与 `fix hashes --repair` 之语义一致，提交未被漂移所阻亦未带病而入。**

## S4 bait

A bare-git commit with NO mx_metadata note follows this line. Studio Pro 11.12.6: does it back-fill a placeholder on fetch?

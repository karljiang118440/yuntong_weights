# 云通 RT-DETR-L 模型权重版本管理与 Release 发布操作手册

编写日期：2026-10-23（北京时间）
适用目录：`D:\git\TH_code-main\TH_code`
GitHub 用户名：`karljiang118440`
模型来源：`yuntong_weights/yuntong_rtdetr_l_20261021`

> 本文件是针对本机现状编写的执行手册。本次仅落地此 Markdown；尚未初始化仓库、修改 Git 配置或忽略规则、提交权重、创建远程仓库、打包或发布。下文命令需按顺序执行，并替换明确标出的实际信息。按 GitHub 场景编写，尚未核实你的远程服务与仓库名称。

## 1. 推荐的实际管理场景

算法工程师完成一轮训练 → 在固定验证集上选出候选权重 → 提交模型说明、配置及经验证的权重 → 建立模型专属 Tag → 打包并检查 SHA256 → 上传 Release 草稿 → 下载复核 → 正式发布 → 部署端按 Tag 和校验值更新或回滚。

采用以下分工：

| 内容 | 管理方式 | 原因 |
| --- | --- | --- |
| 源码、类别表、训练配置、模型说明、评估记录、发布说明 | 普通 Git | 可比较、可审阅、可追溯 |
| 明确选定的发布权重 `.pt`，以后需要的 `.onnx` | Git LFS | 避免把大二进制塞进 Git 对象历史 |
| 各个 epoch 检查点、临时下载文件、训练批次图片 | 本地或内部训练归档 | 不让每次训练消耗大量 LFS 存储 |
| 固定版本 ZIP、ZIP 的 SHA256 校验文件 | GitHub Release 附件 | 部署端可直接下载，无须克隆代码和安装 LFS |

本项目建议“代码与发布模型在同一个仓库内管理”，不在权重子目录中创建第二个 `.git`。Git Tag 标记整个仓库的提交，使用 `yuntong-rtdetr-l-v1.0.0` 区分模型版本与其他算法版本。

GitHub 普通 Git 会拒绝超过 100 MiB 的文件；大权重应使用 LFS。[官方文件大小说明](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)

LFS 存储真实二进制，Git 提交存储指针。每次权重变化通常增加一个新 LFS 对象；LFS 与 Release 不是无限免费存储，发布前查看实际账户配额和计费设置。[Git LFS 说明](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-git-large-file-storage)

## 2. 本机检查结果与首次发布决策

2026-10-23 实际读取文件，发现：

- 项目目录及其至 D 盘根目录的各级父目录未发现 `.git`，因此当前尚不是本地 Git 工作副本；文件夹名称不能证明其远程地址或历史分支。
- 当前终端的 PATH 中找不到 `git`、`git-lfs` 和 `gh`；未证明整台电脑未安装这些工具。
- `.gitignore` 以 `*` 开头，采取默认全部忽略的白名单策略；`.pt`、`.gitattributes`、本手册、JSON 和许多说明文件需要定向放行。
- 模型目录共有 22 个文件，合计 2,047,099,735 字节，约 1.91 GiB。
- 根目录 `readme.md` 里有遗留合并冲突标记；不要直接把整个项目一次性提交。

| 文件 | 实测大小 | 本次建议 |
| --- | ---: | --- |
| `weights/last.pt` | 66,141,013 字节，约 63.08 MiB | 唯一建议先评估的发布候选 |
| `weights/epoch0.pt` 至 `epoch80.pt`，每隔 10 个 epoch | 共 9 个，每个约 188.75 MiB | 保留训练归档，不纳入首次发布 |
| `weights/epoch90.pt.filepart` | 0 字节 | 未完成文件，不上传、不打包 |
| `weights/best.pt` | 未发现 | 不能在命令中假定它存在 |
| PNG/JPG 图像 | 10 个 | 仅人工审核后选取评估图；训练样本图不默认发布 |

`last.pt` 实测 SHA256：

```text
7de766cf54460b3e52270497e4184e96499b298b361c537c4ccc3e0f1da69d14
```

文件名 `last.pt` 不意味着最佳精度，也不能单凭文件大小认定它能部署。首次发布应先加载并在业务验证集上测试；如果最终选用另一个文件，要同步调整下面的跟踪路径、模型卡及打包脚本。

**场景示例：** 云通检测服务将这轮训练作为 `v1.0.0` 候选；评估通过后发布，部署端锁定版本。日期 `20261021` 表示训练批次，`v1.0.0` 表示交付版本，两者不混用。

## 3. 全部操作限定在当前项目

### 3.1 前置准备

需要可用的 Git for Windows 和 Git LFS。严格不改其他目录时，可预先在本项目 `.tools/` 中准备官方便携工具，并在当前 PowerShell 会话中临时将其可执行目录加入 PATH；不要为了本项目修改系统 PATH。若已有可用安装，使用现有程序即可。`.tools/` 不提交，也不进入模型包。

下载入口：[Git for Windows](https://gitforwindows.org/)、[Git LFS](https://git-lfs.com/)。工具安装本身与项目 Git 配置是两件事；本手册不要求进行系统级安装。

以下所有命令在 PowerShell 中运行。原生命令失败后立即停止，不要继续后面的提交或发布步骤。

```powershell
Set-Location -LiteralPath 'D:\git\TH_code-main\TH_code'
$ErrorActionPreference = 'Stop'
git --version
if ($LASTEXITCODE -ne 0) { throw 'Git 不可用' }
git lfs version
if ($LASTEXITCODE -ne 0) { throw 'Git LFS 不可用' }
```

不要执行 `git config --global`、`git lfs install`（不带 `--local`），也不要执行 `gh auth setup-git`。项目内工具目录、包和临时文件均保留在当前项目内。

### 3.2 先确认是新仓库还是已有远程仓库的代码副本

目前没有 `.git`，有两条不同路径，必须根据真实情况选择：

**A. 确实准备新建仓库：** 在 GitHub 网页用指定账户新建一个空仓库，建议先选 Private；不要勾选自动生成 README、License 或 `.gitignore`。仓库名例如 `TH_code`，但这只是建议，不代表远程已经存在。然后执行：

```powershell
git init -b main
if ($LASTEXITCODE -ne 0) { throw '初始化失败' }
```

**B. 此目录其实是已有仓库下载的源码 ZIP：** 不要建立一段无关的新历史后强推。先备份本地改动，取得实际远程 URL 和目标分支；可在本项目的临时子目录克隆原仓库，再经核对恢复其 Git 历史。此路径需要结合实际远程历史处理，不照搬下面“空远程首次推送”的命令，也不使用 `--allow-unrelated-histories` 或强推解决。完成后确认仓库根仍是本项目目录，再继续。

本手册后续以 A“新建空仓库”为完整示例。

### 3.3 设置仅对本仓库生效的提交身份与 LFS

```powershell
$repoRoot = (git rev-parse --show-toplevel).Trim()
if ($LASTEXITCODE -ne 0) { throw '当前不是 Git 仓库' }
if ([IO.Path]::GetFullPath($repoRoot) -ne [IO.Path]::GetFullPath((Get-Location).Path)) {
    throw '仓库根目录不是当前项目，停止配置'
}
git config --local user.name 'karljiang118440'
$email = Read-Host '输入 GitHub Settings > Emails 中确认的邮箱或准确的 noreply 邮箱'
if ([string]::IsNullOrWhiteSpace($email)) { throw '邮箱不能为空' }
git config --local user.email $email
git lfs install --local
if ($LASTEXITCODE -ne 0) { throw 'LFS 本地初始化失败' }
git config --local --get user.name
git config --local --get user.email
git config --local --get filter.lfs.process
```

`user.name` 是提交署名，不是登录凭证；邮箱不要根据用户名猜测。[LFS 安装参数说明](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-install.adoc)

### 3.4 登录只影响本项目的做法

GitHub HTTPS Git 操作使用 PAT 或凭据管理器，不使用账户登录密码。[GitHub 认证说明](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)

为严格避免更改其他项目的身份和认证配置，本手册默认：每次需要联网的 Git 命令临时禁用继承的 credential helper，交互输入用户名与 PAT，不把 PAT 保存到项目文件中：

```powershell
# 以后 push/fetch 均使用这个形式；这里先不执行 push。
# git -c credential.helper= push ...
```

在安全的交互终端按提示输入 `karljiang118440`；Password 提示中输入 PAT。PAT 在 GitHub 网页生成，限制到目标仓库并授予执行任务所需的内容读写权限。不要把 PAT 或登录密码写入远程 URL、脚本、MD、环境配置文件或聊天。

如果使用 Git Credential Manager，可单独为此仓库配置 helper 与 `credential.useHttpPath=true`，但它可能在 Windows 凭据库保存凭据；不属于严格“仅项目目录保存配置”的默认路径。本手册不执行该方案。

此前已在聊天中披露的密码建议在对应平台更换；本手册不保存也不使用它。

## 4. 建议目录与模型说明

```text
TH_code/
├─ .git/                              # 初始化后生成，本地元数据
├─ .gitignore
├─ .gitattributes                     # LFS 路径规则，必须提交
├─ 模型权重版本管理与Release发布流程.md
├─ yuntong_weights/
│  └─ yuntong_rtdetr_l_20261021/
│     ├─ MODEL_CARD.md                # 模型用途、约束、指标和兼容性
│     ├─ RELEASE_NOTES.md             # 人工核实的交付说明
│     ├─ requirements-inference.txt   # 实际验证过的推理依赖
│     ├─ classes.json                 # 完整且顺序准确的类别映射
│     ├─ inference.yaml               # 输入尺寸、阈值、归一化等
│     ├─ training_metadata.json       # 训练追溯信息
│     └─ weights/
│        ├─ last.pt                   # 首次发布候选，LFS
│        ├─ epoch*.pt                 # 本地训练归档，不提交
│        └─ *.filepart                # 未完成文件，不提交
└─ dist/                              # 本地构建产物，不提交
```

上述配套文件目前尚未发现，需要发布前补齐，不能用猜测的类别、指标或依赖版本充数。

`MODEL_CARD.md` 至少记录：模型标识、训练批次、用途、来源权重、SHA256、类别顺序、输入尺寸/颜色顺序/预处理、后处理阈值、验证集标识与版本、mAP50/mAP50-95/Precision/Recall、硬件与批量/延迟测试方法、Python/PyTorch/框架版本、许可证与使用范围、已知不足、验收人和日期。指标必须来源于实际评估，不根据曲线图片臆测。

`training_metadata.json` 至少记录训练代码仓库与提交、训练命令、配置、数据集版本及划分、随机种子、基础预训练权重标识、训练时间和训练环境。无法追回的历史信息明确填写 `unknown` 并说明原因；当前新仓库的提交不能冒充原训练代码提交。

`requirements-inference.txt` 使用实际验证环境中需要的精确版本；不要直接复制任意训练机全部依赖。`classes.json` 必须与模型输出索引一致。`inference.yaml` 记录部署端实际读取的键和约定。若项目推理需要自定义模块，另补实际推理入口与代码获取方式；仅 `.pt` 和依赖列表不能保证独立运行。

**发布门槛：** 在干净环境中，按已记录步骤加载该文件，对固定样例完成推理，确认输出类别、阈值、精度、速度及接口兼容性；结果记录进模型卡。PyTorch 权重仅在可信来源及相应框架环境加载，不通过盲目反序列化未知文件来检查格式。

## 5. 定向修改白名单与启用 LFS

### 5.1 在项目根 `.gitignore` 最后追加

不要删除已有规则。以下内容追加在现有文件的最末尾，确保覆盖上方 `*`，同时不把其他权重意外放行：

```gitignore
# === 云通模型发布：精准白名单 ===
!/.gitattributes
!/模型权重版本管理与Release发布流程.md
!/yuntong_weights/yuntong_rtdetr_l_20261021/MODEL_CARD.md
!/yuntong_weights/yuntong_rtdetr_l_20261021/RELEASE_NOTES.md
!/yuntong_weights/yuntong_rtdetr_l_20261021/requirements-inference.txt
!/yuntong_weights/yuntong_rtdetr_l_20261021/classes.json
!/yuntong_weights/yuntong_rtdetr_l_20261021/inference.yaml
!/yuntong_weights/yuntong_rtdetr_l_20261021/training_metadata.json
!/yuntong_weights/yuntong_rtdetr_l_20261021/weights/last.pt

# 本地构建、工具、其他检查点和未完成文件不提交
/dist/
/.tools/
/yuntong_weights/yuntong_rtdetr_l_20261021/weights/epoch*.pt
*.filepart
```

原文件已有 `!*/`，允许继续遍历目录，因此这里可精确放行文件。若父目录下还有其他 `.gitignore` 或全局规则，通过 `git check-ignore -v` 核查。

### 5.2 仅跟踪选定权重

```powershell
$model = 'yuntong_weights/yuntong_rtdetr_l_20261021'
git lfs track "$model/weights/last.pt"
if ($LASTEXITCODE -ne 0) { throw 'LFS 跟踪失败' }
Get-Content -LiteralPath '.gitattributes'
git check-attr filter diff merge -- "$model/weights/last.pt"
git check-ignore -v -- "$model/weights/epoch0.pt" "$model/weights/epoch90.pt.filepart"
```

`.gitattributes` 应有以下一行，并保留文件已有的其他规则：

```gitattributes
yuntong_weights/yuntong_rtdetr_l_20261021/weights/last.pt filter=lfs diff=lfs merge=lfs -text
```

不要用 `git add -f` 绕过规则。新模型批次或新增发布文件必须同时增加白名单与 LFS 路径，再明确添加；仅做 `git lfs track` 不会自动解除忽略。

如果权重已经被普通 Git 提交，LFS 跟踪不自动转换历史。这时先检查历史，再制定迁移方案；`git lfs migrate import` 会重写提交标识，不在共享分支上直接运行或强推。当前目录尚无历史，因此本次不需要迁移。

## 6. 创建可审阅的首次模型提交

先补齐第 4 节的文件并完成评估，再执行显式添加，避免误收业务代码、图片和训练检查点：

```powershell
$model = 'yuntong_weights/yuntong_rtdetr_l_20261021'
git add -- '.gitignore' '.gitattributes' '模型权重版本管理与Release发布流程.md' `
    "$model/MODEL_CARD.md" "$model/RELEASE_NOTES.md" `
    "$model/requirements-inference.txt" "$model/classes.json" `
    "$model/inference.yaml" "$model/training_metadata.json" "$model/weights/last.pt"
if ($LASTEXITCODE -ne 0) { throw '暂存失败，检查缺失文件或忽略规则' }
git status --short
git diff --cached --stat
git diff --cached --check
if ($LASTEXITCODE -ne 0) { throw '差异检查未通过' }
git show ':yuntong_weights/yuntong_rtdetr_l_20261021/weights/last.pt'
git lfs status
git lfs ls-files --long
```

`git show` 的输出应是很短的 LFS 指针，含 `version`、`oid sha256:` 和 `size 66141013`，而不是二进制。该候选文件未变化时，oid 应等于第 2 节实测 SHA256；这证明暂存的是 LFS 指针并对应正确权重。

```powershell
git commit -m 'model(yuntong): add validated RT-DETR-L v1.0.0 weights and metadata'
if ($LASTEXITCODE -ne 0) { throw '提交失败' }
git lfs fsck
if ($LASTEXITCODE -ne 0) { throw 'LFS 完整性检查失败' }
git status --short
```

若当前模型需依赖本仓库源码，还需在此版本中提交审阅过的实际推理代码或记录可取得的准确代码提交。不要让一个只含权重的初始提交被误认为包含完整业务运行代码。

## 7. 关联远程仓库、上传并建立模型 Tag

以下仅适用于已在网页创建的“空远程”。用户名已明确，仓库名/组织归属仍需使用真实值，不猜测。

```powershell
$remoteUrl = Read-Host '输入实际空仓库 HTTPS URL（不可含 token 或密码）'
if ($remoteUrl -notmatch '^https://github\.com/[^/]+/[^/]+(?:\.git)?$' -or $remoteUrl.Contains('@')) {
    throw '请使用不含凭据的实际 GitHub HTTPS 仓库地址'
}
git remote add origin $remoteUrl
if ($LASTEXITCODE -ne 0) { throw '添加远程失败；已有 origin 时先核实，不直接覆盖' }
git remote -v
git -c credential.helper= push -u origin main
if ($LASTEXITCODE -ne 0) { throw '分支或 LFS 上传失败，停止发布' }
```

正常 push 时 LFS 的 pre-push 钩子上传相应对象。若配额、权限或网络报错，先解决后再继续；普通 Git 提交存在不代表远程 LFS 对象可下载。

```powershell
$tag = 'yuntong-rtdetr-l-v1.0.0'
git tag -a $tag -m 'Yuntong RT-DETR-L v1.0.0; training run 20261021'
if ($LASTEXITCODE -ne 0) { throw '创建 Tag 失败；已有版本不要覆盖' }
git -c credential.helper= push origin $tag
if ($LASTEXITCODE -ne 0) { throw 'Tag 上传失败，停止发布' }
git rev-parse HEAD
git rev-list -n 1 $tag
```

两次提交标识应一致。Tag 一旦对外使用，不移动、不覆盖；模型变化使用新版本。

## 8. 在项目内部打包可校验的 Release

完成上面的提交与 Tag 后，在同一项目根目录、同一个 PowerShell 中执行。该脚本只复制一个经选定的权重及已提交配套文件，不包括训练样本图和 epoch 文件；如果你的推理还需额外文件，明确补充白名单、提交与打包清单。

下面命令中的变量在本节重新定义，不依赖前文会话。打包目录已存在时停止，避免把上次残留文件带入新包。

```powershell
$ErrorActionPreference = 'Stop'
Set-Location -LiteralPath 'D:\git\TH_code-main\TH_code'
$model = 'yuntong_weights/yuntong_rtdetr_l_20261021'
$tag = 'yuntong-rtdetr-l-v1.0.0'
$version = '1.0.0'

$dirty = @(git status --porcelain)
if ($LASTEXITCODE -ne 0 -or $dirty.Count -gt 0) { throw '先处理未提交修改' }
$commit = (git rev-parse HEAD).Trim()
if ($LASTEXITCODE -ne 0) { throw '无法读取 HEAD' }
$tagCommit = (git rev-list -n 1 $tag).Trim()
if ($LASTEXITCODE -ne 0 -or $commit -ne $tagCommit) { throw 'HEAD 不对应发布 Tag' }
git lfs fsck
if ($LASTEXITCODE -ne 0) { throw 'LFS 校验失败' }

$weight = Join-Path $model 'weights/last.pt'
$weightHash = (Get-FileHash -LiteralPath $weight -Algorithm SHA256).Hash.ToLowerInvariant()
$pointer = @(git show "${tag}:$model/weights/last.pt")
if ($LASTEXITCODE -ne 0 -or $pointer -notcontains "oid sha256:$weightHash") {
    throw '本地真实权重与 Tag 中 LFS 指针不一致，先执行 git lfs pull 并核查'
}
$weightBytes = (Get-Item -LiteralPath $weight).Length
if ($pointer -notcontains "size $weightBytes") { throw '权重大小与 LFS 指针不符' }

$required = @('MODEL_CARD.md','RELEASE_NOTES.md','requirements-inference.txt',
              'classes.json','inference.yaml','training_metadata.json')
foreach ($name in $required) {
    if (-not (Test-Path -LiteralPath (Join-Path $model $name) -PathType Leaf)) {
        throw "缺少发布配套文件：$name"
    }
}
$pkg = Join-Path 'dist' $tag
$zip = Join-Path 'dist' "$tag.zip"
$checksumFile = Join-Path 'dist' "$tag.zip.sha256"
foreach ($p in @($pkg,$zip,$checksumFile)) {
    if (Test-Path -LiteralPath $p) { throw "产物已存在，停止覆盖：$p" }
}
New-Item -ItemType Directory -Path (Join-Path $pkg 'weights') -Force | Out-Null
Copy-Item -LiteralPath $weight -Destination (Join-Path $pkg 'weights/last.pt')
foreach ($name in $required) {
    Copy-Item -LiteralPath (Join-Path $model $name) -Destination (Join-Path $pkg $name)
}

# 先验证复制后的权重，发现训练程序仍在改写文件时立即停止。
$packagedHash = (Get-FileHash -LiteralPath (Join-Path $pkg 'weights/last.pt') -Algorithm SHA256).Hash.ToLowerInvariant()
if ($packagedHash -ne $weightHash) { throw '复制后权重发生变化，停止打包' }
$manifest = [ordered]@{
    model_id = 'yuntong_rtdetr_l'
    version = $version
    tag = $tag
    source_commit = $commit
    training_run = 'yuntong_rtdetr_l_20261021'
    artifact = 'weights/last.pt'
    artifact_sha256 = $packagedHash
    artifact_bytes = (Get-Item -LiteralPath (Join-Path $pkg 'weights/last.pt')).Length
    packaged_at = [DateTimeOffset]::Now.ToString('o')
}
$manifest | ConvertTo-Json -Depth 5 | Set-Content -LiteralPath (Join-Path $pkg 'manifest.json') -Encoding UTF8

# 对包内所有交付文件逐一生成相对路径校验表，校验表自身不纳入。
$pkgFull = (Resolve-Path -LiteralPath $pkg).Path
$lines = @(Get-ChildItem -LiteralPath $pkgFull -File -Recurse | Sort-Object FullName | ForEach-Object {
    $relative = $_.FullName.Substring($pkgFull.Length + 1).Replace('\','/')
    $hash = (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash.ToLowerInvariant()
    "$hash  $relative"
})
$lines | Set-Content -LiteralPath (Join-Path $pkgFull 'SHA256SUMS.txt') -Encoding ASCII
Compress-Archive -LiteralPath $pkgFull -DestinationPath $zip -CompressionLevel Optimal
$zipHash = (Get-FileHash -LiteralPath $zip -Algorithm SHA256).Hash.ToLowerInvariant()
"$zipHash  $tag.zip" | Set-Content -LiteralPath $checksumFile -Encoding ASCII
Get-Item -LiteralPath $zip,$checksumFile | Select-Object Name,Length
```

产物示例：

```text
dist/yuntong-rtdetr-l-v1.0.0.zip
dist/yuntong-rtdetr-l-v1.0.0.zip.sha256
```

ZIP 中有顶层目录 `yuntong-rtdetr-l-v1.0.0/`，包含 `weights/last.pt`、模型卡、配套配置、发布说明、`manifest.json` 和 `SHA256SUMS.txt`。文件 SHA256 保证内容校验，不等同于发布者数字签名。重新压缩可能因时间戳改变 ZIP 的哈希，不覆盖已发布附件。

发布提交标识在打包阶段生成到 `manifest.json`，避免“先把自己的提交 SHA 写回提交”形成循环。训练源码提交与此发布提交分别记录。

## 9. 创建 Release 草稿、验证后正式发布

### 9.1 推荐：网页操作，无须配置额外 CLI 登录

1. 浏览器登录 `karljiang118440`，进入实际仓库。
2. 打开 Releases → Draft a new release。
3. 从现有 Tag 中选择 `yuntong-rtdetr-l-v1.0.0`；不要顺手从错误分支创建新 Tag。
4. 标题填写 `云通 RT-DETR-L v1.0.0（训练批次 20261021）`。
5. 将已提交的 `RELEASE_NOTES.md` 内容填入正文，附上打包生成的 source_commit、权重 SHA256、ZIP SHA256。
6. 上传本地生成的 ZIP 和 `.zip.sha256` 两个附件，保存草稿。
7. 从草稿下载附件，使用第 10 节校验；并按模型卡在干净验证环境完成推理验收。
8. 确认验收通过、许可与数据范围适合仓库可见性后点击 Publish release。若尚未验收，保留草稿；若是试用交付，明确标成 Pre-release。

Release 正文应至少说明：模型与版本、训练批次、代码提交、数据版本、真实指标、环境及硬件、输入输出协议、类别映射、阈值、变更点、限制、部署方法、回滚版本、权重和 ZIP 的 SHA256。首次发布无可回滚版本时写明“首次发布”。

**不要将 GitHub 自动提供的 Source code (zip) 当成模型部署包。** 自动源码压缩包是否包含 LFS 对象取决于仓库设置；交付时明确让使用方下载手工上传的同名模型 ZIP。[Release 与 LFS 归档说明](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)

### 9.2 可选：已经具备 CLI 时发布

不必为了本流程安装 `gh`。如果已有 GitHub CLI 且愿意使用它，认证请使用当前进程临时 `GH_TOKEN`，不要把 token 字符串写进脚本，也不要执行影响全局 Git 的认证配置。本段假设已安全设置该进程凭据，且确认账户身份和目标仓库。

```powershell
# 示例值必须替换成实际存在的 owner/repo
$repo = Read-Host '输入实际 owner/repo'
$tag = 'yuntong-rtdetr-l-v1.0.0'
gh release create $tag "dist/$tag.zip" "dist/$tag.zip.sha256" `
    --repo $repo --verify-tag --draft `
    --title '云通 RT-DETR-L v1.0.0（训练批次 20261021）' `
    --notes-file 'yuntong_weights/yuntong_rtdetr_l_20261021/RELEASE_NOTES.md'
if ($LASTEXITCODE -ne 0) { throw 'Release 草稿创建或附件上传失败' }
# 下载复核与推理验收后，才执行：
# gh release edit $tag --repo $repo --draft=false
```

`--verify-tag` 要求远程 Tag 已存在，`--draft` 先保存草稿。[创建 Release 参数](https://cli.github.com/manual/gh_release_create)、[发布草稿参数](https://cli.github.com/manual/gh_release_edit)

## 10. 下载验收、部署与回滚

### 10.1 校验实际下载的附件

在当前项目下创建一个新的验收目录，例如 `dist/verify-v1.0.0/`；将从草稿或已发布 Release 下载的两个附件放进去，而不是复用原始构建文件。然后执行：

```powershell
$downloadDir = 'dist/verify-v1.0.0'
$tag = 'yuntong-rtdetr-l-v1.0.0'
$zip = Join-Path $downloadDir "$tag.zip"
$sum = Join-Path $downloadDir "$tag.zip.sha256"
$expected = ((Get-Content -LiteralPath $sum -Raw).Trim() -split '\s+')[0]
$actual = (Get-FileHash -LiteralPath $zip -Algorithm SHA256).Hash
if ($expected -notmatch '^[0-9a-fA-F]{64}$' -or $actual -ne $expected) {
    throw '下载附件 SHA256 不一致，停止部署'
}
$extract = Join-Path $downloadDir 'extracted'
if (Test-Path -LiteralPath $extract) { throw '解压目录已存在，使用新的验收目录' }
Expand-Archive -LiteralPath $zip -DestinationPath $extract
$root = (Resolve-Path -LiteralPath (Join-Path $extract $tag)).Path
foreach ($line in Get-Content -LiteralPath (Join-Path $root 'SHA256SUMS.txt')) {
    if ($line -notmatch '^([0-9a-fA-F]{64})  (.+)$') { throw '包内校验表格式错误' }
    $expectedFileHash = $Matches[1]
    $relativeFile = $Matches[2]
    $filePath = [IO.Path]::GetFullPath((Join-Path $root $relativeFile))
    if (-not $filePath.StartsWith($root + [IO.Path]::DirectorySeparatorChar, [StringComparison]::OrdinalIgnoreCase)) {
        throw '校验表路径越界'
    }
    $actualFileHash = (Get-FileHash -LiteralPath $filePath -Algorithm SHA256).Hash
    if ($actualFileHash -ne $expectedFileHash) { throw "包内文件不一致：$relativeFile" }
}
Get-Content -LiteralPath (Join-Path $root 'manifest.json')
```

再核对 manifest 的 Tag、source_commit 和权重 SHA256 与 Release 正文一致，并执行模型卡中的真实推理验收。此流程未自动生成或运行推理代码，因此打包成功不代表模型质量已通过。

### 10.2 确认远程 LFS 对象也能恢复

Release ZIP 成功不代表远程 LFS 对象完整。可在本项目 `dist/` 的新子目录做独立恢复验证，避免切换当前工作目录中的模型：

```powershell
$remoteUrl = (git remote get-url origin).Trim()
$tag = 'yuntong-rtdetr-l-v1.0.0'
$cloneDir = 'dist/verify-lfs-v1.0.0'
if (Test-Path -LiteralPath $cloneDir) { throw '恢复验证目录已存在' }
# 临时跳过 clone 时的 LFS 下载，以免拉取默认分支其他模型。
$priorSkip = $env:GIT_LFS_SKIP_SMUDGE
try {
    $env:GIT_LFS_SKIP_SMUDGE = '1'
    git -c credential.helper= clone --no-checkout $remoteUrl $cloneDir
    if ($LASTEXITCODE -ne 0) { throw '验证克隆失败' }
    git -C $cloneDir lfs install --local
    if ($LASTEXITCODE -ne 0) { throw '验证仓库 LFS 初始化失败' }
    git -C $cloneDir checkout --detach $tag
    if ($LASTEXITCODE -ne 0) { throw '检出版本失败' }
} finally {
    $env:GIT_LFS_SKIP_SMUDGE = $priorSkip
}
git -C $cloneDir -c credential.helper= lfs pull --include 'yuntong_weights/yuntong_rtdetr_l_20261021/weights/last.pt' --exclude ''
if ($LASTEXITCODE -ne 0) { throw '远程 LFS 权重恢复失败' }
git -C $cloneDir lfs fsck
if ($LASTEXITCODE -ne 0) { throw '远程恢复完整性失败' }
Get-FileHash -LiteralPath (Join-Path $cloneDir 'yuntong_weights/yuntong_rtdetr_l_20261021/weights/last.pt') -Algorithm SHA256
```

输出 SHA256 应与已发布版本一致。这些验证目录位于 `dist/`，不会被项目提交。

### 10.3 业务部署与回滚约定

部署端按 Tag 下载指定模型，核验后解压到独立版本目录；先预加载并通过样例检查，再切换服务的模型路径。保留上一个已验收目录；回滚时切回上一个固定版本并重载服务。实际服务重载方式以业务系统为准，不在未知服务上执行通用重启命令。

如果推理配置、类别映射或依赖同时变化，随包一起切换，不能只替换 `.pt`。记录部署时间、环境、模型 Tag、source_commit、权重哈希与操作者。生产配置不要只写 `latest`，也不要依赖不断被训练覆盖的工作目录 `last.pt`。

## 11. 后续迭代规则

| 场景 | 示例版本 | 执行规则 |
| --- | --- | --- |
| 第一次正式交付 | `v1.0.0` | 完成基线验收，建立指标与兼容说明 |
| 同一协议下的小幅修复或再训练 | `v1.0.1` | 新提交、新 Tag、新包，记录指标变化 |
| 向后兼容的能力扩展 | `v1.1.0` | 验证原接口和业务场景继续可用 |
| 类别索引、输入输出协议或加载方式不兼容 | `v2.0.0` | 提供迁移说明，部署端同步适配 |

版本号按业务兼容性确定，不能只根据“又训练了一次”机械升级。下一轮建议新建 `yuntong_rtdetr_l_YYYYMMDD/`，定向放行并跟踪选定权重；更新模型卡、评估结果和打包脚本中的模型路径、Tag、版本。

同样的二进制内容可由 LFS 复用；不同权重不要以“省空间”为由覆盖已有 Release。旧版本保留规则结合合规与回滚需求制定。删除分支、Tag 或本地文件不代表远程 LFS 历史对象立即释放计费空间。

## 12. 常见问题定位

| 现象 | 原因与处理 |
| --- | --- |
| `git` 无法识别 | 当前会话找不到程序；确认工具位置，仅在本会话调整 PATH |
| `git add` 提示 ignored | 定向白名单缺失或被后置规则覆盖，先检查 `.gitignore` |
| `.gitattributes` 没提交 | 原 `*` 把它忽略；补精确放行后再添加 |
| 推送出现超过 100 MiB | 权重曾以普通 Git 对象进入提交；检查暂存指针与历史，不仅补 track |
| LFS 配额/权限报错 | 检查真实账户配额、仓库权限和 PAT 范围；失败时不发布 |
| 下载到很小的 `.pt` | 可能是 LFS 指针；安装 LFS 后拉取对应版本对象并核验哈希 |
| 找不到 `best.pt` | 本机确实没有；使用经评估候选，不直接改名假装最佳 |
| `epoch90.pt.filepart` 无法加载 | 零字节未完成文件，不进入正式发布 |
| Release 只有源码 ZIP | 还未上传部署包；上传实际模型 ZIP 和 SHA256 附件 |
| 推送被拒绝 non-fast-forward | 远程不是空仓库或已更新；获取并核对历史，不强推 |
| 新模型输出类别错位 | 核查 classes 顺序、预处理及后处理配置是否对应权重 |

## 13. 发布完成的判定与本次实际交付

真正完成发布，必须同时具备：可恢复的远程提交、正确 Tag、可下载的 LFS 权重、正式 Release、完整 ZIP、SHA256、真实评估和推理验收记录，以及明确部署/回滚方式。

本次交付仅为当前项目中的本手册及本机文件检查结果。尚需提供或确认实际远程仓库地址、确认的提交邮箱、模型类别/配置/依赖和真实验收结果；没有把未知信息写成事实，没有使用聊天中披露的密码，没有变更其他项目或全局 Git 设置。

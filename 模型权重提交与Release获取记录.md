# 模型权重提交与 Release 获取记录

日期：2026-10-23；项目根：D:\git\TH_code-main\TH_code。

本记录覆盖此前手册中的历史检查结论：本次重新检查已发现本机 Git 2.56.0、Git LFS 3.8.0，目录新增 best.pt、args.yaml、results.csv 等文件。

## 本次管理范围

仅在当前项目初始化仓库，分支 main；仓库本地设置 user.name=karljiang118440、user.email=karljiang118440@users.noreply.github.com，LFS 使用 --local 初始化。不更改全局配置。该 noreply 邮箱仅为本地提交署名，尚未确认其为账户设置中的邮箱，不保证 GitHub 账户归属关联；正式协作前可替换为账户中核实的邮箱。

候选 Tag：yuntong-rtdetr-l-v1.0.0-rc.1。LFS 收录 best.pt 和 last.pt，普通 Git 收录训练参数、CSV、评估曲线、模型卡、元数据、发布说明和本操作记录。没有提交其他业务代码、epoch 检查点或样本图。

模型没有独立推理验收，使用 rc.1 候选版本，不宣称生产可用。未知的类别、框架版本、数据集版本不伪造。

## 获取本地 Release 包

包路径：dist/yuntong-rtdetr-l-v1.0.0-rc.1.zip。
校验文件：dist/yuntong-rtdetr-l-v1.0.0-rc.1.zip.sha256。
实际提交、Tag、包大小、ZIP SHA256 及验证结果由打包完成后生成的 dist/release-build-record.json 记录；包内 manifest.json 记录发布提交和文件哈希。

PowerShell 校验方式：

```powershell
Set-Location -LiteralPath 'D:\git\TH_code-main\TH_code'
$zip = 'dist/yuntong-rtdetr-l-v1.0.0-rc.1.zip'
$expected = ((Get-Content -LiteralPath ($zip + '.sha256') -Raw).Trim() -split '\s+')[0]
if ((Get-FileHash -LiteralPath $zip -Algorithm SHA256).Hash -ne $expected) { throw 'SHA256 不一致' }
# 解压到新目录，之后按 MODEL_CARD.md 核实环境和推理方式。
Expand-Archive -LiteralPath $zip -DestinationPath 'dist/download-check'
```

## 在线获取方法与待办

本地完成不等于远程已发布。用户已指定远程仓库 https://github.com/karljiang118440/yuntong_weights.git；联网检查当前远程没有 refs。推送和 Release 发布结果以 dist/release-build-record.json 及后续执行记录为准。远程已有历史时先整合，不能直接强推。

若远程是 GitHub：进入实际仓库 → Releases → 对应 yuntong-rtdetr-l-v1.0.0-rc.1 → Assets，下载同名 .zip 与 .zip.sha256。创建 Release 时选择已推送 Tag，粘贴 RELEASE_NOTES.md，上传上述两个附件，勾选 Pre-release。下载链接只有在实际上传后才能提供，不能用推测的仓库名拼接成已发布地址。

使用 Git 克隆的用户需安装 LFS，检出相同 Tag 后拉取 LFS 对象，再核对元数据中的权重哈希。Release ZIP 使用方无须 Git LFS。

已获用户授权提交至上述仓库。使用既有认证，不使用聊天中披露的密码，不修改全局认证配置。如缺少认证则保留本地提交与包，明确报告尚未上传。

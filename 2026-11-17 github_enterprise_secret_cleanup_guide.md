# GitHub Enterprise Secret 泄漏：Git 历史清理操作手册

> 适用场景：GitHub Enterprise 仓库中曾经提交过 Secret（密码、Token、API
> Key、私钥等）；即使后来删除了代码或分支，扫描仍能从历史 Commit
> 中发现。
>
> 适用环境：Windows 11 + Git for Windows + PowerShell；清理工具为
> `git-filter-repo`。
>
> **重要：** 本文中的命令需要根据实际仓库地址、Owner、Repository、Commit
> SHA 和文件路径替换。先在测试仓库演练，再处理生产仓库。

## 目录

-   [1. 问题原因与处理原则](#1-问题原因与处理原则)
-   [2. 整体处理流程](#2-整体处理流程)
-   [3. 事前准备与风险控制](#3-事前准备与风险控制)
-   [4. Windows 11 环境准备](#4-windows-11-环境准备)
-   [5. 定位泄漏内容](#5-定位泄漏内容)
-   [6. 镜像备份并获取仓库全部引用](#6-镜像备份并获取仓库全部引用)
-   [7. 选择历史清理方案](#7-选择历史清理方案)
    -   [7.1 方案 A：替换历史中的 Secret](#71-方案-a替换历史中的-secret)
    -   [7.2 方案
        B：从全部历史中删除敏感文件](#72-方案-b从全部历史中删除敏感文件)
    -   [7.3 关于"删除整个 Commit"](#73-关于删除整个-commit)
-   [8. 清理后本地验证](#8-清理后本地验证)
-   [9. 推送重写后的历史](#9-推送重写后的历史)
-   [10. GitHub Enterprise
    服务器端处理](#10-github-enterprise-服务器端处理)
    -   [10.1 GitHub Enterprise
        Cloud（GHEC）](#101-github-enterprise-cloudghec)
    -   [10.2 GitHub Enterprise
        Server（GHES）](#102-github-enterprise-serverghes)
-   [11. Developer 本地仓库处理](#11-developer-本地仓库处理)
-   [12. 后续防护建议](#12-后续防护建议)
-   [13. 验收清单](#13-验收清单)
-   [14. 官方参考资料](#14-官方参考资料)

------------------------------------------------------------------------

## 1. 问题原因与处理原则

Git
的提交历史是内容对象和引用组成的有向无环图。删除文件、删除代码行或删除
Branch，通常只会改变当前引用指向的历史，不会自动改写旧 Commit。

因此：

-   删除包含 Secret 的代码，不等于删除历史中的 Secret。
-   删除曾经提交代码的 Branch，不代表该 Commit 已经不可达；其他
    Branch、Tag、Pull Request、Fork 或缓存仍可能引用它。
-   Force Push 只更新可推送的 Git 引用，不保证 GitHub
    服务器上的所有缓存、PR 引用或不可达对象立即清除。

**处理原则：**

1.  先撤销或轮换泄漏的凭证，消除凭证继续被使用的风险。
2.  再重写 Git 历史，清除所有相关引用可达的敏感内容。
3.  推送清理后的引用，并检查 PR、Fork、缓存和服务器端对象。
4.  通知所有协作者重新获取清理后的仓库，避免旧历史重新进入仓库。

> 历史重写会改变 Commit
> SHA，并可能影响分支、标签、PR、签名提交和自动化流程。务必先协调仓库管理员及团队。

## 2. 整体处理流程

1.  确认最早引入 Secret 的 Commit、敏感文件和凭证类型。
2.  在凭证所属系统中撤销、禁用或轮换旧凭证。
3.  暂停仓库 Push、Merge、Release 等写入操作。
4.  备份仓库，并使用镜像克隆获取全部可见 Git 引用。
5.  使用 `git-filter-repo` 替换敏感字符串，或从全部历史中删除敏感文件。
6.  检查清理结果、所有 Branch、Tag、PR 和 Fork。
7.  在确认影响范围后，强制更新远程引用。
8.  根据 GHEC / GHES 部署类型处理服务器端残留。
9.  通知 Developer 重新 Clone 或基于新历史重建工作分支。
10. 增加 Secret 扫描和 Push Protection 等防护。

## 3. 事前准备与风险控制

### 3.1 先撤销泄漏凭证

不要把"历史已经清理"当成凭证安全的前提。凭证一旦提交到远程仓库，应按已泄漏处理。

  Secret 类型                    处理建议
  ------------------------------ --------------------------------
  数据库账号密码                 修改密码，检查访问日志
  AWS / 云平台 Access Key        禁用旧 Key，创建新 Key
  GitHub Personal Access Token   Revoke 旧 Token，重新生成
  API Key                        撤销旧 Key，重新生成
  SSH 私钥                       更换密钥对，并移除旧公钥授权
  JWT 签名密钥                   轮换签名密钥，并评估存量 Token

同时检查凭证是否被用于其他环境、仓库、CI/CD 变量或部署配置。

### 3.2 冻结仓库写入

在清理期间通知团队暂停：

-   Push / Force Push
-   Merge Pull Request
-   创建或更新 Release / Tag
-   自动化机器人对仓库的写入

如果使用分支保护规则或 Ruleset，提前确认后续是否需要管理员临时调整。

### 3.3 备份的安全要求

原始备份包含泄漏的 Secret，必须：

-   存放在受控、加密的存储位置。
-   限制访问人员。
-   不要上传到新的普通 Git 仓库、工单或聊天群。
-   明确保留期限，并在事件结束后按公司制度销毁或归档。

## 4. Windows 11 环境准备

安装 Git for Windows 和 Python。打开 PowerShell 检查：

``` powershell
git --version
python --version
python -m pip --version
```

安装或升级 `git-filter-repo`：

``` powershell
python -m pip install --upgrade git-filter-repo
```

检查工具：

``` powershell
git filter-repo --version
```

如果提示找不到 `git filter-repo`，检查 Python Scripts 目录是否已加入
PATH，或使用 Python 环境对应的安装方式修复 PATH。

建议使用较新的 `git-filter-repo`
版本。执行前先在非生产仓库验证命令和结果。

## 5. 定位泄漏内容

在 GitHub Enterprise 仓库中找到 Secret Scanning 告警或扫描报告，记录：

  信息                示例
  ------------------- -------------------------------------------
  仓库                `company/backend-service`
  原始 Commit SHA     `abc123...`
  敏感文件            `src/main/resources/application-prod.yml`
  泄漏类型            数据库密码 / API Key
  曾经存在的 Branch   `feature/test`
  相关 Pull Request   PR 编号（如有）

需要尽可能定位**最早引入 Secret 的 Commit**，而不只是后续删除代码的
Commit。

不要将真实 Secret 粘贴到工单、聊天或文档中。后续命令中涉及 Secret
的示例均为占位内容。

## 6. 镜像备份并获取仓库全部引用

### 6.1 创建工作目录

``` powershell
New-Item -ItemType Directory -Force D:\GitCleanup
Set-Location D:\GitCleanup
```

### 6.2 镜像克隆

建议从仓库 URL
创建一个新的镜像克隆。镜像克隆用于获取远程可见的分支、标签及其他可见引用：

``` powershell
git clone --mirror https://github.enterprise.com/company/backend-service.git backend-service.git
```

进入目录：

``` powershell
Set-Location D:\GitCleanup\backend-service.git
git show-ref
git remote -v
```

确认远程地址、分支和标签符合预期。某些 GitHub 内部引用（例如
`refs/pull/*`）不能像普通分支一样通过标准 Push
管理；不要假设镜像克隆就能获取或清除所有服务器内部引用。

### 6.3 备份

在 `D:\GitCleanup` 目录下执行：

``` powershell
Set-Location D:\GitCleanup
git -C .\backend-service.git bundle create .\backend-service-before-cleanup.bundle --all
git bundle verify .\backend-service-before-cleanup.bundle
```

将 Bundle 放入受控加密存储。它包含旧历史和泄漏内容。

> 镜像克隆与 Bundle 备份并不替代 GitHub Enterprise
> 的服务器端备份，也不包括 GitHub 托管的所有 PR、缓存或 Fork 数据。

## 7. 选择历史清理方案

根据泄漏内容选择一种主要方案。**不要不加判断地同时执行两种方案。**

### 7.1 方案 A：替换历史中的 Secret

适用情况：

-   敏感信息是代码中的一个或多个字符串。
-   Commit 还包含需要保留的正常业务代码。
-   希望保留提交结构，只替换泄漏值。

在 `D:\GitCleanup` 创建 `replacements.txt`。示例：

``` text
旧数据库密码示例==>***REMOVED***
旧APIKey示例==>***REMOVED***
```

格式为：

``` text
原始字符串==>替换后的字符串
```

注意：

-   左侧必须是历史中真实出现的字符串。
-   右侧使用统一的占位文本。
-   一条规则占一行。
-   如果 Secret 包含换行或特殊字符，需要参考 `git-filter-repo`
    的替换文件格式处理。
-   确保该文件不在仓库目录内，不要提交或公开它。
-   使用适当的文件权限保护文件内容。

在镜像仓库目录中执行：

``` powershell
Set-Location D:\GitCleanup\backend-service.git

git filter-repo --sensitive-data-removal --replace-text D:\GitCleanup\replacements.txt
```

该操作会在被处理的历史内容中替换匹配字符串。清理后应使用 Secret
扫描工具检查，不要只依赖单个字符串搜索。

### 7.2 方案 B：从全部历史中删除敏感文件

适用情况：

-   整个文件都不应出现在 Git 历史中。
-   文件包含多个敏感值，且没有必要保留该文件的历史版本。

例如要从全部历史删除：

``` text
src/main/resources/application-prod.yml
```

执行：

``` powershell
Set-Location D:\GitCleanup\backend-service.git

git filter-repo --sensitive-data-removal --invert-paths --path src/main/resources/application-prod.yml
```

如果文件曾经改名或移动过，需要把所有历史路径都加入参数：

``` powershell
git filter-repo --sensitive-data-removal --invert-paths `
  --path src/main/resources/application-prod.yml `
  --path config/application-prod.yml
```

路径相对于仓库根目录。PowerShell 使用反引号 `` ` ``
作为换行续行符；反引号必须放在行尾，后面不要有空格。

该操作会从重写后的历史中移除指定路径的文件及其历史版本。执行前确认文件是否包含需要保留的非敏感配置。

### 7.3 关于"删除整个 Commit"

Git 历史重写不是简单地把某一个 Commit 文件删除。一个 Commit
可能包含多项业务修改，后续 Commit 也可能依赖它。

如果只想清除其中的 Secret，优先采用方案
A；如果整个文件都不应保留，采用方案 B。

如果确实要移除整个 Commit
的全部业务变更，需要先分析它与后续提交的父子关系、合并提交及依赖关系，再设计交互式重写或回放方案。不要仅凭
Commit SHA 直接删除对象，也不要把普通 `git revert`
误认为历史清除：`git revert` 只新增反向提交，旧 Secret 仍留在历史中。

## 8. 清理后本地验证

### 8.1 检查重写后的引用

``` powershell
git show-ref
git log --all --oneline -20
```

确认预期分支和标签仍然存在，且没有意外丢失。

### 8.2 检查原始 Commit

``` powershell
git cat-file -t 原始CommitSHA
```

如果返回 `commit`，表示对象仍在本地对象库中；如果报
`Not a valid object name`，表示该对象在当前对象库中不可访问。

**注意：** 对象仍存在不一定代表清理失败，因为 Git
对象可能成为不可达对象；对象不可访问也不能单独证明所有远程引用、PR、Fork
和缓存均已清理。

### 8.3 检查替换字符串

对每个旧 Secret，可以执行：

``` powershell
git log --all -S"待检查的字符串" --oneline
```

没有输出表示该搜索方式没有找到相关差异，但这不是完整安全扫描，也不能覆盖所有二进制文件、编码形式或其他泄漏值。

更可靠的做法是：

-   对清理后的仓库运行 Gitleaks 或企业安全扫描工具。
-   对照原始 Secret 清单逐项检查。
-   检查文件内容、提交差异和相关配置。
-   由安全团队确认扫描结果。

不要将真实 Secret 写入共享终端日志或 CI 输出。

### 8.4 查看 git-filter-repo 记录

检查：

``` powershell
Get-ChildItem .git\filter-repo
```

镜像仓库通常是裸仓库，Git
元数据目录本身就是仓库目录，因此实际路径可能是：

``` text
D:\GitCleanup\backend-service.git\filter-repo
```

查看变更引用记录（如果该版本生成了该文件）：

``` powershell
Get-Content .\filter-repo\changed-refs
```

记录工具输出的 `First Changed Commit(s)` 等信息，后续联系管理员或 GitHub
Support 时可能需要。

## 9. 推送重写后的历史

### 9.1 检查远程地址

`git-filter-repo` 出于安全考虑可能移除 `origin`。检查：

``` powershell
git remote -v
```

如果没有远程地址，重新添加：

``` powershell
git remote add origin https://github.enterprise.com/company/backend-service.git
```

如果 `origin` 仍存在，确认它指向正确的目标仓库，不要重复添加。

### 9.2 推送前再次确认

确认：

-   仓库写入仍处于冻结状态。
-   已备份原始仓库。
-   清理后的 Branch、Tag 和引用经过检查。
-   已评估 PR、Fork、签名提交、Release 和 CI/CD 的影响。
-   已取得管理员批准。
-   当前镜像中没有不应推送或删除的引用。

### 9.3 强制镜像推送

``` powershell
git push --force --mirror origin
```

**高风险说明：**

-   `--mirror`
    会同步所有本地可推送引用，并可能删除远端存在但本地不存在的引用。
-   它不只是更新某一个 Branch。
-   远程保护规则可能拒绝 Force Push。
-   GitHub 管理的 `refs/pull/*` 等只读引用不能通过普通 Push 删除。
-   不要在未检查引用差异的情况下直接对生产仓库执行。

如果推送失败，保存完整错误信息，逐项处理分支保护、权限、只读引用和其他被拒绝的引用；不要为了让命令显示成功而盲目扩大权限。

## 10. GitHub Enterprise 服务器端处理

Force Push 更新引用后，仍需评估服务器端不可达对象、PR 引用、缓存和
Fork。

首先确认部署类型：

-   **GHEC**：GitHub Enterprise Cloud，由 GitHub 托管。
-   **GHES**：GitHub Enterprise Server，由企业自行部署和管理。

### 10.1 GitHub Enterprise Cloud（GHEC）

处理建议：

1.  完成凭证撤销或轮换。
2.  完成历史重写和 Force Push。
3.  检查仍然存在的 Branch、Tag、PR 和 Fork 引用。
4.  记录原始泄漏 Commit SHA、`git-filter-repo` 输出的
    `First Changed Commit(s)`、受影响 PR 编号。
5.  通过 GitHub Support
    提交敏感数据清理请求，说明已经完成的修复步骤，并询问是否能清理相关
    PR 引用、缓存或服务器端对象。

支持入口：

https://support.github.com/

可参考以下英文说明模板：

``` text
Repository:
company/backend-service

Original leaked commit:
<original commit SHA>

First changed commit(s):
<git-filter-repo output>

Affected pull requests:
<PR numbers, if any>

Remediation completed:
1. Revoked or rotated the exposed credentials.
2. Rewrote Git history using git-filter-repo.
3. Force-pushed the cleaned repository.
4. Reviewed branches, tags, and known references.

Request:
Please advise whether any server-side cached views, pull request references,
or unreachable Git objects related to the exposed data require further cleanup.
```

GitHub Support 是否能够执行服务器端清理，需要由 GitHub
根据具体情况确认；不要承诺或假设所有对象都一定可以被立即清除。

### 10.2 GitHub Enterprise Server（GHES）

GHES 的服务器端操作应由有权限的 Site Administrator 按照**当前 GHES
版本**的官方文档执行。

建议管理员：

1.  检查仓库及 Fork 是否仍有引用指向原始 Commit。
2.  评估 PR 内部引用和缓存。
3.  在确认对象不可达且符合维护要求后，执行官方支持的 Git 垃圾回收流程。
4.  按当前版本文档处理 Git 缓存失效。
5.  再次检查仓库和相关引用。

部分 GHES 版本文档提供了 `ghe-repo`、`ghe-repo-gc`
等管理工具。例如文档可能涉及：

``` bash
ghe-repo OWNER/REPOSITORY -c 'git ref-contains COMMIT_SHA'
```

以及：

``` bash
ghe-repo-gc -v --prune OWNER/REPOSITORY
```

**重要：** 这些是 GHES Appliance 管理操作，不是 Windows Developer
的命令。具体命令是否适用、参数和执行时机，必须以你们实际 GHES
版本的官方文档为准；不要未经确认直接在生产 Appliance 上执行。执行 GC
前应确认备份、引用状态和维护窗口。

## 11. Developer 本地仓库处理

历史重写后，所有 Developer 的本地仓库可能仍然保留旧 Commit。

建议：

1.  暂停旧仓库 Push。
2.  备份尚未提交的业务修改和必要分支内容。
3.  删除或隔离旧工作目录。
4.  从远程重新 Clone 清理后的仓库。
5.  对未合并的业务修改，基于新历史重新创建分支并谨慎移植提交。
6.  不要把旧仓库的完整历史分支直接 Merge 或 Push 回新仓库。

重新 Clone 示例：

``` powershell
Set-Location D:\Projects
git clone https://github.enterprise.com/company/backend-service.git
```

如果 Developer 需要保留尚未提交的修改，应先使用
Patch、独立文件备份等方式保存工作内容；不要将包含旧泄漏历史的旧分支直接推回远程。

## 12. 后续防护建议

建议至少建立三层防护：

  -----------------------------------------------------------------------------------------------------
  防护层                  工具 / 机制                                        作用
  ----------------------- -------------------------------------------------- --------------------------
  本地提交前              Gitleaks pre-commit hook                           尽早阻止敏感内容进入
                                                                             Commit

  远程 Push 阶段          GitHub Secret Scanning / Push                      阻止已识别的 Secret 被推送
                          Protection（按许可证、部署版本及配置确认可用性）   

  CI/CD 阶段              Gitleaks 或企业安全扫描工具                        对提交和代码变更持续扫描
  -----------------------------------------------------------------------------------------------------

Gitleaks：

https://github.com/gitleaks/gitleaks

GitHub Push Protection：

https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection

额外建议：

-   不要把生产密码写入 `application-prod.yml` 并提交到仓库。
-   使用企业 Secret Manager、CI/CD Secret Variables
    或环境变量注入敏感配置。
-   对开发、测试、生产环境使用不同凭证。
-   定期扫描历史仓库，而不仅仅扫描最新代码。
-   为 Secret 轮换和安全事件处置建立明确流程。

## 13. 验收清单

-   [ ] 已撤销或轮换泄漏的 Secret
-   [ ] 已记录最早泄漏 Commit SHA
-   [ ] 已备份原始仓库并安全保存
-   [ ] 已获取并检查仓库可见的 Branch、Tag 和引用
-   [ ] 已使用 `git-filter-repo` 重写历史
-   [ ] 已检查替换结果或敏感文件删除结果
-   [ ] 已使用 Secret 扫描工具复查
-   [ ] 已检查受影响的 Pull Request
-   [ ] 已检查 Fork 和其他历史引用
-   [ ] 已 Force Push 清理后的历史
-   [ ] 已由管理员处理适用的服务器端对象和缓存问题
-   [ ] 所有 Developer 已重新 Clone 或基于新历史重建分支
-   [ ] 已增加提交前、Push 阶段和 CI/CD 扫描防护

**关闭事件的判断标准：** 不能只看当前 Branch 已经搜索不到
Secret。还要确认旧凭证已失效、远程引用已清理、PR/Fork/缓存风险已评估、团队旧历史不会重新推送，并由仓库或安全管理员完成验收。

## 14. 官方参考资料

-   GitHub：Removing sensitive data from a repository\
    https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository

-   GitHub：About secret scanning\
    https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning

-   GitHub：About push protection\
    https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection

-   git-filter-repo 项目\
    https://github.com/newren/git-filter-repo

-   Gitleaks 项目\
    https://github.com/gitleaks/gitleaks

------------------------------------------------------------------------

**最后提醒：** 历史清理不能替代凭证轮换。只要 Secret
曾经进入远程仓库，就应先按泄漏事件处理，再进行 Git 历史清理。

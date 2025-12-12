# 妙算本地开发 CLI

使用本地 IDE（VS Code / PyCharm 等）开发妙算仿真模型、与云端工作区（Workspace）进行代码同步，并支持本地一键运行云端仿真任务的命令行工具。

> 通过 `miaosuan`，你可以把云端 Workspace 映射到本地目录，像写普通 Python 项目一样开发仿真模型，再使用 `push/pull/run` 命令与平台协同。

> 提示：
> 🚫禁止同时在web端和本地同时进行进行模型的编辑，否则可能会造成**修改丢失**，务必在一段修改完成，保存/推送后，关闭编辑器再在另一端进行编辑和修改。

---

## 功能概览

- 本地初始化云端 Workspace（`miaosuan init`）
- 本地增量上传到云端（`miaosuan push`）
- 从云端拉取最新代码（`miaosuan pull`）
- 一键运行妙算云端仿真任务并回显日志（`miaosuan run`）
- 自动维护模型文件中的 `pythonPackageFiles` 列表
- 自动创建 Python 虚拟环境（使用 `uv venv`）
- 安装/升级 `miaosuan-stub` 提示包（`miaosuan update-stub`）
- CLI 自身版本更新检查（`miaosuan check-update`）

---

## 安装

### 运行环境要求

本工具的虚拟环境、安装stub包等功能依赖于[uv](https://github.com/astral-sh/uv)。
- windows安装包中已经自带该工具，安装完成后可以直接在命令行运行。
- Linux/MacOS：请参考 [uv官方文档](https://docs.astral.sh/uv/getting-started/installation/) 自行安装 uv，并确保其在 `PATH` 中可用。

### 下载地址

当前最新版本：v0.2.0

[蓝奏云下载](https://www.ilanzou.com/s/3hon0K75)

版本历史：
- v0.2.0 [下载](https://www.ilanzou.com/s/3hon0K75) 支持提交任务时使用场景中保存的参数信息，并支持通过命令行传入新的参数覆盖场景参数 
- v0.1.3 [下载](https://www.ilanzou.com/s/daNnyoLF) 实现基本功能 

### 安装
1. 命令行工具安装：
 - Windows: 下载 installer安装包，直接双击运行安装即可。
 - Linux/MacOS: 下载对应平台的可执行文件，放置到系统 `PATH` 中的某个目录下，并赋予可执行权限，例如 `/usr/local/bin`。

2. VS Code 扩展安装：

妙算本地开发CLI除了一个命令行工具外，还额外提供了一个 VS Code 扩展，该扩展用于在 VS Code 中，实现可视化的进程模型编辑功能。用于实现在本地和Web端一致的进程模型编辑体验。

安装方法：
- 从对应链接下载 `.vsix` 后缀的扩展包
- 打开 VS Code，点击 `扩展/Extensions`，选择右上角的 `...` 菜单，选择 `从VSIX安装/Install from VSIX...`，选择下载好的扩展包进行安装即可。

---

## 首次使用与配置

CLI 的全局配置保存在当前用户目录下：

- `~/.miaosuan/config.json`

首次执行任意需要访问服务器的命令（例如 `miaosuan init`、`miaosuan push`、`miaosuan pull`、`miaosuan run`），如果发现配置不存在或不完整，会进入交互式配置流程：

```text
配置本地妙算 CLI 所需的服务器地址和 API Key。
蓝图服务器地址 (例如 https://app.znetlink.com):
API Key (通过网页获取，格式类似 <id>.<secret>):
```

示例配置文件：

```json
{
  "version": "1.0",
  "server_url": "https://app.znetlink.com",
  "api_key": "<your-api-key>",
  "last_version_check_time": 0,
  "last_known_latest_version": ""
}
```

> 提示：API Key 需要在网页端的“API Key 管理”中申请。

### 自动更新检查

默认情况下，CLI 会在必要时（约每 24 小时最多一次）对比服务器上的最新 CLI 版本，并在发现新版本时给出提示。

如果你不希望自动检查更新，可设置环境变量关闭：

```bash
export MIAOSUAN_NO_UPDATE_CHECK=1
```

---

## 工作区结构

执行 `miaosuan init` 后，会在本地得到一个以`工作区名称`命名的 Workspace 根目录，内部典型结构如下：

```text
<workspace-root>/
  pycodes/            # Python 代码目录（一个或多个 Python 包，即实际的进程模型和管道模型的代码库）
    <modelA>/
      pyproject.toml
      ...
    msps_<stageName>/
      pyproject.toml
      ...
  procs/              # 进程模型 *.pr.m
    demo.pr.m
  pipelines/          # 管道阶段模型 *.ps.m
    demo.ps.m
  .miaosuan/          # 工作区元数据
    file_state.json   # 上次成功同步的文件快照
    workspace_id.json # 当前目录对应的 Workspace ID
    ignore.txt        # 忽略规则，语法类似 .gitignore
  .venv/              # 由 uv venv 创建的虚拟环境
```

`.miaosuan/` 中的文件说明：

- `workspace_id.json`：记录当前目录绑定的 Workspace ID。
- `file_state.json`：记录上一次同步时所有追踪文件的 SHA-256，用于计算增量。
- `ignore.txt`：自定义忽略规则，语法类似 `.gitignore`（支持通配符、目录结尾 `/` 等）。

CLI 内置会忽略以下路径/文件：

- `.git/`
- `.miaosuan/`
- `.venv/`
- `__pycache__/`
- `*.pyc`

你可以在 `ignore.txt` 中追加自己的规则，例如：

```text
# .miaosuan/ignore.txt
dist/
*.log
```

---

## 命令总览

执行 `miaosuan` 或 `miaosuan --help` 可以查看命令列表：

```text
妙算本地开发 CLI

用法:
  miaosuan init <workspace_id> [target_dir]   在当前目录下创建子目录并初始化为指定 Workspace 的本地副本
  miaosuan push                                将本地模型代码文件修改增量上传到服务器
  miaosuan pull                                从服务器拉取模型代码的最新文件
  miaosuan run <scenario_name>                 运行妙算仿真任务并回显日志
  miaosuan update-stub                         更新当前 workspace 虚拟环境中的 miaosuan-stub 包
  miaosuan check-update                        手动检查 CLI 是否有新版本
  miaosuan version                             显示 CLI 版本信息
```

下面分别介绍。

---

## 命令详解

### 1. `miaosuan init <workspace_id> [target_dir]`

在本地初始化一个云端 Workspace 的副本。典型用法：

```bash
# 在当前目录下创建一个以 Workspace 名称命名的新目录
miaosuan init <workspace-id>

# 或者显式指定目标目录
miaosuan init <workspace-id> my-workspace
```

执行流程（简要）：

1. 读取或交互配置服务器地址与 API Key；
2. 根据 `workspace_id` 访问 `/workspaces` 接口，获取 Workspace 名称；
3. 在当前目录下创建目标目录：
   - 若提供了 `target_dir`，则使用该目录（相对或绝对路径）；
   - 否则以 Workspace 名称为目录名；
   - 如果目标路径已存在（无论是否为空），会报错并终止。
4. 在新目录中拉取该 Workspace 的全部模型文件：
   - `/models/pycodes/**` → 本地 `pycodes/**`
   - `/models/procs/*.pr.m` → 本地 `procs/*.pr.m`
   - `/models/pipelines/*.ps.m` → 本地 `pipelines/*.ps.m`
5. 创建 `.miaosuan/ignore.txt`（如果不存在），写入常用默认规则；
6. 扫描当前文件生成初始 `file_state.json`；
7. 调用 `uv venv` 在工作区根目录创建 `.venv`；
8. 自动以可编辑模式安装 `pycodes/` 下带 `pyproject.toml` 的一级子目录；
9. 在虚拟环境中安装 `miaosuan-stub` 包。

成功后，你就获得了一个完整可开发的本地 Workspace。

> 提示：`pycodes/<包名>/pyproject.toml` 存在时，CLI 会自动执行 `uv pip install --no-deps -e .`。

---

### 2. `miaosuan push`

将本地修改增量上传到云端。

```bash
cd <workspace-root>
miaosuan push
```

执行流程：

1. 读取 `.miaosuan/ignore.txt` 和内置忽略规则；
2. **自动维护 `pythonPackageFiles` 字段**：
   - 遍历 `procs/*.pr.m`：
     - 对于 `procs/<name>.pr.m`，查找本地 `pycodes/<name>/`，递归收集其中所有文件路径；
     - 忽略 `.git/`、`.venv/`、`.miaosuan/`、`__pycache__/`、`*.pyc` 等；
     - 将收集到的列表写入对应模型 JSON 中的 `pythonPackageFiles` 字段；
   - 遍历 `pipelines/*.ps.m`：
     - 对于 `pipelines/<name>.ps.m`，查找本地 `pycodes/msps_<name>/`，逻辑同上；
3. 对整个工作区做一次文件快照（计算 SHA-256），与 `.miaosuan/file_state.json` 中的上次快照比对；
4. 得到两部分结果：
   - `changed`：新增或修改过的文件；
   - `deleted`：上次存在但现在不存在的文件；
5. 逐个处理差异：
   - 对 `changed` 文件：
     - 计算远端路径（见“文件映射规则”一节），调用 `/storage/upload` 覆盖上传；
   - 对 `deleted` 文件：
     - 调用 `/storage/delete` 删除远端文件；
6. 全部操作成功后，更新本地 `file_state.json`。

> 注意：`push` 使用的是“最后写入覆盖”策略（Last Write Wins），不会执行冲突合并。建议团队协作时约定好操作顺序。

---

### 3. `miaosuan pull`

从云端拉取最新模型文件到本地，同时保留本地尚未同步的修改（通过备份）。

```bash
cd <workspace-root>
miaosuan pull
```

执行流程：

1. 先对当前本地文件做一次快照（作为“拉取前快照”）；
2. 分别从服务器列出并下载以下前缀下的所有文件：
   - `/models/pycodes/**`
   - `/models/procs/**`
   - `/models/pipelines/**`
3. 对于每个远端文件：
   - 使用本地“拉取前快照”中对应路径，计算 `remoteHash` 与 `localHash`；
   - 情况：
     - **本地不存在**：直接创建该文件；
     - **本地存在且 hash 相同**：不做任何修改；
     - **本地存在且 hash 不同（冲突）**：
       - 将本地文件重命名为：
         - `<file>.local.bak`，如已存在则为 `<file>.local.bak.1`、`.2` 等；
       - 将远端内容写回原路径；
       - 在输出中提示发生冲突，并告知备份文件名；
4. 拉取结束后重新扫描，并覆盖写入新的 `file_state.json`。

> 通过这种方式，可以在拉取远端修改时保护本地尚未 push 的代码，冲突文件会以 `.local.bak` 形式保存在同目录下，方便你手工比对与合并。

---

### 4. `miaosuan run <scenario_name>`

触发云端妙算仿真任务，并在终端回显日志。

```bash
cd <workspace-root>
miaosuan run demo_scenario
# 或通过命令行追加 / 覆盖仿真参数（可重复多次传入 --param，支持“多组参数”）：
miaosuan run demo_scenario --param duration=120 --param "stats:filters=[\"udp.*\",\"tcp.*\"]"
```

执行流程：

1. 对当前本地文件做快照，与 `file_state.json` 对比；
2. 如果发现本地有新增/修改/删除：
   - 在终端提示“检测到本地有修改”，并询问：
     - 是否先执行 `push` 同步到服务器；
   - 若选择 `y/yes`，则先执行一次 `miaosuan push`；
   - 若选择 `N` 或直接回车，则继续使用服务器上一次同步的代码运行仿真；
3. 尝试从 `/models/scenarios/<scenario_name>.scen.m` 读取场景配置，并解析其中与妙算引擎相关的仿真参数：
   - `simParams.MIAOSUAN`：作为妙算引擎的仿真参数对象，其内部的键值对会“原样”合并到任务参数中（不做额外转换或默认值填充）；
   - `statsFilters`：场景统计量过滤配置（字符串数组），在提交时映射为任务参数中的 `stats:filters`（如存在）。
4. 调用 `/task/submit` 提交仿真任务，参数包括：
   - `scenarioName`
   - `workspace`（当前 Workspace ID）
   - 从场景文件 `simParams.MIAOSUAN` 中读取并合并的仿真参数（如果存在）
   - 以及 `stats:filters`（若从场景文件中解析到非空数组）；
5. 每隔 2 秒轮询任务状态 `/task/{id}`：
   - 状态变化时在终端打印；
6. 当任务进入 `RUNNING`、`COMPLETED` 或 `FAILED` 状态时，开始轮询 `/task/{id}/logs` 输出日志内容（增量打印）；
7. 当状态为：
   - `COMPLETED`：打印“任务已完成”并退出；
   - `FAILED`：打印失败原因并以错误码退出；
   - `CANCELLED`：提示任务已被取消并退出。

---

### 5. `miaosuan update-stub`

miaosuan-stub 包是妙算引擎的 Python 接口提示包，包含类型定义与接口说明，方便在本地 IDE 中获得代码补全与静态检查支持。

在当前 Workspace 的虚拟环境中更新 `miaosuan-stub` Python 包：

```bash
cd <workspace-root>
miaosuan update-stub
```

等价于在工作区根目录执行：

```bash
uv pip install --index=https://pypi.znetlink.com/simple --upgrade miaosuan-stub
```

> 建议在平台更新了 stub 包版本后执行一次，以获得最新的类型提示与接口定义。

---

### 6. `miaosuan check-update`

手动检查 CLI 是否有新版本：

```bash
miaosuan check-update
```

行为：

1. 调用固定的版本元信息地址（默认 `https://pypi.znetlink.com/miaosuan-cli/latest.json`）；
2. 对比本地 `cliVersion` 与远端 `latest` 字段；
3. 输出当前版本、新版本以及下载地址；
4. 同时更新 `~/.miaosuan/config.json` 中的：
   - `last_version_check_time`
   - `last_known_latest_version`

> 若自动更新检查被关闭（`MIAOSUAN_NO_UPDATE_CHECK=1`），你仍可以手动运行此命令。

---

### 7. `miaosuan version`

显示当前 CLI 版本：

```bash
miaosuan version
# 或
miaosuan --version
```

版本号在构建时通过 `-ldflags "-X main.cliVersion=<version>"` 注入，例如使用 `build.sh` 脚本会自动从 git tag 推导。

---

## 文件映射规则

本地相对路径与云端 OSS 对象键之间的映射规则如下：

- 本地 `pycodes/**`：
  - `pycodes/<anything>` → `/models/pycodes/<anything>`
- 本地进程模型：
  - `procs/<name>.pr.m` → `/models/procs/<name>.pr.m`
- 本地管道阶段模型：
  - `pipelines/<name>.ps.m` → `/models/pipelines/<name>.ps.m`
- 其他未匹配路径：
  - `<rel>` → `/models/pycodes/<rel>`

> 换言之，只要你的 Python 代码都放在 `pycodes/` 下，模型定义放在 `procs/` 和 `pipelines/` 下，就可以和服务器约定好的存储结构无缝对应。

---

## 典型工作流示例

1. **首次绑定 Workspace**

   ```bash
   # 1. 初始化本地配置（首次运行会提示输入 server_url 与 api_key）
   miaosuan init <workspace-id>
   cd <workspace-name>
   ```

2. **本地开发**

   - 在 `pycodes/<模型名>/` 中编写/修改 Python 代码；
   - 如需新建进程模型或管道阶段，编辑对应的 `procs/*.pr.m` 与 `pipelines/*.ps.m` 文件；
   - CLI 在 `push` 时会自动更新其中的 `pythonPackageFiles` 列表。

3. **提交修改**

   ```bash
   miaosuan push
   ```

4. **从云端拉最新版本**

   ```bash
   miaosuan pull
   # 如有冲突，本地版本会自动备份为 *.local.bak
   ```

5. **运行仿真任务**

   ```bash
   miaosuan run demo_scenario
   # 过程中会提示是否先 push
   ```

6. **更新 stub 包**

   ```bash
   miaosuan update-stub
   ```

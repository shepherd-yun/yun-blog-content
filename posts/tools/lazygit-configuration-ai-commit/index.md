---
title: LazyGit 使用与配置：常用快捷键和 AI 约定式提交
slug: lazygit-configuration-ai-commit
tags: [LazyGit, Git, CLI, Conventional Commits, AIChat, delta]
published: true
excerpt: 从 LazyGit 的面板与常用快捷键入手，完整拆解我的 Cursor、delta 和自定义命令配置，并介绍如何按大写 C 调用 AIChat，根据暂存区差异生成带 Body 与 Refs Footer 的约定式提交。
cover: ./lazygit-configuration-ai-commit.png
metaTitle: LazyGit 使用与配置指南：快捷键、delta 与 AI 约定式提交
metaDescription: 介绍 LazyGit 常用快捷键、文件暂存、分支、提交、stash 操作，解析 Cursor 与 delta 配置，并用 AIChat 自定义命令生成 Conventional Commits 提交信息。
metaKeywords: LazyGit,LazyGit 配置,LazyGit 快捷键,Conventional Commits,约定式提交,AIChat,git delta,Git TUI
---

Git 命令本身并不难，真正消耗注意力的是不断确认上下文：哪些文件改了、哪些已经暂存、当前在哪个分支、这次提交包含什么、rebase 会影响哪些 commit。LazyGit 把这些状态放进一个终端界面，让暂存、提交、分支、日志、stash 和远程操作都围绕当前选中项完成。

![LazyGit、Git diff 与 AI 约定式提交工作流](./lazygit-configuration-ai-commit.png)

我的日常终端工作流里，LazyGit 负责 Git 操作，Cursor 负责打开文件，delta 负责渲染 diff，AIChat 则根据暂存区内容生成约定式提交。最常用的一条路径已经缩短成：

```text
检查 diff → 暂存文件或行 → 按大写 C → 输入可选需求 ID → AI 生成提交信息 → 完成 commit
```

本文基于我本机的 LazyGit 0.64.1 和 AIChat 0.30.0 整理。LazyGit 的按键具有面板上下文，同一个键在不同面板可能执行不同动作；不确定时，随时按 `?` 查看当前面板的实际帮助。

## LazyGit 是什么

LazyGit 是一个 Git TUI，也就是运行在终端里的文本界面。它没有重新实现 Git，而是把 `git status`、`git add`、`git commit`、`git branch`、`git rebase`、`git stash` 等命令组织成可浏览、可选择的交互界面。

这种方式有几个直接好处：

- 工作区、暂存区和提交历史同时可见，不必在多条命令之间来回确认状态。
- 文件、hunk 和单行暂存都能直接操作，提交边界更容易控制。
- branch、commit、stash 都是可选择的对象，merge、rebase、cherry-pick 不需要先复制一串名称或 hash。
- 每个面板只显示当前上下文可执行的动作，按 `?` 就能获得现场帮助。

在 macOS 上可以使用 Homebrew 安装：

```bash
brew install lazygit
```

启动时进入一个 Git 仓库，然后执行：

```bash
lazygit
```

也可以指定仓库路径：

```bash
lazygit --path /path/to/repository
```

## 先理解界面和移动方式

LazyGit 左侧默认由多个上下文面板组成，通常包括 Status、Files、Local Branches、Commits 和 Stash；右侧主视图显示当前选中项的详细内容，例如文件 diff、提交内容或分支日志。

最基础的移动方式接近 Vim：

| 按键 | 作用 |
|---|---|
| `j` / `↓` | 移到下一项 |
| `k` / `↑` | 移到上一项 |
| `h` / `←` | 移到左侧面板 |
| `l` / `→` / `Tab` | 移到右侧或下一个面板 |
| `1` ～ `5` | 直接跳到对应的左侧面板 |
| `0` | 聚焦右侧主视图 |
| `Enter` | 进入当前项的下一级内容 |
| `Esc` | 返回或取消当前操作 |
| `/` | 搜索或过滤当前视图 |
| `?` | 查看当前上下文的按键帮助 |
| `q` | 退出 LazyGit |

右侧 diff 很长时，不必先把焦点切过去。可以用 `K`、`Ctrl+U` 向上滚动主视图，用 `J`、`Ctrl+D` 向下滚动。按 `+`、`_` 可以在普通、半屏和全屏模式之间切换，检查大段 diff 时很实用。

## Files 面板：暂存和提交

Files 是日常使用频率最高的面板。它把 unstaged 与 staged 状态放在一起，也是普通提交和我的 AI 提交命令所在的上下文。

| 按键 | 作用 |
|---|---|
| `Space` | 暂存或取消暂存当前文件 |
| `a` | 暂存全部文件；再次按下可取消全部暂存 |
| `Enter` | 进入文件 diff，按 hunk 或单行暂存 |
| `c` | 提交已暂存的更改 |
| `A` | 用已暂存内容 amend 最近一次提交 |
| `e` | 用外部编辑器打开当前文件 |
| `o` | 用系统默认程序打开当前文件 |
| `d` | 打开丢弃更改的操作 |
| `s` | stash 全部更改 |
| `S` | 打开更细的 stash 选项 |
| `f` | fetch 远程更新 |
| `` ` `` | 在文件树和平铺列表之间切换 |
| `C` | 在我的配置中，调用 AIChat 生成约定式提交 |

### 按行暂存

如果一个文件同时包含两个目的不同的修改，不要为了省事把整个文件都放进同一个 commit。选中文件按 `Enter` 进入 staging 视图后：

| 按键 | 作用 |
|---|---|
| `h` / `←` | 上一个 hunk |
| `l` / `→` | 下一个 hunk |
| `Space` | 暂存或取消暂存当前 hunk / 选中行 |
| `a` | 在整段 hunk 与逐行选择模式之间切换 |
| `v` | 开始或结束范围选择 |
| `Tab` | 在 staged 和 unstaged diff 之间切换 |
| `e` | 在外部编辑器中打开文件 |
| `Esc` | 回到 Files 面板 |

AI 生成提交信息时读取的是 `git diff --cached`，也就是 staged diff。因此暂存不是提交前随手点一下，而是在明确告诉 AI 和 Git：“这一次提交只包含这些内容。”

## Branches 面板：切换、合并与 rebase

Local Branches 面板适合把“当前分支”和“选中分支”放在一起理解。涉及两个分支的操作一定要留意方向。

| 按键 | 作用 |
|---|---|
| `Space` | checkout 选中的分支 |
| `n` | 新建分支 |
| `c` | 输入名称并 checkout 分支 |
| `-` | 切回上一个分支 |
| `d` | 删除选中的本地或远程分支 |
| `R` | 重命名分支 |
| `M` | 把选中的分支 merge 到当前分支 |
| `r` | 把当前分支 rebase 到选中的分支之上 |
| `u` | 设置或取消 upstream |
| `f` | fast-forward 或获取远程更新，具体动作取决于当前子面板 |
| `Enter` | 查看选中分支的提交 |

例如当前位于 `feature/login`，光标选中 `main`：按 `M` 是把 `main` 合并进 `feature/login`；按 `r` 是把 `feature/login` rebase 到 `main` 之上。第一次操作前先看确认框里的分支名称，不要只凭按键记忆方向。

## Commits 面板：整理提交历史

Commits 面板不仅能看日志，也能执行 reword、squash、fixup、rebase、revert 和 cherry-pick。

| 按键 | 作用 |
|---|---|
| `Enter` | 查看当前 commit 修改的文件 |
| `r` | 修改提交信息（reword） |
| `A` | 把暂存区内容 amend 到选中的 commit |
| `s` | 把选中的 commit squash 到它下面的 commit |
| `f` | 把选中的 commit 作为 fixup 合入下面的 commit |
| `F` | 针对选中的 commit 创建 `fixup!` 提交 |
| `i` | 开始交互式 rebase |
| `C` | 标记 commit 为待复制，也就是复制 cherry-pick 对象 |
| `V` | 把已复制的 commit cherry-pick 到当前分支 |
| `t` | 为选中的 commit 创建 revert commit |
| `g` | 打开 reset 选项 |
| `Space` | checkout 该 commit，进入 detached HEAD |
| `y` | 复制 hash、消息、作者等 commit 属性 |

这里也能看出按键上下文的重要性：Files 面板的大写 `C` 被我改成了 AI 提交，而 Commits 面板的大写 `C` 仍然表示复制 cherry-pick 对象。

`reword`、`squash`、`fixup` 和对旧 commit 的 amend 都可能改写历史。如果相关提交已经推送并被其他人基于它继续开发，操作前要先确认协作方式，不要把“界面里按一下”误认为没有影响。

## Stash 面板：临时保存工作区

在 Files 面板按 `s` 可以快速 stash；需要选择已经存在的 stash 时，进入 Stash 面板：

| 按键 | 作用 |
|---|---|
| `Space` | apply：应用 stash，但保留 stash 记录 |
| `g` | pop：应用 stash，并在成功后移除记录 |
| `d` | drop：删除 stash 记录 |
| `r` | 重命名 stash |
| `n` | 从 stash 创建新分支 |
| `Enter` | 查看 stash 中的文件 |

不确定改动能否干净应用时，优先 `Space` 执行 apply；确认结果没有问题后再删除 stash。`g` 更省一步，但冲突处理和 stash 是否保留需要额外留意。

## 全局常用操作

有些动作不依赖左侧面板，可以在大多数上下文直接使用：

| 按键 | 作用 |
|---|---|
| `p` | pull 当前分支 |
| `P` | push 当前分支 |
| `R` | 刷新本地 Git 状态，不等于 fetch |
| `z` | 根据 reflog 撤销最近一次受支持的 Git 操作 |
| `Z` | 重做刚撤销的 Git 操作 |
| `Ctrl+S` | 打开提交日志过滤选项 |
| `Ctrl+W` | 显示或忽略纯空白变化 |
| `\|` / `\` | 在已配置的 diff renderer 之间正向 / 反向切换 |
| `:` | 输入并执行一条 shell 命令 |
| `@` | 查看命令日志选项 |

`z` 不是通用的文件恢复键。LazyGit 会参考 reflog 撤销它能够识别的 Git 操作，但不会替你恢复任意工作区编辑。涉及 reset、discard、drop stash 等动作时，仍然要认真阅读确认框。

## 我的 LazyGit 配置

macOS 默认把全局配置放在：

```text
~/Library/Application Support/lazygit/config.yml
```

Linux 通常位于 `~/.config/lazygit/config.yml`。如果设置过 `XDG_CONFIG_HOME`，实际位置可能变化，可以直接查询：

```bash
lazygit --print-config-dir
```

下面是我当前使用的完整配置。它只覆盖我确实需要修改的部分，没有复制整份默认配置。

```yaml
# 编辑器：lazygit 内置 preset 里没有 cursor，靠 $EDITOR 猜会退回终端 vim，
# 所以照搬内置 vscode preset 的模板，把 code 换成 cursor（两者 CLI 参数一致）
os:
  edit: 'cursor --reuse-window -- {{filename}}'
  editAtLine: 'cursor --reuse-window --goto -- {{filename}}:{{line}}'
  editAtLineAndWait: 'cursor --wait --goto -- {{filename}}:{{line}}'
  openDirInEditor: 'cursor --reuse-window -- {{dir}}'
  # false = 不挂起终端，让 cursor 以 GUI 方式打开
  editInTerminal: false
# 使用 delta 作为 diff 分页器（继承 ~/.gitconfig 的 [delta] 全部配色与 side-by-side 设置）
git:
  diffRenderers:
    - # git diff 输出始终带颜色，再交给 delta 渲染
      colorArg: always
      # --paging=never：滚动由 lazygit 负责，delta 不再启动自己的分页器(less)
      command: delta --paging=never
customCommands:
  - key: "C"
    description: "AI-powered conventional commit"
    context: files
    prompts:
      - type: input
        key: reqId
        title: "需求 ID（多个用英文逗号分隔；留空则不生成 footer）："
        initialValue: ""
    command: |
      bash -c '
      set -e

      if git diff --cached --quiet; then
        echo "没有暂存的更改。请先添加文件！"
        exit 1
      fi

      # $1 是 lazygit 弹框输入的需求 ID：白名单过滤掉可能干扰 shell/footer 的字符，再去掉首尾空白
      req_id="$(printf "%s" "$1" | tr -cd "A-Za-z0-9#_./, -" | sed "s/^[[:space:]]*//; s/[[:space:]]*\$//")"

      if [ -n "$req_id" ]; then
        extra_rule="8. 本次改动对应的需求 ID 是：${req_id}，可作为背景参考。但你绝对不要在输出中包含任何 footer（例如 Refs:、Closes:），footer 由脚本统一追加。"
      else
        extra_rule="8. 不要输出任何 footer（例如 Refs:、Closes:）。"
      fi

      msg_file="$(mktemp)"
      trap "rm -f \"$msg_file\"" EXIT

      git diff --cached | aichat "你是一个编写 Git 提交信息的专家。请根据 staged diff 自动生成一个符合 Conventional Commits 规范的 Git commit message。

      严格格式如下：
      <type>(<scope>): <简短中文描述>

      - 详细改动条目 1
      - 详细改动条目 2

      要求：
      1. type 由你根据代码变更自动判断，例如 feat、fix、refactor、chore、docs、test、style、perf。
      2. scope 由你根据变更模块自动判断，例如 backend、frontend、pom、config、api、db 等。
      3. 第一行是标题 Header，格式必须是：<type>(<scope>): <简短中文描述>
      4. 第二行必须为空行。
      5. 第三行开始是正文 Body，用中文列表描述详细改动。
      6. 正文尽量 5 条以内，讲清关键的架构、逻辑、配置变动，做了什么或为什么做即可。
      7. 只输出最终 commit message，不要输出序号、不要 markdown 代码块、不要解释。
      $extra_rule" > "$msg_file"

      if [ ! -s "$msg_file" ]; then
        echo "AI 没有生成提交信息"
        exit 1
      fi

      # 追加 footer：$(cat) 会吃掉尾部换行，保证 body 与 footer 之间正好一个空行
      if [ -n "$req_id" ]; then
        printf "%s\n\nRefs: %s\n" "$(cat "$msg_file")" "$req_id" > "$msg_file.tmp"
        mv "$msg_file.tmp" "$msg_file"
      fi

      git commit -F "$msg_file"
      ' _ "{{.Form.reqId}}"
    loadingText: "aichat is thinking..."
    output: log
```

这份配置可以分成三个部分。

### 用 Cursor 打开文件

`os` 下的四条命令分别处理普通编辑、跳到指定行、打开并等待编辑器退出，以及打开目录：

- `{{filename}}`、`{{line}}` 和 `{{dir}}` 是 LazyGit 在执行时替换的模板变量。
- `--reuse-window` 复用已经打开的 Cursor 窗口，避免每次编辑都启动一个新窗口。
- `--goto` 让冲突处理或跳转定位可以直接打开到具体行。
- `editAtLineAndWait` 使用 `--wait`，适合 Git 必须等编辑器完成后才能继续的流程。
- `editInTerminal: false` 表示 Cursor 是外部 GUI 编辑器，LazyGit 不需要为它挂起整个终端界面。

因此，在 Files、Commits 或冲突处理界面按 `e` 时，文件会进入 Cursor，而不是因为 `$EDITOR` 推断失败退回终端 Vim。

### 用 delta 渲染 diff

`git.diffRenderers` 是一个有顺序的 renderer 列表。这里仅配置 delta：

```yaml
git:
  diffRenderers:
    - colorArg: always
      command: delta --paging=never
```

`colorArg: always` 让 Git 始终输出颜色信息，delta 再按 `~/.gitconfig` 中已有的主题和 side-by-side 设置渲染。`--paging=never` 禁止 delta 再启动 `less`，滚动仍由 LazyGit 的主视图管理，不会出现分页器套分页器的问题。

以后如果增加多个 renderer，可以用全局按键 `|` 和 `\` 循环切换。

### 把大写 C 变成 AI 提交

自定义命令的关键字段是：

| 字段 | 当前配置中的作用 |
|---|---|
| `key: "C"` | 绑定大写 `C` |
| `context: files` | 只在 Files 面板生效 |
| `description` | 显示在帮助和命令描述中 |
| `prompts` | 提交前弹出需求 ID 输入框 |
| `{{.Form.reqId}}` | 读取带 `key: reqId` 的表单值 |
| `command` | 执行生成和提交脚本 |
| `loadingText` | AI 生成期间显示等待提示 |
| `output: log` | 把命令输出写入 LazyGit 日志 |

LazyGit 默认在 Files 面板把大写 `C` 用作“通过 Git 编辑器提交”。这条自定义命令占用了同一个上下文的按键，因此默认动作被 AI 提交流程替代；其他面板中的 `C` 不受影响。

## 大写 C 的执行流程

按下大写 `C` 后，脚本按以下顺序执行。

### 1. 只接受已经暂存的改动

```bash
if git diff --cached --quiet; then
  echo "没有暂存的更改。请先添加文件！"
  exit 1
fi
```

如果暂存区为空就直接结束。这样可以避免 AI 根据整个工作区猜提交范围，也不会把尚未决定是否提交的改动带进去。

### 2. 获取并清理需求 ID

弹框允许输入一个或多个需求 ID，多个 ID 使用英文逗号分隔。脚本只保留字母、数字以及 `#_./, -` 等白名单字符，并去掉首尾空白。

需求 ID 为空时不生成 Footer；不为空时，AI 可以把它当作背景，但不能自己输出 `Refs:` 或 `Closes:`。这条约束是为了让 Footer 始终由脚本以确定格式追加，避免重复。

### 3. 把 staged diff 通过 stdin 交给 AIChat

核心管道是：

```bash
git diff --cached | aichat "...提示词..." > "$msg_file"
```

AIChat 的命令行模式可以读取标准输入，因此左侧是 staged diff，命令行参数则负责规定输出格式。生成结果先写入 `mktemp` 创建的临时文件；`trap` 会在脚本退出时清理它。

这里要注意数据边界：staged diff 会被发送给 AIChat 当前配置的模型服务。包含密钥、客户数据、内部地址或其他敏感信息的改动，不应该在没有确认模型与数据策略的情况下直接送出。

### 4. 由脚本追加 Footer

如果输入了需求 ID，脚本会在 AI 生成的 Body 后保留一个空行，再追加：

```text
Refs: REQ-123,REQ-456
```

最后通过下面的命令提交：

```bash
git commit -F "$msg_file"
```

也就是说，大写 `C` 不是只生成一段文字供预览，而是生成成功后直接创建 commit。提交完成后应在 Commits 面板快速检查结果；需要修改时，可以选中刚才的 commit 按 `r` reword。

## 约定式提交的完整格式

[Conventional Commits 1.0.0](https://www.conventionalcommits.org/zh-hans/v1.0.0/) 的结构可以写成：

```text
<type>[可选 scope][可选 !]: <description>

[可选 Body]

[可选 Footer]
```

我的 AI 脚本进一步把它收窄为：必须有 `type`、`scope`、中文 description 和中文列表 Body，Footer 则由需求 ID 决定。

### Header：一行说明这是什么改动

Header 的基本格式是：

```text
<type>(<scope>): <简短描述>
```

例如：

```text
feat(api): 增加订单批量取消接口
fix(auth): 修复令牌刷新时的并发覆盖
refactor(db): 合并重复的分页查询逻辑
docs(lazygit): 补充 AI 提交配置说明
```

各部分含义如下：

- `type` 表示改动类别。规范明确要求新增功能使用 `feat`，修复缺陷使用 `fix`。
- `scope` 是可选的影响范围，放在括号中，例如 `api`、`frontend`、`db`、`config`。我的脚本要求 AI 自动补全它。
- 冒号后必须有一个空格，再写简短 description。
- description 说明结果，不必重复文件名，也不要在一行里塞入所有实现细节。

工程中常见的 type 包括：

| type | 适用场景 |
|---|---|
| `feat` | 新增用户可感知的功能 |
| `fix` | 修复缺陷 |
| `refactor` | 不改变外部行为的代码重构 |
| `perf` | 性能优化 |
| `docs` | 只修改文档 |
| `test` | 新增或调整测试 |
| `style` | 不影响逻辑的格式、空白等修改 |
| `build` | 构建系统或外部依赖变更 |
| `ci` | CI 配置与脚本变更 |
| `chore` | 难以归入以上类别的维护工作 |

`feat` 和 `fix` 是规范定义了语义的核心类型，其他 type 可以由团队约定。项目里最重要的不是把类别拆得越细越好，而是长期保持一致。

### Body：说明做了什么以及为什么

Body 必须和 Header 之间隔一个空行，内容本身是自由格式，可以是一段或多段文字。我的提示词要求使用不超过五条的中文列表：

```text
feat(api): 增加订单批量取消接口

- 新增批量取消请求模型并校验订单编号列表
- 复用单订单取消规则，统一状态与权限检查
- 补充部分失败时的错误明细和接口测试
```

好的 Body 应该补充 Header 容纳不下的信息：

- 关键行为或数据结构发生了什么变化。
- 为什么选择这种实现，尤其是反直觉的取舍。
- 兼容性、迁移、风险或测试范围有哪些变化。

Body 不需要把 diff 逐行翻译一遍。诸如“修改了 `FooService.java`”“增加了一个 if”通常不能帮助后来的人理解提交目的。

### Footer：关联事项与不兼容变更

Footer 与 Body 之间同样需要一个空行。每条 Footer 使用类似 Git trailer 的形式：token 后接 `: ` 或 ` #`，例如：

```text
Refs: REQ-123
Closes: #456
Reviewed-by: Yun
```

token 中如果需要分隔单词，应使用连字符，例如 `Reviewed-by`。`BREAKING CHANGE` 是规范允许包含空格的特殊 token。

我的脚本使用 `Refs:` 表示“这次提交关联哪些需求”，而不是声明合并后自动关闭某个 Issue。输入多个需求 ID 时会保留为同一条 Footer：

```text
Refs: REQ-123,REQ-456
```

### Breaking Change：明确不兼容变化

不兼容变化有两种标准写法。第一种是在 type 或 scope 后加 `!`：

```text
feat(api)!: 移除旧版订单查询参数
```

第二种是在 Footer 中明确说明：

```text
feat(api): 调整订单查询参数

- 使用统一过滤对象替代多个独立查询参数

BREAKING CHANGE: 客户端必须改用 filter 对象传递查询条件
```

两种方式也可以同时使用。按照约定式提交与语义化版本的对应关系，`fix` 通常对应 PATCH，`feat` 通常对应 MINOR，Breaking Change 通常对应 MAJOR。是否真正自动发版，仍取决于项目的 release 工具和团队规则。

## 一次完整的提交示例

假设我修改了 LazyGit 文章和封面，先在 Files 面板逐项检查，按 `Space` 暂存，确认 staged diff 只有这篇文章的内容，然后按大写 `C`，输入：

```text
BLOG-42
```

最终生成的 commit message 可能是：

```text
docs(lazygit): 增加配置与 AI 提交使用指南

- 整理文件、分支、提交和 stash 面板的常用快捷键
- 说明 Cursor 外部编辑器与 delta diff renderer 配置
- 拆解 AIChat 根据 staged diff 生成提交信息的执行流程
- 补充 Header、Body、Footer 与 Breaking Change 格式

Refs: BLOG-42
```

这条消息里，Header 方便日志快速浏览，Body 解释提交范围，Footer 建立需求关联。更重要的是，它来自已经整理好的暂存区：AI 负责表达，提交边界仍然由人决定。

## 我的实际使用习惯

最后把日常流程收敛成几条规则：

1. 先读 diff，再决定暂存内容；不要让 AI 替我决定 commit 边界。
2. 一个 commit 只表达一个可说明的目的，必要时按 hunk 或行拆分。
3. 普通小改动按 `c` 手写提交；需要完整 Body 和需求关联时按大写 `C`。
4. AI 提交完成后立即看一次 Header、Body 和 Footer，scope 或描述不准确就按 `r` 修改。
5. 已推送的历史谨慎做 squash、fixup、amend 和 rebase。
6. 忘记按键就按 `?`，它比任何静态速查表都更接近当前版本和当前面板。

LazyGit 真正节省的不是几次 `git add` 或 `git commit`，而是把 Git 状态持续放在眼前。再把 staged diff、约定式提交和 AIChat 串起来以后，提交信息的质量更稳定，但流程的控制权仍然留在清晰的暂存区和最后一次人工检查上。

## 参考资料

- [LazyGit 官方按键表](https://github.com/jesseduffield/lazygit/blob/master/docs/keybindings/Keybindings_en.md)
- [LazyGit 用户配置文档](https://github.com/jesseduffield/lazygit/blob/master/docs/Config.md)
- [LazyGit 自定义命令文档](https://github.com/jesseduffield/lazygit/blob/master/docs/Custom_Command_Keybindings.md)
- [LazyGit 自定义 diff renderer 文档](https://github.com/jesseduffield/lazygit/blob/master/docs/Custom_DiffRenderers.md)
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/zh-hans/v1.0.0/)
- [AIChat 官方仓库](https://github.com/sigoden/aichat)

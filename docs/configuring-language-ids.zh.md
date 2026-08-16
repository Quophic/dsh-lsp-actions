# 配置语言服务器与语言 ID（extensionToLanguage）

> 面向 dsh-lsp-actions 的用户。本文教你：安装一门语言的 LSP 服务器后，
> 怎么在配置里给它加一条 `servers` 条目，以及每个文件扩展名该填什么 `languageId`。
> 读完你能独立配好 TypeScript、JSX、Python、C# 等任何一门 dsh-lsp-actions 支持的语言。

## 0. 前置：dsh-lsp-actions 的配置入口在哪

dsh-lsp-actions 本身不内置任何语言服务器（`servers` 默认空表）。它靠 `servers` 配置表才知道
「用哪个可执行文件、认哪些扩展名」。配置表写在 **dsh profile 的 `cordis.patch.yml`** 里
（也可以用 `dsh plugin` 的 bundle 行或 `--patch` 覆盖，README 有说明）。模块里有一个 `lsp-actions`
条目，它的 `config.servers` 就是我们要填的地方：

```yaml
- id: lsp-actions
  config:
    servers:
      # 在这里给每门语言加一个条目
```

**要点**：`servers` 表为空时，`lsp_*` 工具调用会响亮失败（`LSP_ACTION_UNAVAILABLE`），
并提示你去配置——它**绝不会**擅自启动你没配置的服务器。

## 1. 为什么 languageId 填错会出大事

dsh-lsp-actions 拿到一个文件，按**扩展名**查 `extensionToLanguage`，得到 `languageId`，
再把这个 id 告诉语言服务器。**服务器完全信赖这个 id** 来决定怎么解析文件：

- id 对 → 服务器用正确语法解析（JSX 当 JSX、纯 TS 当纯 TS）
- id 错 → 服务器用错规则，报一堆看似「语法错误」的假错

**关键：languageId 不是你编的，是语言服务器自己声明「我认哪些 id」。查得到才是对的，拍脑袋填必错。**

## 2. 填错的典型症状（先认识它，免得被误导）

最典型的是 `.tsx`/`.jsx`（含 JSX）被填成普通 TS/JS 的 id。服务器把 `<div>` 当泛型/比较符、
把 `/` 当正则，于是每个 JSX 标签都报错——**但文件能正常编译**，这些是假错。

| `lsp_diagnostics` 报错 | 含义 |
|---|---|
| `'>' expected` / `')' expected` | JSX 的 `<...>` 被当成比较/泛型 |
| `Unterminated regular expression literal` | 模板里的 `/` 被当正则起始 |
| `Operator '<' cannot be applied to types 'boolean' and 'RegExp'` | `<` 被当比大小 |
| 很小的 `.tsx` 也满屏语法错（但能编译） | 几乎必是 id 配错，不是文件坏 |

**鉴别真错/假错**：假错的最大特征是「明明能编译，却报一片语法错误」，尤其当报错把 `<` 当运算符、
把 `/` 当正则——正常代码不会这样，这是解析模式不对的信号。

## 3. 三步：给一门语言配对 languageId

### 第 1 步：判断这门语言有几种「文件类型」

先想清楚有没有「普通文件」和「特殊语法变体」之分：

| 语言 | 文件类型 | 需要的 id 数 |
|---|---|---|
| TypeScript | `.ts`（纯）+ `.tsx`（含 JSX） | 两个 |
| JavaScript | `.js`（纯）+ `.jsx`（含 JSX） | 两个 |
| Python | `.py` 一种 | 一个 |
| C# | `.cs` 一种 | 一个 |
| Go / Rust | `.go` / `.rs` 一种 | 一个 |

**规则**：只有一种文件类型 → 一个 id；有 React/dialect 变体（TSX/JSX/Vue 等）→ 每个变体独立 id。

### 第 2 步：查这个服务器「认哪些 id」（权威来源）

**不要猜。** 两种方法都比拍脑袋可靠：

- **查服务器源码**（最准）：在服务器程序里搜 `languageId` / `mode` / `ScriptKind` 等，
  看它定义/分支了哪些 id。例如 `typescript-language-server` 的 `mode2ScriptKind` 明确写出：
  ```
  typescript       -> TS    （纯 TypeScript）
  typescriptreact  -> TSX   （TS + JSX，即 .tsx）
  javascript       -> JS    （纯 JavaScript）
  javascriptreact  -> JSX   （JS + JSX，即 .jsx）
  ```
  由此确定：`.tsx → typescriptreact`、`.jsx → javascriptreact`，一个字符都不差。
- **查官方文档 / 通用约定**：主流语言服务器的 languageId 是业界标准，基本稳定：
  `typescript`、`typescriptreact`、`javascript`、`javascriptreact`、`python`、`csharp`、
  `go`、`rust`、`json`、`xml`、`html`、`css`、`vue` …

> **记忆技巧**：React/dialect 变体多在基础 id 上加 `react` 后缀
> （`typescript → typescriptreact`）。但「约定之外的方言」仍要回源码确认，别只靠后缀硬猜。

### 第 3 步：探针实测（终审，最可靠）

配完用一个真实的小文件（如一个含 JSX 的 `.tsx`，或新语言的最简文件）跑 `lsp_diagnostics` 验证：

- 报第 2 节那种「把 `<` 当运算符」的假语法错 → id 配错了，回第 2 步查正确 id
- 无报错 / 只有真实类型提示 → id 正确，收工

实测是最终判定，比任何记忆都可靠——前两步判断失误也能被这一步兜住。

## 4. 一个正确配置示例（模板，非任何特定环境）

下面展示一个**结构完整**的 `servers` 配置长什么样，可作贴改模板（路径是示例占位，请填成你实际的）：

```yaml
- id: lsp-actions
  config:
    servers:
      typescript:
        command: /path/to/node
        args: ["/path/to/typescript-language-server/lib/cli.mjs", "--stdio"]
        extensionToLanguage:           # 扩展名 -> languageId 一一对应
          ".ts": typescript
          ".tsx": typescriptreact      # JSX 必须用 typescriptreact，不能是 typescript
          ".js": javascript
          ".jsx": javascriptreact      # JSX 必须用 javascriptreact，不能是 javascript
          ".mjs": javascript
          ".cjs": javascript
      python:
        command: /path/to/node
        args: ["/path/to/pyright/langserver.index.js", "--stdio"]
        extensionToLanguage:
          ".py": python
```

## 5. 加一门新语言：完整步骤

1. **安装**该语言的 LSP 服务器（`typescript-language-server`、`pyright`、`csharp-ls`、`gopls`、`rust-analyzer` …）。
2. 在 `cordis.patch.yml` 的 `servers` 下新增一个条目，填 `command` + `args` + `extensionToLanguage`。
3. 用探针文件跑 `lsp_diagnostics` 验证第 2 节/第 3 节的判定。

> 注意（Windows）：dsh-lsp-actions 用 no-shell spawn，**不能直接 spawn `.cmd` 脚本**（会 EINVAL）。
> 对以 `.cmd` 入口分发的服务器，应把 `command` 指到 `node.exe`、`args` 带真实 JS 入口 + `--stdio`。

> 没有服务器的语言，dsh-lsp-actions 不认识，`lsp_*` 会退回 `grep`/`read` 兜底——不会崩，只是没有真语法树。

## 6. 一句话总结

> **`languageId` 是服务器认不认文件的钥匙：先判断这门语言有几种文件类型，再查服务器认哪些 id
> （有 react/dialect 变体就给它独立 id），最后用探针实测兜底。不要拍脑袋填，
> 也不要因为「碰巧没报错」就当配对了。**

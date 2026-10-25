# 插件编写指南（manifest.json）

本文面向插件作者，讲清楚一件事：**如何从零写出一个能被「工具插件」App 加载运行的插件**。

插件 = 一份 `manifest.json`（外加可选的图标）。它声明：

1. **输入**：运行前向用户收集哪些信息（手机号、验证码……）
2. **流程**：按什么顺序发哪些 HTTP 请求、中途如何与用户交互
3. **输出**：运行结束后向用户展示哪些结果

引擎负责执行；插件只负责声明。整份文件是纯数据，**不含任何可执行代码**（唯一的例外是可选的 `set.js` 逃生舱，见[第 6 节](#6-js-沙箱set-js-逃生舱)）。

> 契约层权威定义见仓库根目录 `CONTRACT.md`；本文是面向作者的实操教程，两者冲突时以 `CONTRACT.md` 为准。

## 目录

1. [插件是什么](#1-插件是什么)
2. [最小可运行插件](#2-最小可运行插件)
3. [manifest 字段详解](#3-manifest-字段详解)
4. [flow：四种步骤类型](#4-flow四种步骤类型)
5. [表达式系统与 fn 函数库](#5-表达式系统与-fn-函数库)
6. [JS 沙箱（set.js 逃生舱）](#6-js-沙箱set-js-逃生舱)
7. [inputs：声明插件需要的信息](#7-inputs声明插件需要的信息)
8. [output：声明运行结果](#8-output声明运行结果)
9. [安全约束（必读）](#9-安全约束必读)
10. [调试技巧](#10-调试技巧)
11. [完整示例](#11-完整示例)
12. [错误码速查表](#12-错误码速查表)

---

## 1. 插件是什么

一个插件的核心就是**一份 `manifest.json`**。安装到手机后，它以 `plugins/<id>/manifest.json` 的形式存放——所以一个插件在磁盘上就是一个只含 manifest 的目录。

分发与导入时有两种等价的**纯文本**格式：

- **裸 manifest**：`manifest.json` 原文。
- **`.qbpkg` 信封**：给裸 manifest 套一层信封，附带导出信息：

```json
{
  "qbpkg": "1.0",
  "manifest": { "id": "demo.hello", "……": "完整 manifest 原样内嵌" },
  "meta": { "exportedAt": "……", "exportedBy": "……", "note": "" }
}
```

App 导入时自动识别两种格式；导出插件时生成信封格式。`.qbpkg` 是纯文本 JSON——肉眼可读、可 diff、可粘贴，**不是** zip 压缩包。

顶层字段概览（★ 为必填）：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | ★ | 唯一标识，至少两段小写点分，如 `demo.hello` |
| `name` | string | ★ | 展示名，1–64 字符 |
| `version` | string | ★ | 语义化版本，如 `1.0.0` |
| `engine` | string | ★ | 引擎版本范围，如 `">=1.0 <2.0"`，不满足则拒绝加载 |
| `allowHosts` | string[] | ★ | 网络白名单，非空，见[第 9 节](#9-安全约束必读) |
| `inputs` | array | ★ | 表单字段定义，可为空数组 |
| `flow` | array | ★ | 步骤数组，非空 |
| `output` | object | ★ | 结果展示定义 |
| `author` | string | — | 作者名，0–128 字符 |
| `description` | string | — | 简介，0–512 字符 |
| `vars` | object | — | 静态/模板变量，按声明顺序求值 |
| `secrets` | string[] | — | 声明需从安全存储读取的键 |
| `allowInsecureLocalhost` | bool | — | 默认 `false`，仅测试用，见第 9 节 |
| `hooks` | object | — | JS 沙箱配置（内置库、超时），见第 6 节 |

**未知字段一律拒绝**（加载时报 `E_SCHEMA`）。这是刻意的：字段名拼错时宁可失败，也不要静默失效。

## 2. 最小可运行插件

下面是一个完整可运行的插件：向 `api.example.com` 发一个 GET 请求，把响应里的 `data.token` 展示给用户。

```json
{
  "id": "demo.hello",
  "name": "最简示例",
  "version": "1.0.0",
  "engine": ">=1.0 <2.0",
  "allowHosts": ["api.example.com"],
  "inputs": [
    {
      "key": "phone",
      "label": "手机号",
      "type": "phone",
      "required": true
    }
  ],
  "flow": [
    {
      "id": "fetch",
      "type": "request",
      "request": {
        "method": "GET",
        "url": "https://api.example.com/demo/token?phone={{input.phone}}"
      },
      "extract": [
        { "name": "token", "from": "json", "path": "$.data.token" }
      ]
    }
  ],
  "output": {
    "title": "获取成功",
    "fields": [
      { "label": "Token", "value": "{{steps.fetch.token}}" }
    ]
  }
}
```

把这份 JSON 保存为 `manifest.json`，在 App 里用「粘贴导入」即可运行。

仓库自带两个完整的教学插件，建议对照阅读：

- `plugins/demo.pipeline/`：不访问任何网络（`allowHosts` 用的是不存在的 `demo.invalid`），专门演示管道式写法、bytes 的 hex 中转、断言写法。
- `plugins/demo.setjs.utf16le/`：演示 `set.js` 逃生舱——用一段 JS 把文本编码成 UTF-16LE，这是声明式 `{{fn.*}}` 做不到的。

## 3. manifest 字段详解

### 3.1 标识类

- **`id`**：正则 `^[a-z0-9]+(\.[a-z0-9]+)+$`，至少两段，全小写。批量导入时跨文件查重。惯用 `<作者前缀>.<功能>`，如 `demo.hello`。
- **`name` / `version` / `author` / `description`**：纯展示信息。`version` 必须形如 `\d+\.\d+\.\d+`。
- **`engine`**：版本范围表达式，如 `">=1.0 <2.0"`。App 内置引擎版本不在此范围时，插件拒绝加载（`E_ENGINE`）。

### 3.2 `vars`：顶层静态变量

```json
"vars": {
  "base": "https://api.example.com",
  "ua": "{{env.appVersion}}",
  "phoneE164": "{{fn.concat('+86', input.phone)}}"
}
```

- 值是模板字符串，**按键的声明顺序求值**，后面的键可以引用前面的键。
- `set` 步骤在运行中可以覆盖或新增这里的值。
- 引用不存在的变量 → `E_UNDEF`（引擎不做静默 null，这是刻意的：静默 null 会让协议错误极难排查）。

### 3.3 `secrets`：敏感键声明

```json
"secrets": ["apiKey"]
```

声明后，流程里用 `{{secrets.apiKey}}` 引用，值由宿主在运行时从安全存储注入。要点：

- `secrets.*` 的值**不写日志、不落盘**，错误消息里也强制脱敏。
- 引用未声明的键 → `E_SCHEMA`。
- 适合放服务器侧颁发的固定密钥之类的东西；**不要**用来放用户密码（用户输入走 `inputs` 的 `password` 类型，同样有日志脱敏保护）。

## 4. flow：四种步骤类型

`flow` 是步骤数组，从 `flow[0]` 开始顺序执行。每个步骤的公共字段：

```json
{
  "id": "login",
  "type": "request",
  "when": "input.phone != ''",
  "goto": "next",
  "branch": [
    { "when": "steps.init.code == '1016'", "goto": "captcha" }
  ],
  "onFail": { "message": "{{response.message}}" },
  "end": false
}
```

| 字段 | 说明 |
|---|---|
| `id` | 必需。`^[a-zA-Z_][a-zA-Z0-9_]*$`，数组内唯一 |
| `type` | 必需。`request` \| `poll` \| `ui` \| `set` |
| `when` | 可选。条件表达式，求值为 `false` 则跳过本步 |
| `goto` | 可选。跳转到指定 step id；省略或 `"next"` = 顺序执行 |
| `branch` | 可选。按序求值，**首个为真者生效**，优先于 `goto` |
| `onFail` | 可选。本步失败时的处理，见 [4.5](#45-onfail) |
| `end` | 可选。`true` = 正常结束进 output；`"error"` = 失败结束但**仍渲染 output**（用于失败时把诊断值给用户看）。其他取值 → `E_SCHEMA` |

执行规则：

- 每步执行完按 **`branch` → `goto` → 顺序** 的优先级决定下一步。
- `goto`/`branch` 指向不存在的 id → 加载期报 `E_GOTO`。
- 任何步骤失败且未定义 `onFail` → 整个流程失败终止。
- 同一步骤累计执行次数超过 `flow.length * 3` → `E_LOOP`（防死循环）。

### 4.1 `type: "request"`：发 HTTP 请求

```json
{
  "id": "login",
  "type": "request",
  "request": {
    "method": "POST",
    "url": "{{vars.base}}/auth/login",
    "headers": {
      "Content-Type": "application/json;charset=UTF-8",
      "User-Agent": "DemoApp/1.0"
    },
    "query": { "page": "1" },
    "body": {
      "type": "json",
      "value": { "phone": "{{input.phone}}", "ts": "{{fn.ts14()}}" }
    },
    "timeoutMs": 20000,
    "retry": { "times": 2, "backoffMs": 1000, "on": [500, 502, 503, 504] },
    "followRedirects": true
  },
  "extract": [ /* 见 4.6 */ ],
  "assert": { /* 见 4.7 */ }
}
```

| 字段 | 约束 |
|---|---|
| `method` | `GET` / `POST` / `PUT` / `PATCH` / `DELETE` |
| `url` | 必须是 `https://`（例外见第 9 节）；host 必须命中 `allowHosts` |
| `query` | 值为模板字符串，自动 urlencode |
| `body` | GET 时禁止出现。`type` 取值见下表 |
| `timeoutMs` | 默认 20000，范围 1000–120000 |
| `retry.times` | 重试次数（不含首次），0–5 |
| `retry.on` | 默认 `[500, 502, 503, 504]`；可含 `"network"` 表示网络异常也重试 |
| `followRedirects` | 默认 `true`，每一跳都重新校验白名单，最多 5 跳 |

`body.type` 语义：

| type | `value` | 序列化 |
|---|---|---|
| `json` | object / array（递归渲染模板） | JSON 编码；未指定 Content-Type 时补 `application/json` |
| `form` | object | `application/x-www-form-urlencoded`，值 urlencode |
| `text` | string | 原样；未指定时补 `text/plain; charset=UTF-8` |
| `hex` | hex 字符串 | 解码为字节作为请求体；未指定时补 `application/octet-stream` |

### 4.2 `type: "poll"`：轮询等待

```json
{
  "id": "waitSms",
  "type": "poll",
  "poll": {
    "steps": ["smsStatus"],
    "until": "steps.smsStatus.status == '1'",
    "intervalMs": 1000,
    "maxAttempts": 5,
    "onTimeout": { "message": "短信状态尚未就绪" }
  }
}
```

- `steps`：要重复执行的 step id 列表，1–5 个。这些 id 必须是 `flow` 里真实存在的步骤。
- 执行顺序：先跑一遍 `steps`，再求 `until`；为真则继续，为假则等 `intervalMs`（默认 1000，范围 100–60000）后重试。
- `maxAttempts`（1–60）次后仍未满足 → 按 `onTimeout`（必需）处理。
- 典型场景：提交后隔几秒查一次结果。

### 4.3 `type: "ui"`：与用户交互

```json
{
  "id": "askSms",
  "type": "ui",
  "ui": {
    "type": "text",
    "label": "短信验证码",
    "input": "smsCode",
    "maxLength": 6,
    "pattern": "^\\d{6}$"
  }
}
```

`ui.type` 五种：

| type | 用途 | 关键字段 |
|---|---|---|
| `text` | 让用户输入一段文本（验证码、密码……） | `label`、`input`、`maxLength`、`pattern` |
| `image` | 图片验证码 | `image`（base64，支持模板）、`refreshStep`（点击刷新时重跑的步骤 id） |
| `confirm` | 让用户确认后继续 | `message`（正文） |
| `webview` | 滑块/网页验证码 | `url` 或 `html`+`baseUrl` 二选一，`bridge`（默认 `FlutterBridge`） |
| `select` | 流程内多选一 | `options`：`[{value, label}]`，静态字面量 |

通用规则：

- `ui` 步骤**阻塞等待用户**；用户取消 → 流程以「用户取消」终止（不算错误）。
- 用户提交的值写入 `input.<input 键>`，后续步骤用 `{{input.xxx}}` 引用。
- `label` / `hint` / `message` / `image` / `url` / `html` / `baseUrl` 支持模板；`type`、`input`、`options` 等结构字段不支持。
- `webview` 的 `url`（或 `html` 模式下的 `baseUrl`）host 必须在 `allowHosts` 内。网页通过 JS 桥回传结果：
  - 成功：`FlutterBridge.postMessage(JSON.stringify({ action: 'success', value: '字符串' }))`
  - 取消：`FlutterBridge.postMessage(JSON.stringify({ action: 'close' }))`

### 4.4 `type: "set"`：纯计算

```json
{
  "id": "calc",
  "type": "set",
  "set": {
    "vars": {
      "androidId": "{{fn.randomHex(16)}}",
      "ts": "{{fn.ts14()}}"
    }
  }
}
```

`set.vars` 里的键**按声明顺序求值**，后面的键可以引用前面刚算出的键（管道式写法）：

```json
"set": { "vars": {
  "a": "{{vars.seed}}",
  "b": "{{vars.a}}-1",
  "c": "{{vars.b}}-2"
} }
```

若 `seed = "abc"`，则 `a=abc`、`b=abc-1`、`c=abc-1-2`。长表达式拆成多行、每行只做一件事，可读性远好于一行 275 字符的 5 层嵌套。

⚠️ **bytes 不能存进 vars**：`set` 步的求值结果必须是字符串。`fn.unhex(...)` 的结果是 bytes，直接存会报 `E_TYPE`。bytes 链的写法是「每行 `fn.hex` 收尾、下一行 `fn.unhex` 开头」，用 hex 字符串中转：

```json
// ✅ 可行
"set": { "vars": {
  "h1": "{{fn.hex(fn.unhex(vars.inner))}}",
  "h2": "{{fn.hex(fn.bytesReplace(fn.unhex(vars.h1), fn.unhex('2b3836'), fn.utf8(input.phone)))}}"
} }
```

### 4.5 `onFail`

```json
"onFail": { "message": "{{response.message}}" }
"onFail": { "goto": "retryPath" }
```

- `message` 与 `goto` **互斥**：要么带着消息终止，要么跳转继续。
- `{{response.message}}` 只在 `onFail` 内可用：优先取响应体 JSON 的 `message` 字段，其次 HTTP 状态文本。

### 4.6 `extract`：从响应中取值

挂在 `request` 步上，数组，每项提取一个值，存入 `steps.<stepId>.<name>`：

```json
"extract": [
  { "name": "token", "from": "json", "path": "$.result.token" },
  { "name": "code",  "from": "json", "path": "$.a.b", "optional": true, "default": "" },
  { "name": "raw",   "from": "raw" },
  { "name": "ct",    "from": "header", "header": "Set-Cookie" },
  { "name": "tid",   "from": "regex", "pattern": "token=([a-f0-9]{32})", "group": 1 },
  { "name": "vs",    "from": "binary", "tag": "162b", "length": 43, "encoding": "ascii" }
]
```

| `from` | 必需字段 | 语义 |
|---|---|---|
| `json` | `path` | JSONPath 子集（见下） |
| `raw` | — | 整个响应体（bytes 类型） |
| `header` | `header` | 响应头，大小写不敏感；多值取最后一个 |
| `regex` | `pattern`，可选 `group`（默认 1） | 对响应体按 UTF-8 解码后匹配；`group: 0` 为整串 |
| `binary` | `tag`（hex）、`length`、`encoding` | 在响应体原始字节中定位 tag，按 `offset`（默认 2，可负）读取 |

`from: "json"` 的补充：

- **JSONPath 子集**：只支持 `$` 根、`.key` 成员访问、`[n]` 非负数组下标。不支持通配、递归、过滤器、切片。例：`$.responseData.data.smsId`、`$.result.items[0].token`。
- **`asJson: true`**（双层 JSON 场景）：先按 `path` 取值（必须是字符串），把它当 JSON 再解析一次，然后按 `innerPath`（可省）取值。
- 路径不存在 → `E_EXTRACT`；声明 `optional: true` 时改用 `default`。

`from: "binary"` 的 `encoding`：`ascii` / `utf8` / `hex` / `be32`（读 4 字节大端无符号转十进制，此时 `length` 必须为 4）/ `raw`（返回 bytes）。

### 4.7 `assert`：断言

挂在 `request` 步上，校验响应体，不通过则本步失败（`E_ASSERT`）：

```json
"assert": {
  "json": "$.responseData.resultCode",
  "equals": "0000",
  "message": "登录失败"
}
```

共 **13 个操作符，恰好用一个**（用 0 个或 ≥2 个都会报 `E_SCHEMA`）：

| 操作符 | 取值 | 语义 |
|---|---|---|
| `equals` / `notEquals` | 模板 | 精确字符串比较（比较前两侧都转字符串） |
| `contains` | 模板 | 包含子串 |
| `matches` | 模板 | 正则匹配 |
| `isInteger` | `true`/`false` | 值是否为整数 |
| `isNumeric` | `true`/`false` | 值是否为数字（含小数） |
| `isEmpty` / `isNotEmpty` | `true`/`false` | 是否为空（空串/空列表） |
| `exists` | `true`/`false` | 提取到的值是否存在（非 null） |
| `greaterThan` / `lessThan` | 数值（或模板字符串，如 `"{{vars.limit}}"`） | 数值比较 |
| `startsWith` / `endsWith` | 模板 | 前缀 / 后缀 |

## 5. 表达式系统与 fn 函数库

### 5.1 模板语法

所有字符串字段里的 `{{ ... }}` 都会被求值替换。花括号内是一个表达式：

```
"{{vars.base}}/auth/login"
"{{fn.ts14()}}"
"{{input.phone}}"
```

渲染规则：

| 值类型 | 渲染结果 |
|---|---|
| string | 原样 |
| int / double | 十进制（`3.0` 渲染为 `"3.0"`，保留 `.0`） |
| bool | `"true"` / `"false"` |
| null | 空串（不是 `"null"`） |
| list / map | 紧凑 JSON 序列化 |
| **bytes** | ❌ 不可直接渲染 → `E_TYPE`，必须先经 `fn.b64` / `fn.hex` / `fn.latin1` |

- 需要字面量 `{{` 时写作 `\{\{`。
- 模板语法错误（如 `{{` 后没有 `}}`）→ `E_TEMPLATE`。

### 5.2 表达式语法

用于 `{{ }}` 内部、`when`、`until`、`branch[].when`：

```
{{steps.init.code == '1016' && input.phone != ''}}
{{vars.token != '' && vars.token != null}}
```

- 支持：`||`、`&&`（短路）、`==`、`!=`、`>=`、`<=`、`>`、`<`、`!`、括号、数字、单引号字符串（不支持转义）、`true`/`false`/`null`、路径引用、`fn.*` 调用。
- 比较规则：两侧同为数字 → 数值比较；两侧同为 bytes → 逐字节比较（bytes 参与比较**不需要**先转 hex）；bytes 与 string 混合 → `==` 恒为 `false`；其余转字符串比较。
- `when` / `until` / `branch[].when` 的顶层表达式必须求值为 bool。

### 5.3 变量命名空间（唯一合法来源）

| 命名空间 | 内容 | 只读 |
|---|---|---|
| `input.<key>` | 用户表单值 + `ui` 步写入的值 | 是 |
| `vars.<key>` | 顶层 `vars` + `set` 步写入 | 否（`set` 可写） |
| `steps.<stepId>.<name>` | 某步骤 `extract` 出的值 | 是 |
| `env.<key>` | 运行环境（见下） | 是 |
| `secrets.<key>` | 安全存储读取 | 是 |
| `response.message` | 仅 `onFail` 内可用 | 是 |

`env` 命名空间：

| key | 类型 | 说明 |
|---|---|---|
| `env.deviceModels` | string[] | 内置机型列表（用于随机 UA 之类） |
| `env.appVersion` | string | App 版本 |
| `env.platform` | string | `android` / `ios` |
| `env.locale` | string | 如 `zh-CN` |

引用不存在的路径 → `E_UNDEF`，**不返回 null**。

### 5.4 fn 函数库（73 个，全表）

`fn.*` 在模板和表达式里直接调用：`{{fn.md5(input.phone)}}`。

**同一输入必须产生同一输出**（确定性要求），豁免清单：`fn.now` / `fn.ts10` / `fn.ts13` / `fn.ts14` / `fn.daysBeforeNow` / `fn.daysAfterNow`（依赖当前时间）以及随机函数（`fn.randomHex` / `fn.randomDigits` / `fn.randomOf` / `fn.randomInt` / `fn.randomModOffset` / `fn.uuid`）。引擎为随机与时间注入受控种子/时钟，用于测试复现。

> 以下行号均指 `engine/lib/src/js/builtin.js`（v2.0：函数库由内置 JS 沙箱提供，`{{fn.*}}` 语法与旧版完全一致，现有插件无需改写）。

**编码 / 转换（13 个）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.b64(x)` | L1450 | string/bytes → base64 字符串 |
| `fn.b64d(s)` | L1455 | base64 → 解码为 UTF-8 **字符串** |
| `fn.unb64(s)` | L1459 | base64 → 解码为 **bytes** |
| `fn.b64url(x)` | L1463 | URL-safe base64（无 `=` 填充，`+`→`-`，`/`→`_`） |
| `fn.unb64url(s)` | L1470 | URL-safe base64 → bytes |
| `fn.hex(x)` | L1477 | string/bytes → 小写 hex 字符串 |
| `fn.unhex(s)` | L1482 | hex → bytes |
| `fn.utf8(s)` | L1486 | string → UTF-8 bytes |
| `fn.latin1(b)` | L1490 | bytes → latin1 字符串 |
| `fn.urlenc(s)` | L1496 | RFC 3986 urlencode |
| `fn.gz(b)` | L1500 | gzip 压缩（mtime=0）。⚠️ 输出**不保证跨引擎逐字节相同**，别把它的字节当校验值用 |
| `fn.gunz(b)` | L1504 | 解压任意标准 gzip 流 |
| `fn.jsonParse(s)` | L1508 | JSON 字符串 → 值（失败报 `E_ARG`） |

**哈希（5 个）**：输出均为小写 hex

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.md5(s)` | L1518 | MD5 |
| `fn.sha1(s)` | L1519 | SHA-1 |
| `fn.sha256(s)` | L1520 | SHA-256 |
| `fn.hmacSha256(k, s)` | L1521 | HMAC-SHA256，k 为密钥 |
| `fn.crc32(x)` | L1524 | CRC-32/ISO-HDLC（同 zlib），输出 8 位小写 hex |

**加密（3 个）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.rsaPkcs1(pubB64, plaintext, bt?)` | L1537 | RSA PKCS#1 v1.5 加密，公钥为 base64，输出标准 base64 且**定长=模长**。第 3 参块类型：`'02'`（默认）或 `'01'` |
| `fn.aesCbc(keyHex, ivHex, dataB64)` | L1557 | AES-CBC 加密，key/iv 为 hex、数据为 base64，输出 base64 |
| `fn.aesEcb(keyHex, dataB64)` | L1563 | AES-ECB 加密，同上参数约定 |

**字符串处理（11 个）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.shift2(s)` | L1570 | 每字符码点 +2 |
| `fn.shiftN(s, n)` | L1578 | 每字符码点 +n |
| `fn.padRight(s, n, c)` / `fn.padLeft(s, n, c)` | L1587 / L1588 | 用 `c` 右/左填充到长度 `n` |
| `fn.repeat(c, n)` | L1589 | 首字符重复 `n` 次 |
| `fn.substring(s, a, b?)` | L1601 | 子串，越界自动截断、无负索引 |
| `fn.concat(...)` | L1616 | 拼接；首参为 string → 字符串拼接，为 bytes → 字节拼接 |
| `fn.len(x)` | L1649 | 长度（string/bytes/list/map） |
| `fn.trim(s)` | L1656 | 去首尾空白 |
| `fn.upper(s)` / `fn.lower(s)` | L1657 / L1658 | 大写 / 小写 |

**bytes 操作（10 个）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.bytesReplace(b, from, to)` | L1661 | 字节级查找替换 |
| `fn.be32Write(b, offset, value)` | L1679 | 大端写 4 字节，返回新 bytes |
| `fn.be32Read(b, offset)` | L1692 | 大端读 4 字节 → int |
| `fn.be16Write(b, offset, value)` | L1698 | 大端写 2 字节 |
| `fn.be16Read(b, offset)` | L1709 | 大端读 2 字节 → int |
| `fn.u8Write(b, offset, value)` | L1715 | 写 1 字节 |
| `fn.slice(x, a, b?)` | L1725 | 切片，**支持负索引**（`-1` = 末尾；`fn.slice(x, 0, -1)` 去掉最后 1 字节） |
| `fn.indexOf(h, n, start?)` | L1741 | 查找子串/子字节串位置，未找到返回 -1 |
| `fn.at(b, i)` | L1759 | 取第 `i` 个字节 → int（越界报 `E_ARG`） |
| `fn.scanAscii(b, tagHex, length)` | L1765 | 定位 tag（hex）后跳过整个 tag 读 `length` 字节为 ASCII 字符串 |

**随机 / 时间（10 个）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.randomHex(n)` | L1784 | `n` 位随机 hex 字符串 |
| `fn.randomDigits(n)` | L1785 | `n` 位随机数字串 |
| `fn.randomOf(list)` | L1786 | 从列表随机取一项（空列表报 `E_ARG`） |
| `fn.randomInt(min, max)` | L1791 | 闭区间随机整数 |
| `fn.randomModOffset(mod, off)` | L1797 | `off + random(mod)`；参数支持十进制或 `0x` 十六进制，内部用大整数 |
| `fn.uuid()` | L1812 | UUID v4 |
| `fn.now()` | L1821 | 当前毫秒时间戳（number） |
| `fn.ts14()` | L1822 | UTC 时间 `yyyyMMddHHmmss`（14 位） |
| `fn.ts10()` | L1829 | 秒级时间戳字符串 |
| `fn.ts13()` | L1830 | 毫秒时间戳字符串 |

**空值处理（2 个）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.coalesce(...)` | L1833 | 返回第一个非 null、非空串的参数 |
| `fn.default(x, y)` | L1842 | `x` 为 null/空串时返回 `y` |

**字符串增强（5 个，v1.4）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.replace(s, old, rep)` | L1848 | 全量替换 |
| `fn.replaceRegex(s, pat, rep)` | L1855 | 正则替换（`g` 标志），`rep` 支持 `$1` 反向引用 |
| `fn.split(s, sep)` | L1877 | 分割；`sep` 为空串时按单字符拆 |
| `fn.startsWith(s, p)` / `fn.endsWith(s, p)` | L1887 / L1890 | 前缀 / 后缀判断 |

**类型转换（3 个，v1.4）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.toInt(x)` | L1895 | 转 int；字符串支持十进制与 `0x` 十六进制 |
| `fn.toFloat(x)` | L1901 | 转 double |
| `fn.toString(x)` | L1910 | 转字符串；null→空串，bytes→UTF-8 严格解码（失败回退 hex），list/map→JSON |

**字符集 / HTML（3 个，v1.4）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.charsetDecode(b, name)` | L1925 | 解码：`utf-8` / `latin-1` / `ascii` / `gbk`（`gb2312`、`gb18030` 同表）；`big5` 不支持 |
| `fn.htmlUnescape(s)` | L1949 | HTML 实体反转义 |
| `fn.htmlEscape(s)` | L1950 | HTML 实体转义 |

**集合（4 个，v1.4）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.count(x)` | L1953 | 元素个数（同 `fn.len` 的输入范围） |
| `fn.first(x)` / `fn.last(x)` | L1960 / L1971 | 首个 / 末个元素（空集合报 `E_ARG`） |
| `fn.nth(x, i)` | L1982 | 第 `i` 个元素，支持负索引 |

**日期（4 个，v1.5，时区统一 UTC）**

| 函数 | 行号 | 语义 |
|---|---|---|
| `fn.toDate(x, fmt?)` | L2000 | 解析日期 → ISO UTC 字符串 |
| `fn.dateFormat(x, fmt)` | L2004 | 按格式化占位符输出 |
| `fn.daysBeforeNow(x)` | L2010 | 距今天几天（过去为正数） |
| `fn.daysAfterNow(x)` | L2017 | 距今天几天（未来为正数） |

### 5.5 高频组合示例

```json
"vars": {
  "phoneE164": "{{fn.concat('+86', input.phone)}}",
  "sign": "{{fn.md5(fn.concat(vars.plain, vars.salt))}}",
  "deviceModel": "{{fn.randomOf(env.deviceModels)}}",
  "ua": "{{fn.concat('okhttp/4.11.0 ', vars.deviceModel)}}",
  "authHeader": "{{fn.concat('Bearer ', steps.login.token)}}"
}
```

## 6. JS 沙箱（set.js 逃生舱）

当模板表达式确实不够用（复杂控制流、自定义编码算法、大整数运算、任意字节布局……），可以在 `set` 步里嵌入一段纯计算 JS：

```json
{
  "id": "enc",
  "type": "set",
  "set": {
    "vars": { "plainHex": "{{vars.plainHex}}" },
    "js": {
      "code": "return fn.hex(new Uint8Array([0x01, 0x02]));",
      "inputs": { "a": "{{vars.someInput}}" },
      "output": "resultHex"
    }
  }
}
```

| 字段 | 说明 |
|---|---|
| `code` | JS 代码。`inputs` 里的每个键是**同名 JS 变量**；必须 `return` 一个值 |
| `inputs` | 输入映射，值经模板渲染后传入 |
| `output` | 返回值写入 `vars` 的键名 |

沙箱规则（强制，不可绕过）：

| 项 | 规定 |
|---|---|
| `fetch` / `XMLHttpRequest` | ❌ 调用即报 `E_JS`。`set.js` 是纯计算上下文，网络请求只能走 flow 的 `request` 步骤 |
| `eval` / `new Function`（含 `constructor` 间接路径） | ❌ 禁用 |
| `import` / `require` | ❌ 禁用 |
| `setTimeout` / `setInterval` / `clearTimeout` / `clearInterval` | ❌ 禁用 |
| `Math.random` / `Date.now` | 替换为引擎注入的确定性 PRNG / 时钟 |
| `secrets.*` / `env.*` | 以**只读冻结对象**注入 JS；日志输出强制脱敏 |
| 超时 | 默认 5000ms，超时 → `E_JS` |
| 内存 | 32MB 上限，超限 → `E_JS` |
| 整数 | 超过 2^53 → `E_JS` |

`hooks` 字段配置沙箱：

```json
"hooks": {
  "libs": ["forge@1.4.0"],
  "timeoutMs": 5000
}
```

- `libs`：内置库白名单，目前提供 `forge@1.4.0`（RSA / AES / HMAC / 哈希，BSD-3-Clause）。**不支持运行时从网络安装依赖**——供应链风险、离线失效、版本漂移都不可接受。
- `timeoutMs`：单次 JS 执行超时，默认 5000，范围 100–60000。

类型映射（Dart ⇄ JS）：`String↔string`；`int/double↔number`；`bool↔boolean`；`Uint8List↔Uint8Array`；`List↔Array`；`Map↔Object`（键必须是 string）。

## 7. inputs：声明插件需要的信息

`inputs` 数组自动生成运行前的表单：

```json
"inputs": [
  {
    "key": "phone",
    "label": "手机号",
    "type": "phone",
    "required": true,
    "maxLength": 11,
    "placeholder": "11 位手机号",
    "sensitive": false
  },
  {
    "key": "mode",
    "label": "操作类型",
    "type": "select",
    "options": [
      { "value": "1", "label": "查询余额" },
      { "value": "2", "label": "查询积分" }
    ]
  }
]
```

| 字段 | 约束 |
|---|---|
| `key` | `^[a-zA-Z_][a-zA-Z0-9_]*$`，manifest 内唯一，流程里用 `{{input.<key>}}` |
| `type` | `phone` / `text` / `password` / `number` / `select` |
| `required` | 默认 `true` |
| `default` | 可选，仅 string/number |
| `pattern` | 可选，ECMA 正则（不加定界符）；`type: phone` 未显式给 pattern 时隐含 `^1\d{10}$` |
| `maxLength` / `minLength` | 可选 |
| `options` | `type=select` 时必需，其余类型禁止出现。`{value, label}` 静态字面量，不支持模板 |
| `sensitive` | 默认 `false`；`true` 表示该输入不写日志、不落盘 |

校验失败 → 拒绝提交表单，向用户展示 `label` + 原因。

## 8. output：声明运行结果

```json
"output": {
  "title": "查询结果",
  "fields": [
    {
      "label": "Token",
      "value": "{{steps.login.token}}",
      "copyable": true,
      "sensitive": false,
      "multiline": false
    }
  ]
}
```

| 字段 | 约束 |
|---|---|
| `title` | 必需 |
| `fields` | 必需，1–32 项 |
| `label` / `value` | 必需；`value` 是模板 |
| `copyable` | 默认 `true`，是否允许复制 |
| `sensitive` | 默认 `false`；`true` 则界面默认打码显示，用户点击才明文 |
| `multiline` | 默认 `false`，长内容建议开启 |

⚠️ **output 只应引用 `vars.*` / `input.*`**。直接引用 `steps.*` 有陷阱：引用从未执行过的步骤（因 `end: true` 提前结束、或 `when` 为假跳过）会报 `E_UNDEF`，把已成功的流程判为失败。要展示步骤产出，先用 `set` 步搬进 `vars`（并在顶层 `vars` 给好默认值）。

## 9. 安全约束（必读）

安全模型的核心是 **`allowHosts` 白名单**——插件能访问的网络范围在加载时就固定下来，用户导入插件时可以在确认页看到全部域名。

**规则（全部强制，不可绕过）：**

1. `allowHosts` 每项格式：`host` 或 `host:port`，**不含 scheme**、不允许路径；小写**精确匹配**；**不做子域通配**（明确不支持 `*.example.com` 写法）。
2. 每个 `request.url` 的 host（+端口）必须命中白名单，否则**在发出请求前**报 `E_HOST`——不会有任何字节发出去。
3. 重定向的**每一跳都重新校验**白名单，跨 host 且不在白名单 → `E_HOST`；最多 5 跳。
4. 默认**禁止明文 `http://`** → `E_INSECURE`。唯一例外：`allowInsecureLocalhost: true` 且目标是本机回环地址（`localhost` 等）——**仅供本地调试，正式插件不得使用**。
5. **禁止关闭 TLS 证书校验**。引擎不提供任何关闭校验的逃生口。
6. `webview` 步骤中 HTML 内引用的外部子资源（script/css/XHR）的域名**不在引擎强制拦截范围内**——这是插件作者的责任：请在 `allowHosts` 里声明所有会被访问的 host，导入确认页才能如实展示。

**写插件时的自检清单：**

- [ ] `allowHosts` 只包含真正需要的域名，一个不多
- [ ] 所有 URL 都是 `https://`
- [ ] 用户密码、验证码等输入字段的 `sensitive: true`（或用 `password` 类型）
- [ ] 固定密钥走 `secrets` 声明，不硬编码在 manifest 里
- [ ] 没有把 `secrets.*` 的值拼接进 output 或日志字段

## 10. 调试技巧

1. **先用「粘贴导入」快速试错**。改 JSON → 粘贴 → 看校验错误，比反复打包快得多。导入时的错误信息会精确到字段。
2. **把中间值放进 output**。调不通时，临时把可疑的中间变量加进 `output.fields`（`multiline: true`），一次运行看清整条链路。
3. **长表达式拆管道**。单行模板超过百来字符就该拆成多个 `set` 变量，后者引用前者；每行只做一件事。bytes 链记得用 `fn.hex` / `fn.unhex` 中转（见 4.4）。
4. **用 `assert` 早早失败**。在每个关键请求后断言响应结构（如 `$.code equals 0`），配上有意义的 `message`。失败得越早，定位越容易。
5. **`onFail` 里用 `{{response.message}}`**。让服务端的错误文案直接展示给用户，省得对着超时猜测。
6. **注意确定性豁免**。随机/时间函数每次运行都不同；调试签名问题时把随机值临时固定成常量，跑通后再换回去。真实插件里请把随机值隔离在少数几个 `vars` 里，出错时好定位。
7. **`fn.gz` 的字节不要当校验值**。不同引擎的 gzip 字节可能不同（这是明确过的固有差异，不是 bug）；它只用于「压缩后发送、对方解压」的场景。
8. **看日志页**。App 会记录每步的执行轨迹；敏感值自动以占位符脱敏显示。

## 11. 完整示例

一个展示主要能力的演示插件：表单收集手机号 → 构造签名 → POST 登录 → 断言 → 轮询结果 → 展示。全部请求指向演示域名 `api.example.com`。

```json
{
  "id": "demo.showcase",
  "name": "能力演示插件",
  "version": "1.0.0",
  "engine": ">=1.0 <2.0",
  "author": "demo",
  "description": "演示 inputs/vars/request/assert/poll/output 的写法，域名均为演示用途",
  "allowHosts": ["api.example.com"],
  "inputs": [
    {
      "key": "phone",
      "label": "手机号",
      "type": "phone",
      "required": true,
      "maxLength": 11,
      "placeholder": "11 位手机号"
    },
    {
      "key": "smsCode",
      "label": "短信验证码",
      "type": "text",
      "required": true,
      "minLength": 6,
      "maxLength": 6,
      "pattern": "^\\d{6}$",
      "placeholder": "6 位数字"
    }
  ],
  "vars": {
    "base": "https://api.example.com",
    "phoneE164": "{{fn.concat('+86', input.phone)}}",
    "ts": "{{fn.ts14()}}",
    "deviceModel": "{{fn.randomOf(env.deviceModels)}}",
    "nonce": "{{fn.randomHex(16)}}",
    "sign": "{{fn.md5(fn.concat(input.phone, vars.ts, vars.nonce))}}"
  },
  "flow": [
    {
      "id": "login",
      "type": "request",
      "request": {
        "method": "POST",
        "url": "{{vars.base}}/v1/auth/login",
        "headers": {
          "Content-Type": "application/json;charset=UTF-8",
          "User-Agent": "DemoApp/{{env.appVersion}}",
          "X-Nonce": "{{vars.nonce}}",
          "X-Sign": "{{vars.sign}}"
        },
        "body": {
          "type": "json",
          "value": {
            "phone": "{{vars.phoneE164}}",
            "smsCode": "{{input.smsCode}}",
            "deviceModel": "{{vars.deviceModel}}",
            "ts": "{{vars.ts}}"
          }
        },
        "timeoutMs": 20000,
        "retry": { "times": 1, "on": [502, 503, 504] }
      },
      "extract": [
        { "name": "code", "from": "json", "path": "$.code" },
        { "name": "token", "from": "json", "path": "$.data.token" },
        { "name": "ticketId", "from": "json", "path": "$.data.ticketId", "optional": true, "default": "" }
      ],
      "assert": {
        "json": "$.code",
        "equals": "0",
        "message": "登录接口返回非 0"
      },
      "onFail": { "message": "登录失败：{{response.message}}" }
    },
    {
      "id": "saveToken",
      "type": "set",
      "set": {
        "vars": {
          "authHeader": "{{fn.concat('Bearer ', steps.login.token)}}",
          "resultReady": "false"
        }
      }
    },
    {
      "id": "queryStatus",
      "type": "request",
      "request": {
        "method": "GET",
        "url": "{{vars.base}}/v1/ticket/status",
        "query": { "ticketId": "{{steps.login.ticketId}}" },
        "headers": { "Authorization": "{{vars.authHeader}}" }
      },
      "extract": [
        { "name": "status", "from": "json", "path": "$.data.status" },
        { "name": "balance", "from": "json", "path": "$.data.balance", "optional": true, "default": "" }
      ]
    },
    {
      "id": "waitResult",
      "type": "poll",
      "poll": {
        "steps": ["queryStatus"],
        "until": "steps.queryStatus.status == 'READY'",
        "intervalMs": 2000,
        "maxAttempts": 10,
        "onTimeout": { "message": "结果尚未就绪，请稍后重试" }
      }
    },
    {
      "id": "collect",
      "type": "set",
      "end": true,
      "set": {
        "vars": {
          "finalBalance": "{{fn.default(steps.queryStatus.balance, '未知')}}"
        }
      }
    }
  ],
  "output": {
    "title": "演示运行结果",
    "fields": [
      { "label": "Token", "value": "{{steps.login.token}}", "sensitive": true, "copyable": true },
      { "label": "签名（调试用）", "value": "{{vars.sign}}", "multiline": false },
      { "label": "余额", "value": "{{vars.finalBalance}}" },
      { "label": "设备机型", "value": "{{vars.deviceModel}}" },
      { "label": "随机数", "value": "{{vars.nonce}}" }
    ]
  }
}
```

要点回顾：

- `poll.steps` 引用的 `queryStatus` 是 flow 里真实存在的请求步；轮询直到 `status == 'READY'`。
- `collect` 步把步骤产出搬进 `vars`（`finalBalance`），output 只引用 `vars` / `steps` 中**必然执行过**的值——`login` 一定会跑（否则流程早已失败），所以 `steps.login.token` 是安全的。
- Token 标了 `sensitive: true`：界面默认打码，用户点击才明文。
- 签名用 `fn.md5(concat(...))` 组合时间戳与随机数——仅作演示，真实协议按目标站点要求实现。

## 12. 错误码速查表

| 码 | 触发场景 | 排查方向 |
|---|---|---|
| `E_SCHEMA` | manifest 不符合 JSON Schema（缺字段、未知字段、格式错误、assert 操作符不是恰好一个……） | 看错误信息里的字段路径；逐个核对本指南的约束 |
| `E_ENGINE` | `engine` 版本范围与 App 内置引擎不匹配 | 调整 `engine` 范围 |
| `E_TEMPLATE` | 模板语法错误（未闭合、非法表达式） | 定位到报错字符串，检查 `{{`/`}}` 配对 |
| `E_UNDEF` | 引用不存在的路径（变量、步骤产出、input） | 检查拼写；确认引用的步骤确实会执行；output 别引用可能被跳过的步骤 |
| `E_TYPE` | 函数参数类型错误；**bytes 直接渲染**进 vars/output | bytes 先经 `fn.b64`/`fn.hex`/`fn.latin1` 转字符串 |
| `E_ARG` | 参数取值非法（空列表、越界、非法正则、非法规章……） | 看错误消息里的函数名与参数 |
| `E_GOTO` | `goto`/`branch` 指向不存在的 step id | 加载期就会报，核对 id 拼写 |
| `E_HOST` | URL host 不在 `allowHosts`（含重定向跳转） | 把目标 host 加进 `allowHosts`；检查是否被 30x 跳到了别的域名 |
| `E_INSECURE` | 明文 `http://` 且未授权 | 改用 `https://`；本地调试才用 `allowInsecureLocalhost` |
| `E_HTTP` | HTTP 状态非 2xx 且不在 `retry.on` | 先看状态码与响应体；必要时调 `retry` |
| `E_TIMEOUT` | 请求超时 | 网络/目标站点慢；调大 `timeoutMs`（上限 120000） |
| `E_EXTRACT` | 提取失败且非 `optional` | 打印原始响应对照 JSONPath/正则/tag；不稳定的路径加 `optional: true` + `default` |
| `E_ASSERT` | 断言不通过 | 看 assert 的 `message` 与实际响应 |
| `E_LOOP` | 检测到死循环（单步执行次数 > flow.length × 3） | 检查 `goto`/`branch` 是否构成环；`poll` 的 `maxAttempts` 是否过小 |
| `E_END` | `end: "error"` 显式失败终止（output 仍会渲染） | 属于插件主动声明的失败，看 output 里的诊断字段 |
| `E_JS` | JS 沙箱错误：语法错 / 运行时异常 / 超时 / 内存超限 / 调用禁用 API / 整数超 2^53 | 在 `code` 里分步 return 调试；检查是否用了禁用的 API |

导入环节另有三个包级错误（不属流程错误）：

| 码 | 触发场景 |
|---|---|
| `E_PKG_FORMAT` | 内容不是合法 JSON、顶层不是 object，或既不是 `.qbpkg` 信封也不是裸 manifest |
| `E_PKG_MANIFEST` | 包内找不到 `manifest.json`（或信封里没有 manifest 字段） |
| `E_SCHEMA` | 找到 manifest 但校验失败（同上表第一行） |

---

**下一步**：写好插件后，请先阅读《[用户指南](USER_GUIDE.md)》了解导入与运行流程；导入格式细节、抓包录制器等更多内容见 [`docs/PLUGIN_FORMAT.md`](PLUGIN_FORMAT.md)。

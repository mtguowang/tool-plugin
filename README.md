# 工具插件（tool-plugin）

> 一款插件化的 Android 工具箱：每个「工具」是一份 **JSON 插件**，声明要收集什么输入、访问哪些网站、展示什么结果——App 负责安全地执行它。

## 这是什么

很多小工具的共性是：填个手机号/账号 → 按网站的要求发几个请求 → 把结果展示出来。这类流程一旦写死在 App 里，每加一种就要发一次版。

本项目把它反过来：**流程做成数据，而不是代码**。一个插件就是一份 `manifest.json`——输入表单、HTTP 流程、交互步骤、结果字段全部声明式描述，App 内置引擎解释执行。

- **新增一种工具** = 导入一份 JSON，无需重新编译、无需升级 App
- **插件能访问的网络范围**由它自己声明的域名白名单（`allowHosts`）写死，用户导入前看得一清二楚，引擎在请求发出前强制校验
- 插件本体**不含可执行代码**（唯一例外是可选的 `set.js` 纯计算逃生舱，跑在禁网禁文件的 QuickJS 沙箱里）

## 下载

**[⬇ 下载最新版 APK](https://github.com/mtguowang/tool-plugin/releases/latest)**

| 项 | 说明 |
|---|---|
| 版本 | v1.0 |
| 大小 | 22.5 MB |
| 架构 | **arm64-v8a**（2017 年后的手机基本都支持）|
| 系统要求 | Android 7.0（API 24）及以上 |
| 权限 | 仅网络（无存储、无定位、无通讯录）|

> 安装提示：华为/荣耀等机型可能被「纯净模式」拦截，需在设置里临时允许安装未知来源应用。

## 更新日志

### v1.0.1（2026-10-26）

- **插件可控制输入框明文/密文**：`ui` 步骤新增 `obscure` 字段（省略 = 宿主自动判定，
  `true` = 强制掩码，`false` = 强制明文）。只影响界面显示，日志仍按 `sensitive` 脱敏
- **手机号输入不再掩码**：用户需要核对自己输入的内容（输错一位就收不到短信）
- **中通插件失败原因可见**：不再是笼统的「注册失败」，而是显示服务端真实原因
  （如「该手机号已注册（statusCode=USER_ALREADY_EXISTS）」）
- 修复若干测试与插件版本失配问题

### v1.0（2026-10-26）

首个公开版本。包含以下改进：

- **列表到底提示按需显示**：「已经到底啦」只在内容超出一屏、需要滚动时出现；插件很少时不再显示多余提示
- **导出保存到下载目录**：插件导出从「复制到剪贴板」改为保存为文件，路径 `Download/tool-plugin/`，保存后显示完整路径。Android 10+ 无需任何权限
- **输入框键盘修复**：插件库地址框不再触发手机厂商的安全键盘（改用普通键盘，可正常输入中文）
- **长按菜单中文化**：输入框长按菜单显示「剪切 / 复制 / 粘贴 / 全选」（原为英文）
- **首次启动空态修复**：全新安装不再显示「插件目录不存在」错误，正常显示引导空态
## 特性

- **声明式插件**：`inputs`（表单）→ `flow`（request / poll / ui / set 四种步骤）→ `output`（结果字段），支持条件跳转、轮询、验证码/滑块交互、断言与重试
- **域名白名单安全边界**：导入前展示全部域名；请求发出前校验（越权请求零字节外发）；重定向逐跳复检；禁明文 HTTP；TLS 校验不可关闭
- **内置 JS 沙箱与 73 个函数**：`fn.md5` / `fn.rsaPkcs1` / `fn.aesCbc` / `fn.gz` / GBK 解码等开箱即用；不够用时 `set.js` 逃生舱可直接写 JS，跑在带超时与内存上限的 QuickJS 沙箱中，附 `forge` 加密库
- **插件库分发**：任意 GitHub 仓库都能当插件源（`index.json` + `.qbpkg` 包），下载强制 SHA-256 校验，内置多级镜像加速（国内可直连）
- **隐私优先**：仅申请网络权限；零埋点、零统计 SDK；日志中的敏感值自动脱敏；调试日志可整体关闭

## 快速开始

1. 从 [Releases](https://github.com/mtguowang/tool-plugin/releases/latest) 下载最新 APK 并安装
2. 打开 App，点右上角 **云图标**（插件库）
3. 在「插件库设置」里填入插件仓库地址（你自己的库，或别人分享的库）
4. 勾选需要的插件，点「导入选中」
5. 回到列表，点插件卡片上的「开始」，填表单、按提示交互，拿到结果

> 不知道填什么插件库地址？插件库是**你自己或别人维护的一个 GitHub 仓库**，
> 里面是 `index.json` + 若干 `.qbpkg` 文件。App 不内置任何插件。

## 校验插件

写完插件先本地校验，别急着装 App 试：

```bash
# 下载校验器（两个文件放同一目录）
curl -O https://raw.githubusercontent.com/mtguowang/tool-plugin/main/schema/validate.py
curl -O https://raw.githubusercontent.com/mtguowang/tool-plugin/main/schema/manifest.schema.json

# 校验
python validate.py <你的插件目录>/manifest.json
```

零依赖，只需 Python 3.8+。

**它检查什么**：字段名拼写、必填字段、类型/枚举/正则、`goto`/`branch`/`poll.steps` 的目标 id 是否存在、`vars`/`inputs`/`secrets` 引用是否有来源、`allowHosts` 格式与 URL 匹配。

**它不检查什么**：加密算法算得对不对、服务端认不认这些参数、`set.js` 能不能跑通 —— 那些只能真机验证。
## 自己写插件

插件就是一份 JSON 文件。完整格式说明见：

**📖 [插件编写指南（manifest.json）](docs/PLUGIN_JSON_GUIDE.md)**

最小可运行插件长这样：

```json
{
  "id": "demo.hello",
  "name": "示例：打个招呼",
  "version": "1.0.0",
  "engine": ">=1.0 <2.0",
  "allowHosts": ["demo.invalid"],
  "inputs": [
    { "key": "name", "label": "你的名字", "type": "text", "required": true }
  ],
  "flow": [
    { "id": "greet", "set": { "message": "你好，{{input.name}}" } }
  ],
  "output": {
    "fields": [
      { "label": "问候语", "value": "{{vars.message}}" }
    ]
  }
}
```

把这段存成 `manifest.json`，在 App 里点右上角 **加号** → 粘贴 JSON → 导入即可。

## 项目结构

本仓库是**发布仓库**，只包含：

```
tool-plugin/
├── README.md                        ← 本文件
├── LICENSE                          ← MIT
├── docs/
│   └── PLUGIN_JSON_GUIDE.md         ← 插件开发完整文档
└── schema/
    ├── manifest.schema.json         ← manifest 结构契约
    └── validate.py                  ← 零依赖离线校验器
```

**APK 不放在仓库里**，从 [Releases](https://github.com/mtguowang/tool-plugin/releases/latest) 下载（避免仓库体积膨胀）。

App 源码暂未开源。

## 免责声明

> **请务必阅读。**

- 本项目**仅供个人学习与技术研究使用**。
- 插件是第三方编写的数据文件，本项目只提供执行框架。**安装什么插件、用插件访问什么网站，是使用者自己的决定**，作者不对任何第三方插件的行为及其后果负责。
- 使用本工具访问任何网站时，你必须遵守目标网站的服务条款以及当地法律法规。**不得**将本工具用于破解、绕过验证机制、批量爬取数据、侵犯他人权益或任何违法用途。
- 导入插件前请看清它声明的域名白名单；来路不明的插件可能把你输入的内容发到它的白名单域名。
- 如目标平台明确禁止此类自动化操作，请勿使用本工具。

## 许可证

[MIT](LICENSE)

## 联系

- GitHub：[@mtguowang](https://github.com/mtguowang)
- 问题反馈：[Issues](https://github.com/mtguowang/tool-plugin/issues)

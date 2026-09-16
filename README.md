<div align="center">
  <img src="assets/cover.svg" alt="SecTest Atlas — 网络与服务器安全测试完全手册" width="100%">
</div>

# SecTest Atlas

An execution-focused handbook for **authorized security assessments**. SecTest Atlas turns real attack paths into searchable, checkable, and actionable test cases across networks, servers, wireless, applications, cloud-native platforms, and AI systems.

[Live handbook](https://cndoin.github.io/sectest-atlas/) · [Chinese Markdown edition](./docs/handbook.md) · [Install guide](./INSTALL.md) · [AI setup guide](./AI_INSTALL.md) · [Security policy](./SECURITY.md)

> [!WARNING]
> For authorized assessment, education, and defensive validation only. Obtain written permission and define scope, test windows, data handling, rollback, and emergency contacts before running any test.

## Why this project

- **31 chapters and 700+ checks** covering the full assessment lifecycle.
- **No build step:** one HTML file works locally and on GitHub Pages.
- **Operator-friendly UI:** navigation search, persistent checkboxes, dark mode, mobile drawer, and print/PDF export.
- **Evidence-oriented:** findings are structured around observation, proof, impact, remediation, and retest.
- **Bilingual onboarding:** English-first project documentation with the complete Chinese field handbook included.

![Interactive handbook preview](assets/interface-preview.svg)

## Quick start

```bash
git clone https://github.com/cndoin/sectest-atlas.git
cd sectest-atlas
python -m http.server 8080
```

Open `http://localhost:8080`. There are no package dependencies and no build command. See [INSTALL.md](./INSTALL.md) for Windows/macOS/Linux instructions, or give [AI_INSTALL.md](./AI_INSTALL.md) to your coding agent.

## Coverage

| Area | Included topics |
| --- | --- |
| Frameworks | PTES, NIST SP 800-115, OSSTMM, MITRE ATT&CK, CIS Benchmarks |
| Network & host | L2–L4, DNS, segmentation, Linux/Windows, identity, credentials |
| Application & platform | Web, API, microservices, cloud, containers, Kubernetes, CI/CD |
| Wireless & emerging tech | WPA2/WPA3, Wi-Fi 6E/7, BLE, ZigBee, AI/LLM/agents |
| Resilience | Detection, purple teaming, recovery, ransomware, performance, chaos testing |
| Delivery | Risk scoring, reports, authorization, ROE, evidence, acceptance criteria |

## Repository layout

```text
.
├── index.html              # Interactive handbook and GitHub Pages entry
├── assets/                 # Brand and interface visuals
├── docs/handbook.md        # Complete Chinese Markdown handbook
├── INSTALL.md              # Local installation and serving guide
├── AI_INSTALL.md           # Ready-to-use instructions for coding agents
├── CONTRIBUTING.md         # Contribution rules
└── SECURITY.md             # Disclosure policy and legal boundary
```

---

## 中文说明

一份面向**合法授权场景**的中文网络与服务器安全测试手册。以攻击者真实路径为线索，把网络、服务器、无线、应用、云原生与 AI 系统的风险拆成可检索、可勾选、可执行的检查项。

[在线阅读](https://cndoin.github.io/sectest-atlas/) · [Markdown 手册](./docs/handbook.md) · [安装说明](./INSTALL.md) · [AI 安装说明](./AI_INSTALL.md) · [安全政策](./SECURITY.md) · [参与贡献](./CONTRIBUTING.md)

### 内容亮点

| 维度 | 覆盖内容 |
| --- | --- |
| 方法体系 | PTES、NIST SP 800-115、OSSTMM、MITRE ATT&CK、CIS Benchmarks |
| 网络与主机 | L2–L4、DNS、边界与分段、Linux/Windows、身份与凭据 |
| 应用与平台 | Web、API、微服务、云、容器、Kubernetes、CI/CD、供应链 |
| 无线与新技术 | WPA2/WPA3、Wi-Fi 6E/7、BLE、ZigBee、AI/LLM/智能体 |
| 运营与韧性 | 日志检测、紫队、备份恢复、勒索韧性、性能与混沌工程 |
| 交付落地 | 风险评级、报告模板、授权书、ROE、检查记录与验收标准 |

### 快速使用

无需构建、无需安装依赖：直接打开 [`index.html`](./index.html)。页面支持全文目录检索、进度勾选、本地保存、深色模式、移动端目录和打印导出。

也可以直接访问已经部署好的 GitHub Pages 在线版本。

### 仓库结构

```text
.
├── index.html          # 交互式单页手册，可直接部署到 GitHub Pages
├── assets/             # 品牌图标与项目封面
├── docs/handbook.md    # Markdown 完整版
├── INSTALL.md          # 本地安装与预览说明
├── AI_INSTALL.md       # 可直接交给 AI Agent 的安装任务
├── CONTRIBUTING.md     # 内容维护与贡献约定
└── SECURITY.md         # 漏洞披露与安全边界
```

### 使用原则

1. 先授权、再测试；高风险动作需要单独确认与回滚方案。
2. 先在隔离或预生产环境验证，再按窗口进入生产。
3. 证据最小化采集，敏感数据脱敏，测试结束后按约定销毁。
4. 所有发现都应包含：现象、证据、影响、修复与复测结果。
5. 命令是核查参考，不是对任意目标的执行许可。

### 维护说明

安全标准和产品能力会持续变化。提交更新时请附权威来源、适用版本与验证日期，并明确区分强制要求、最佳实践和环境相关建议。

---

如果这份手册对你有帮助，欢迎 Star，也欢迎提交经过验证的修订。

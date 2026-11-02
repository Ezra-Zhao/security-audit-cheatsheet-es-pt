# security-audit 速查表 | security-audit Cheatsheet | Guía rápida de security-audit

> 一页看懂 Cloudflare 开源的 security-audit skill：把你的编程 agent 变成安全审计员——先画架构与信任边界、建覆盖台账，再派隔离的「猎手」分头找洞，每个候选漏洞都交给一个全新的验证者专门去证伪，最后才出结构化结论和中立报告。一句话钩子：让 AI 查漏洞，先让另一个 AI 专门证明它在胡说。这套 skill 正是 Cloudflare 自家漏洞发现系统的种子版。

- 上游项目：[cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill)（2026-11-02 实测 57,111 ★ / 3,712 forks，MIT；根目录只有英文 README）
- 平台：编程 agent 的 skill（经 Skills CLI 安装）；校验脚本为 Node.js 零依赖脚本
- 本页结构：中文主文 → English → Español → Português

---

Apoyo y más: [GitHub Sponsors](https://github.com/sponsors/Ezra-Zhao) · [Gumroad — plantillas y cuadernos que uso](https://ezraflow28.gumroad.com) · [Ko-fi](https://ko-fi.com/ezra777zhao)

## 它是什么、不是什么

| 是 | 不是 |
|---|---|
| 一套审计方法论 skill：六阶段流水线（侦察→按台账猎洞→证伪验证→结构化输出→独立复核→中立报告），全程留文件产物 | 不是一键扫描器：没有「点一下出报告」，是 agent 按阶段干活、人看产物 |
| 找洞和验洞必须是两个 agent：每个候选漏洞交给一个全新的验证者，任务是推翻它 | 不是「AI 说有洞就有洞」：没有完整源码链的结论不许标已确认、不许编严重度 |
| 覆盖靠台账驱动：先建 coverage-ledger.json，猎手按台账单元分工，覆盖评论员专找漏查的角落 | 不查依赖 CVE：依赖漏洞要另外配 Semgrep / Gitleaks / OSV 这类确定性工具（见诚实标注区） |
| 多轮运行是累加的：后续轮次读旧台账和旧结论，专补缺口、复核已变更的源码 | 不是跑一轮就完事：原仓自述单轮只能找到多轮总发现的约一半 |

## 六阶段一图流

| 阶段 | 干什么 | 产物文件 |
|---|---|---|
| 1 侦察 | 画架构、信任边界、输入面，盘点已有证据与确定性覆盖，建成台账并跑台账校验器 | `architecture.md`、`coverage-ledger.json` |
| 2 按台账猎洞 | 按台账单元派隔离的猎手分头找洞，逐项记录查过什么；覆盖评论员（coverage critics）专找没人查的缺口 | 台账逐项检查记录 |
| 3 候选验证 | 每个不重复的候选漏洞交给一个全新的验证者，任务只有一个：证伪它 | 每条候选的证伪结论 |
| 4 结构化输出 | 三种结论写入 findings.json，并用 schema 校验器过一遍（第 5 阶段每次替换后还要再过） | `findings.json`（过 `report-schema.json` 与 `validate-findings.cjs` 校验） |
| 5 独立复核 | 换一批全新 agent 复核最终的源码断言；实质性替换的条目再加一个独立验证者 | 复核后的最终记录 |
| 6 中立报告 | 只从已验证记录与覆盖台账推导报告，报告与具体目标解耦、不带立场 | `REPORT.md`、`FINDINGS-DETAIL.md`、`NEEDS-VALIDATION.md` |

运行要求：支持工具调用与并行子 agent 的编程 agent；Node.js 跑两个零依赖校验器（`validate-coverage-ledger.cjs` 在建台账后及每次台账更新后跑，`validate-findings.cjs` 在第 4 阶段及第 5 阶段每次替换后跑）。

## 三种结论判定卡（最怕看错的一页）

| 结论 | 判定线 | 严重度 |
|---|---|---|
| `confirmed` | 有完整的源码追踪链 + 有边界的实测结果（确实越过了某条边界） | 给严重度，按「可能性 × 影响」算，不按清单偏离度算 |
| `needs_validation` | 只差一个确切事实没核实——事实写明是哪一个 | **不许给严重度**，更不许猜 |
| `rejected` | 已被验证者证伪 | 不删，留档——证明这条查过、排除了 |

配套原则：纵深防御的缺口不是漏洞（A 层已经挡住攻击，B 层缺失只是加固建议）；没核实的线索宁可停在 `needs_validation`，也不许为了「有产出」升级成 confirmed。

## 装法一行版

```bash
# 项目级安装
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit

# 用户级全局安装：多加 --global
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit \
  --global
```

装完在被审代码库里启动你的编程 agent，直接说 `security audit this codebase`（或 find security vulnerabilities in ./src）即可触发。两种模式：直接审计/渗透类请求走**全量审计模式**（full audit）；安全问题咨询与聚焦某个漏洞类的工作走**指导模式**（guidance），除非你要报告产物。全量模式没指定输出目录时，默认写到 `~/security-audit-skill/<仓库名>/run-<N>`；只有你显式选了被版本控制忽略的目录，它才会往目标仓里写文件。

## 15 份指南文档，按目标选哪几份对照

原仓共 15 份 Markdown 指南：4 份是流程（SKILL 总纲、RECONNAISSANCE 侦察、HUNTING 猎洞、VALIDATION-AND-REPORTING 验证与报告），11 份是攻击面专题——猎手按目标类型翻对应的那几份：

| 专题指南 | 对照目标 |
|---|---|
| `ATTACK-CLASSES.md` | 核心 + 通配 + 「显而易见的东西」攻击提示，所有目标先过这一份 |
| `WEB-PROTOCOL-AND-AUTH.md` | HTTP 请求 framing、缓存、鉴权协议类目标 |
| `CLIENT-SIDE.md` | 浏览器/客户端：DOM 注入、消息信任、UI 劫持、原型污染 |
| `AI-AND-LLM.md` | 带 LLM 的目标：prompt 注入、agent/工具滥用、输出处理 |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依赖、CI、发布、签名、更新、插件与扩展 |
| `CLOUD-AND-DEPLOYMENT.md` | IAM、IaC、容器、serverless、入口与运行时配置 |
| `MEMORY-SAFETY-AND-BINARY.md` | 原生/二进制/内核目标的内存安全类 |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC、序列化、队列、broker、webhook、流式协议 |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 共享资源、配额、队列、worker、运营方花费类耗尽攻击 |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | 租户隔离、缓存、搜索、导出、备份、迁移、删除与恢复 |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | 桌面/移动原生应用、深链、webview、导出组件、本地 IPC |

## 诚实标注区

- **没有沙箱就不许执行被审代码**（原仓 Requirements 自述）：执行目标控制的构建、测试、进程、浏览器、模拟器、fuzzer 与夹具，必须有操作系统级强制沙箱——断外网、净化后的白名单环境、资源上限、只许写入指定草稿路径。没有这套控制，工作流只做静态审读，相关线索停在 `needs_validation`，不执行目标代码。
- **单轮只能找到约一半**（原仓设计原则自述）：他们的测试里，单轮运行找到的漏洞约为多轮运行总发现的一半。要覆盖，先按多轮规划，别拿第一轮结果当全量结论。
- **它不查依赖 CVE**（证据 B，西语长文指南口径，非原仓自述）：依赖与已知漏洞扫描请另配 Semgrep、Gitleaks、OSV 这类确定性工具，两边结果分开看。
- **严重度不是清单打分**：confirmed 的严重度按可能性 × 影响；偏离最佳实践但没有实际影响路径的，不构成漏洞结论。
- **本周势头**：57,111 ★ / 3,712 forks（2026-11-02 原仓页实测，证据 A）；本周约 +12,838 星（第三方榜单口径，证据 B）。

---

## English (summary)

Cloudflare's security-audit skill (57,111 stars / 3,712 forks as of 2026-11-02, MIT) turns a coding agent into a security auditor — it is the seed of Cloudflare's own vulnerability-discovery harness. It runs a six-phase audit: (1) reconnaissance — architecture, trust boundaries and a coverage ledger (`architecture.md`, `coverage-ledger.json`); (2) coverage-led hunting by isolated hunter agents, with coverage critics hunting for gaps; (3) candidate validation — every candidate goes to a fresh verifier whose job is to disprove it; (4) structured output — verdicts written to `findings.json` and checked against `report-schema.json`; (5) independent record verification by yet another set of fresh agents; (6) target-neutral reporting (`REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`).

The three verdicts are the heart of it: `confirmed` needs a complete source trace plus a bounded observed result (only then does it get a severity, scored as likelihood × impact); `needs_validation` names one exact unresolved fact and gets **no** severity; `rejected` candidates are kept on record. Install with one command: `npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit` (add `--global` for a user-level install), then ask your agent to "security audit this codebase". The repo ships 15 Markdown guides — 4 process guides plus 11 attack-surface guides (web/auth, client-side, AI/LLM, supply chain, cloud, memory safety, protocols, resource exhaustion, data isolation, desktop/mobile IPC, and a core attack-classes file).

Honest notes: without an OS-enforced sandbox (no external networking, sanitized allowlisted environment, resource limits, scratch-only writes) the workflow stays static-only and leaves leads at `needs_validation` instead of executing target code; in Cloudflare's own test runs a single run found roughly half of what repeated runs found in total, so plan for multiple runs; and the skill does not scan dependencies for CVEs — pair it with deterministic tools such as Semgrep, Gitleaks or OSV (per Spanish-language guides, evidence level B).

---

## Español (guía rápida — traducción de estudio, pendiente de revisión por un hablante nativo)

La security-audit skill de Cloudflare (57.111 estrellas / 3.712 forks a 2 de noviembre de 2026, MIT) convierte a tu agente de programación en un auditor de seguridad: es la semilla del propio sistema de descubrimiento de vulnerabilidades de Cloudflare. Ejecuta una auditoría en seis fases: (1) reconocimiento — arquitectura, límites de confianza y un registro de cobertura (`architecture.md`, `coverage-ledger.json`); (2) caza guiada por el registro, con agentes cazadores aislados y «críticos de cobertura» que buscan huecos; (3) validación de candidatos — cada candidato pasa a un verificador nuevo cuyo trabajo es refutarlo; (4) salida estructurada — los veredictos se escriben en `findings.json` y se validan contra `report-schema.json`; (5) verificación independiente de los registros por otro grupo de agentes nuevos; (6) informe neutral respecto al objetivo (`REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`).

Los tres veredictos son el corazón del método: `confirmed` exige una traza de código completa y un resultado observado acotado (solo entonces recibe severidad, calculada como probabilidad × impacto); `needs_validation` nombra un hecho exacto sin resolver y **no** recibe severidad; los candidatos `rejected` quedan archivados. Instalación en un comando: `npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit` (añade `--global` para instalación de usuario), y luego pide a tu agente «security audit this codebase». El repositorio trae 15 guías en Markdown: 4 de proceso y 11 de superficie de ataque (web/autenticación, lado cliente, IA/LLM, cadena de suministro, nube, seguridad de memoria, protocolos, agotamiento de recursos, aislamiento de datos, escritorio/móvil IPC, más el archivo central de clases de ataque).

**Notas honestas**: sin una sandbox impuesta por el sistema operativo (sin red externa, entorno saneado con listas permitidas, límites de recursos, escritura solo en rutas temporales), el flujo se queda en lectura estática y deja las pistas en `needs_validation` en vez de ejecutar el código auditado; en las pruebas de la propia Cloudflare, una sola pasada encontró aproximadamente la mitad de lo que encontraron las pasadas repetidas en total, así que planifica varias pasadas; y la skill no busca CVE en dependencias — combínala con herramientas deterministas como Semgrep, Gitleaks u OSV (según guías en español, evidencia nivel B).

---

## Português (guia rápido — tradução de estudo, aguardando revisão de um falante nativo)

A security-audit skill da Cloudflare (57.111 estrelas / 3.712 forks em 2 de novembro de 2026, MIT) transforma seu agente de programação em um auditor de segurança: é a semente do próprio sistema de descoberta de vulnerabilidades da Cloudflare. Ela executa uma auditoria em seis fases: (1) reconhecimento — arquitetura, fronteiras de confiança e um registro de cobertura (`architecture.md`, `coverage-ledger.json`); (2) caça guiada pelo registro, com agentes caçadores isolados e «críticos de cobertura» procurando lacunas; (3) validação de candidatos — cada candidato vai para um verificador novo cujo trabalho é refutá-lo; (4) saída estruturada — os vereditos são escritos em `findings.json` e validados contra `report-schema.json`; (5) verificação independente dos registros por outro grupo de agentes novos; (6) relatório neutro em relação ao alvo (`REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`).

Os três vereditos são o coração do método: `confirmed` exige um rastro de código completo e um resultado observado delimitado (só então recebe severidade, calculada como probabilidade × impacto); `needs_validation` nomeia um fato exato não resolvido e **não** recebe severidade; os candidatos `rejected` ficam arquivados. Instalação em um comando: `npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit` (adicione `--global` para instalação de usuário), e depois peça ao seu agente «security audit this codebase». O repositório traz 15 guias em Markdown: 4 de processo e 11 de superfície de ataque (web/autenticação, lado do cliente, IA/LLM, cadeia de suprimentos, nuvem, segurança de memória, protocolos, esgotamento de recursos, isolamento de dados, desktop/mobile IPC, mais o arquivo central de classes de ataque).

**Notas honestas**: sem uma sandbox imposta pelo sistema operacional (sem rede externa, ambiente saneado com listas de permissão, limites de recursos, escrita apenas em rotas temporárias), o fluxo fica só na leitura estática e deixa as pistas em `needs_validation` em vez de executar o código auditado; nos testes da própria Cloudflare, uma única passada encontrou aproximadamente metade do que as passadas repetidas encontraram no total, então planeje várias passadas; e a skill não procura CVE em dependências — combine-a com ferramentas determinísticas como Semgrep, Gitleaks ou OSV (segundo guias em espanhol, evidência nível B).

---

## 来源与声明

- 灵感来源 / Source / Fuente / Fonte: [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill)（背景博客：Cloudflare《Build your own vulnerability harness》）
- 本仓是个人学习整理：内容用我自己的话总结自上游公开文档，不搬运原文；阶段产物、结论判定与安装命令以 2026-11-02 的上游 README 与文件清单为准，上游迭代后请以官方文档为最终依据。
- 西语、葡语部分为学习译稿，待母语校对。Las secciones en español y portugués son traducciones de estudio pendientes de revisión nativa.
- License: MIT © 2026 Guangyi Zhao

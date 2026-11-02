# 第一次全量审计演练 | First full-audit drill: from install to a three-report closeout

把「在自己仓上跑一轮完整审计」这条链路一次走通，顺序别乱：

1. **先备环境**：确认你的编程 agent 支持工具调用与并行子 agent；装好 Node.js（两个校验器要用）；最重要的是备好操作系统级沙箱（断外网、白名单环境、资源上限、只许写草稿路径）——没有沙箱，这一轮就只做静态审读，别让 agent 执行被审代码。
2. **装 skill**：`npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit`（多仓共用就加 `--global`）。
3. **选指南**：按目标类型从 11 份攻击面专题里圈出要对照的几份（Web 应用先圈 WEB-PROTOCOL-AND-AUTH + CLIENT-SIDE，带 LLM 的再加 AI-AND-LLM，原生程序换 MEMORY-SAFETY-AND-BINARY），写下来，审计时逐份销项。
4. **跑全量审计**：在被审仓里对 agent 说 `security audit this codebase`，确认它先产出 `architecture.md` 与 `coverage-ledger.json` 再开始猎洞；中途只看产物文件，不口头催结论。
5. **收三份报告**：跑完应有 `REPORT.md`、`FINDINGS-DETAIL.md`、`NEEDS-VALIDATION.md`，且 `findings.json` 过了 schema 校验。先读 NEEDS-VALIDATION：每条都写明了差哪一个确切事实——那是你下一轮的清单，不是失败清单。
6. **排第二轮**：单轮约只能找到多轮总发现的一半（原仓自述）。把第 5 步的未决事实补上、源码有变更的部分，排进第二轮复跑；两轮的 confirmed 集合并起来，才接近这套方法论的真实覆盖。

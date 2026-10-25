# ghadoctor 优化日志

## Round 1（2026-10-25）

**存活评估**：保留。actionlint / zizmor 是重型（需装/编译/配置），ghadoctor 的「零依赖单文件、确定性强、开箱即跑 + 健康分 + CI 门禁」填补轻量零依赖缝隙，无碾压级替代。

**方向**：单点深度特化「门禁真实性」—— 这是 code-health 家族最高频、最致命的一类缺陷。

### 实测抓到的真缺陷（全部已修）

1. **有害指标（第 11 轮）**：空目录扫描给「健康分 100/100」。改为 `basisComplete=false` → `score=null`，门禁返回 `inconclusive`(rc=3)。
2. **门禁三态缺失（第 9.9/10.2 轮）**：`gatePassed` 默认 true，未配门禁/空输入都报「PASS」；JSON 只有布尔 `passed`。新增 `pass/fail/inconclusive` 三态 + `state`/`configured`/`basisComplete`/`reason`。
3. **process.exit 截断 stdout（第 7.3 轮）**：打印 `--json` 后 `process.exit(2)`。改 `process.exitCode`。
4. **参数容错（第 10.4 轮）**：`--max-high ''` → `Number('')=0` 静默通过；缺整数校验。新增整数 + 空串 + 未知选项（`-x`/长选项/缺值）全拒绝 rc=2。
5. **零 CI / 零 LICENSE**：新增 `package.json`(engines>=18) + `LICENSE`(MIT) + `.github/workflows/ci.yml`（ubuntu/windows × Node 18/20/22 + dogfood 自体检归零）。
6. **README 死命令**：安装命令写 `node ghadoctor.js`（实际入口 `index.js`）—— 已改 `index.js`；补 badges + 英文摘要 + 顶部家族导流 + 版本对齐 18。

### 测试

- 单测 27 → 36（新增参数容错 9 项），全绿。
- 新增 `test/smoke-cli.js`（spawn 级，唯一能验退出码契约）：10 项，全绿。
- 变异测试 3 例（M1 空扫描给满分 / M2 空输入门禁回退 pass / M3 未知选项静默吞）全部 CAUGHT。

### 闭合

分支 `gh-optimizer/2026-10-25` → PR → CI 绿 → squash 合并 → 逐文件核对 master。

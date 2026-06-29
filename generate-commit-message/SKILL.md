---
name: generate-commit-message
description: >-
  分析 git 改动并生成中文 Conventional Commits 提交信息（含单行/正文/改动摘要/破坏性变更选项）。
  在用户说「生成提交注释」「写 commit message」「帮我提交信息」、或要求根据 diff 起草提交说明时使用。
  仅生成注释时不执行 git commit，除非用户明确要求提交。
---

# 生成 Git 提交注释

根据工作区改动生成提交信息。**默认只输出注释，不执行 commit**；用户明确说「提交」「commit」时再走提交流程。

## 工作流

### 1. 并行收集上下文

在同一轮中并行执行：

```bash
git status
git diff HEAD          # 含已暂存 + 未暂存；若用户只要 staged 则用 git diff --cached
git log -8 --oneline   # 对齐仓库近期 subject 风格
```

必要时补充：

- `git diff --cached` — 用户明确只要已暂存改动
- `git diff [base]...HEAD` — 为 PR/分支整体写摘要

### 2. 分析改动

- 区分 **已暂存 / 未暂存 / 未跟踪**；注释范围与用户意图一致（默认：即将提交的全部改动）
- 按文件或模块归纳：**做了什么、为什么**（fix 根因、feat 能力），避免逐行复述 diff
- 对照 `git log` 的 type、scope、语气，保持与仓库一致
- **不要**把 `.env`、密钥等敏感文件写进建议的提交范围；若用户要提交这些文件，先警告
- **识别破坏性变更**：若 diff 涉及以下情况，视为潜在破坏性变更并单独列出（见 §4）：
  - 删除/放宽校验、必填项，导致旧数据或旧流程可通过
  - 删除/重命名对外 API、props、事件、路由、字段
  - 改变默认行为，使依赖旧行为的调用方失效
  - 不兼容的数据结构或接口契约变更
  - 纯 UI/样式/内部重构、行为不变 → **不算**破坏性变更

### 3. 撰写提交信息

遵循 **Conventional Commits**，**subject 与正文均使用中文**：

```
type(scope?): subject

可选正文：1–3 条 bullet，说明原因或影响面
```

| type | 适用 |
|------|------|
| feat | 新功能 |
| fix | 缺陷修复 |
| refactor | 行为不变的重构 |
| perf | 性能 |
| style | 格式/样式，不改逻辑 |
| docs | 文档 |
| chore / ci / build | 工具链、依赖、构建 |

**subject 规则**：祈使句、≤50 字、结尾无句号；scope 用模块名（如 `auto-rule`、`auth`）。

### 4. 输出格式

向用户返回：

1. **推荐单行**（可直接 `git commit -m`）
2. **可选带正文**（多文件或需说明原因时）
3. **改动摘要**（简短表格或列表：文件 → 要点）
4. **破坏性变更（如有）** — 单独一节，供用户选择是否写入提交信息

#### 破坏性变更输出规则

若 §2 识别到破坏性变更，**必须**额外输出以下结构（与普通提交信息分开）：

```markdown
## ⚠️ 破坏性变更

| # | 变更 | 旧行为 | 新行为 | 影响面 |
|---|------|--------|--------|--------|
| 1 | … | … | … | … |

**是否在提交信息中标注？**（请用户选择）

- **A. 不标注** — 使用上方「推荐单行 / 可选带正文」（默认推荐，适合内部前端、行为放宽类改动）
- **B. 标注破坏性变更** — 在 subject 使用 `type(scope)!: subject`，并在正文末尾追加：
  ```
  BREAKING CHANGE: <一句话说明破坏点与迁移/影响>
  ```
```

规则：

- 无破坏性变更时，**省略**第 4 节，不强行输出
- 有破坏性变更时，**同时**给出 A、B 两套完整 commit message，方便用户直接选用
- 默认推荐 A，但在 B 的 `BREAKING CHANGE` 行写清楚「旧 → 新」
- **不要**擅自替用户选定；等用户回复 A/B 后再 commit（若用户要求提交）

不要擅自执行 `git commit`，除非用户明确要求。

---

## 示例

**场景**：规则列表账户筛选项为空；TikTok Smart 类型 default 分支误返回全量枚举。

**输出**：

```
fix(auto-rule): 修复规则列表账户筛选与 TikTok Smart 类型默认选项

- 规则列表搜索表单「广告账户」下拉改为使用 accountOptions
- getAllowedSmartTypes 默认分支仅返回 MANUAL/SMART_1/SMART_2
```

**场景（含破坏性变更）**：移除表单 Deeplink URI Schema 校验。

**输出**：

```
feat(tiktok): 追加创意组接入监测并移除 Deeplink 格式校验

- 列表追加创意组集成 TiktokMonitor
- 移除 apps_cta Deeplink URI Schema 格式校验
```

## ⚠️ 破坏性变更

| # | 变更 | 旧行为 | 新行为 | 影响面 |
|---|------|--------|--------|--------|
| 1 | 移除 Deeplink 校验 | 非 iOS14 应用 Deeplink 须符合 URI Schema | 任意字符串均可提交 | 可能提交无效 Deeplink 至后端 |

**是否在提交信息中标注？**

- **A. 不标注**（上方推荐版本，默认）
- **B. 标注破坏性变更**：
  ```
  feat(tiktok)!: 移除创意组 Deeplink URI Schema 格式校验

  - 列表追加创意组集成 TiktokMonitor

  BREAKING CHANGE: 创意组 apps_cta Deeplink 不再校验 URI Schema 格式，旧流程中不符合协议的链接此前无法提交，现在可以提交。
  ```

---

## 用户要求提交时

仅在用户明确要 commit 时，再按仓库/用户规则执行：

1. `git add` 相关文件（排除敏感文件）
2. `git commit -m "$(cat <<'EOF' ... EOF)"`（HEREDOC 传多行 message）
3. `git status` 确认成功；hook 失败则修问题后**新 commit**，勿随意 `--amend`

**不要** `git push`，除非用户明确要求。

---

## 反模式

- 未读 diff 就猜测改动内容
- subject 用英文（本 skill 默认中文，除非用户指定英文）
- 把「生成注释」和「执行提交」混为一谈
- 一次 commit 消息覆盖无关的大杂烩改动（应建议拆分）
- 发现破坏性变更却只在正文顺带一提、不单独列出 A/B 选项
- 未经用户选择就使用 `!` 或 `BREAKING CHANGE`

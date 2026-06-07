---
name: skill-deployer
description: |
  将任何本地 skill 部署为独立 GitHub 仓库。自动生成 README、LICENSE(MIT)、版本标签、GitHub Actions 验证。
  
  适用场景:
  - 用户要把自建 skill 发布为独立开源仓库
  - skill 达到收费产品标准(经 skill-review G1-G6 全过)需要公开发布
  - 需要为 skill 生成专业 README 和项目结构
  - 批量部署多个 skill 到 GitHub
  
  不适用场景:
  - skill 还不是最终版(还在迭代中) → 先跑 skill-review 确认达标
  - 只是修改已有仓库 → 直接用 git push
  - 创建新 skill(从零写) → skill-creator
  - 评测 skill 质量 → skill-review
  
  触发词: 部署skill, 发布skill, skill发布, 创建skill仓库, skill repo,
           skill上线, 开源skill, 推送skill到GitHub, skill deploy
  
  边界: skill-deployer 负责打包+推送+生成文档，不负责 skill 内容质量(由 skill-review 保证)。
        与 skill-creator 的区别: skill-creator 创建/改写 SKILL.md，skill-deployer 将成品部署到 GitHub。
  
  正例:
  - "把这个skill部署到GitHub" → 触发
  - "skill-review通过了，帮我发布成独立仓库" → 触发
  - "批量部署这3个skill" → 触发
  
  反例:
  - "帮我改这个skill的内容" → skill-creator
  - "评测这个skill质量" → skill-review
  - "push一下代码" → 直接用 git
---

# Skill Deployer v1.0 — 收费产品级部署引擎

## 前置条件(任一不满足=中止，输出缺失项)
- [ ] 源 skill 存在: `Test-Path <skill_dir>/SKILL.md`
- [ ] 已通过 skill-review 收费产品门禁(G1-G6 全绿，或用户明确跳过质检)
- [ ] GitHub CLI 已认证: `gh auth status` 返回成功
- [ ] Git 已配置 user.name 和 user.email

## 部署流程

### Step 1: 读取源 skill 元数据
   - 解析 frontmatter: name, description
   - 统计 body token 数(估算: bytes÷2≈tokens)
   - 提取版本块(从 body 末尾 `## 版本` 段)
   - 输出: skill 摘要(名称/版本/token/主题)

### Step 2: 生成仓库名
   - 规则: `skill-{name}`（如 `skill-review` → `skill-skill-review`；去重 `skill-` 前缀）
   - 检查 GitHub 上是否已存在同名仓库: `gh repo view KKKKhazix/<repo-name> --json name`
   - 冲突处理: 若已存在→询问"覆盖/跳过/重命名"

### Step 3: 构建仓库骨架
   在临时目录创建:
   ```
   <repo-name>/
   ├── SKILL.md              # 原 skill 文件(不修改)
   ├── README.md             # 自动生成(见模板)
   ├── LICENSE               # MIT 许可证
   ├── .gitignore            # 排除 .DS_Store, Thumbs.db, node_modules
   └── .github/
       └── workflows/
           └── validate.yml  # 自动验证 SKILL.md 结构完整性
   ```

### Step 4: 生成 README.md(按模板逐字段填充)
   ```markdown
   # <skill-name>
   
   > <description>
   
   ## 安装
   ```bash
   npx -y skills add KKKKhazix/<repo-name> -g --all
   ```
   
   ## 触发方式
   | 触发词 | 示例 |
   |--------|------|
   | <词1> | "<示例>" |
   | <词2> | "<示例>" |
   
   ## 质量保证
   | 指标 | 值 |
   |------|-----|
   | skill-review 评分 | XX/100 |
   | 收费产品门禁 | ✅ G1-G6 全过 |
   | Token 预算 | XXX tokens |
   
   ## 版本
   | 版本 | 日期 | 变更 |
   |------|------|------|
   | vX.X | YYYY-MM-DD | [从 SKILL.md 版本块提取] |
   
   ## 许可证
   MIT © [年份] KKKKhazix
   ```

### Step 5: 生成 validate.yml(GitHub Actions)
   ```yaml
   name: Validate SKILL.md
   on: [push, pull_request]
   jobs:
     validate:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Check SKILL.md exists
           run: test -f SKILL.md
         - name: Check frontmatter
           run: head -1 SKILL.md | grep -q '^---$'
         - name: Check no hardcoded tokens
           run: '! grep -Eq "sk-[a-zA-Z0-9]{20,}|github_pat_[a-zA-Z0-9]{20,}" SKILL.md'
         - name: Check size
           run: 'test $(wc -c < SKILL.md) -le 10000'
   ```

### Step 6: 创建 GitHub 仓库并推送
   ```powershell
   cd <temp_dir>/<repo-name>
   git init
   git add -A
   git commit -m "vX.X: 初始发布 — 收费产品级 skill" --quiet
   gh repo create KKKKhazix/<repo-name> --public --source=. --remote=origin --push
   ```
   验证: `gh repo view KKKKhazix/<repo-name> --json name,url`

### Step 7: 输出部署报告
   ```markdown
   ## Skill 部署报告
   | 项目 | 值 |
   |------|-----|
   | Skill 名称 | <name> vX.X |
   | GitHub 仓库 | https://github.com/KKKKhazix/<repo-name> |
   | 安装命令 | `npx -y skills add KKKKhazix/<repo-name> -g --all` |
   | Token 预算 | XXX tokens |
   | 质量评分 | XX/100 (skill-review) |
   | 门禁状态 | ✅ 6/6 通过 |
   | 部署时间 | YYYY-MM-DD HH:MM |
   ```

## 失败兜底
| 失败 | 动作 |
|------|------|
| `gh auth status` 失败 | 输出 "GitHub CLI 未认证，运行 `gh auth login`" → 中止 |
| 仓库名冲突 | 输出冲突信息 → 给出3选项(覆盖/跳过/重命名) → 等待选择 |
| `gh repo create` 失败 | 输出错误原因 → 中止(常见: 网络/权限/organization 不存在) |
| SKILL.md 无版本块 | 降级: 自动标注 v1.0 + "版本块缺失，请补充" |
| 临时目录创建失败 | 输出原因 → 中止 |

## 约束
- 不修改源 SKILL.md(只读原文件，副本写入新仓库)
- 默认公开仓库(除非用户指定 `--private`)
- License 固定 MIT(除非用户指定其他)
- 部署前检查 skill-review 门禁(除非用户 `--skip-review`)
- 部署后不删除临时目录(保留用于检查)

---
## 版本
- v1.1 | 2026-06-07 | R2: 批量部署+部署历史
- v1.0 | 2026-06-07 | 初始版本(收费产品级) | 原因: 系统级skill需要独立GitHub仓库+CI验证+专业README
## R2增强: 批量部署 + 部署历史追踪

### 批量部署模式
触发: "批量部署" / "部署所有系统skill"
自动扫描 `C:\Users\董辉\.agents\skills\` 和 `D:\_ai\skills\` → 筛选已通过 skill-review 门禁的 skill → 逐个部署
输出汇总: Skill | 仓库 | 状态 | 版本

### 部署历史
每次部署记录到 `D:\KnowledgeBase\_logs\ledger\DEPLOY_HISTORY.md`:
```
[YYYY-MM-DD HH:MM] skill-name vX.X → songxrui/skill-name ✅ | 门禁: 6/6
```
用途: 追溯部署时间线、检测需要更新的仓库
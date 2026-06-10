# Maintenance Skill Iterator

用于维护 `plan-generator-demo` 中检修方案生成 Skill 的质量自迭代仓库。

这个仓库把 Codex Skill 独立出来，便于团队成员共同修改、评审和复用。核心思路是：不直接评价 `SKILL.md` 写得是否漂亮，而是先让项目实际生成 DOCX，再把生成文档与高质量历史检修方案对比，最后将缺陷反推回对应产品 Skill。

```text
产品 Skill -> /api/dev/plan-test 生成 DOCX -> 评估 DOCX -> 修改产品 Skill -> 再生成 -> 再评估
```

## 目录结构

```text
maintenance-skill-iterator/
├── skills/
│   └── maintenance-skill-iterator/
│       ├── SKILL.md
│       ├── agents/openai.yaml
│       └── scripts/
│           ├── evaluate_plan_quality.py
│           └── generate_and_evaluate_plan.py
├── .gitignore
└── README.md
```

## 安装到 Codex

将 Skill 目录复制到本机 Codex skills 目录：

```powershell
Copy-Item -Recurse -Force `
  ".\skills\maintenance-skill-iterator" `
  "$env:USERPROFILE\.codex\skills\maintenance-skill-iterator"
```

重新打开 Codex 线程后，`maintenance-skill-iterator` 会出现在可用 Skill 列表中。

## 依赖

脚本依赖 Python 3.10+ 和 `python-docx`。如果在 `plan-generator-demo` 的虚拟环境中运行，通常已经具备依赖；否则手动安装：

```bash
pip install python-docx
```

## 推荐环境变量

不同成员本机路径不同，建议用环境变量统一：

```powershell
$env:PLAN_GENERATOR_PROJECT="C:\Users\26441\Documents\New project\plan-generator-demo"
$env:PLAN_GENERATOR_API_URL="http://127.0.0.1:8000/api/dev/plan-test"
$env:MAINTENANCE_REFERENCE_ROOT="D:\个人工作\2026\微创项目\整理后文档\检修方案-整理"
$env:MAINTENANCE_ITERATOR_OUTPUT="$env:PLAN_GENERATOR_PROJECT\iterator-output"
```

## 快速使用

先启动 `plan-generator-demo` 后端：

```powershell
cd "$env:PLAN_GENERATOR_PROJECT\backend"
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

使用完整 state 生成候选 DOCX 并评估：

```powershell
python .\skills\maintenance-skill-iterator\scripts\generate_and_evaluate_plan.py `
  --api-url "$env:PLAN_GENERATOR_API_URL" `
  --state-file ".\examples\slb-recycle-state.json" `
  --reference-dir "$env:MAINTENANCE_REFERENCE_ROOT\SLB" `
  --source-skill "$env:PLAN_GENERATOR_PROJECT\backend\skills\slb-maintenance-plan\SKILL.md" `
  --output-dir "$env:MAINTENANCE_ITERATOR_OUTPUT"
```

只评估已有 DOCX：

```powershell
python .\skills\maintenance-skill-iterator\scripts\evaluate_plan_quality.py `
  --reference-dir "$env:MAINTENANCE_REFERENCE_ROOT\SLB" `
  --candidate-docx "C:\path\to\检修方案.docx" `
  --source-skill "$env:PLAN_GENERATOR_PROJECT\backend\skills\slb-maintenance-plan\SKILL.md"
```

## 维护原则

- 评估脚本只做通用评估与契约执行，不写死具体产品关键词。
- 每类产品必须在自己的 `SKILL.md` 中维护 `## 质量评估契约`。
- 新增 ECS、RDS、Redis、OSS、MQ、K8s 等类型时，优先修改对应产品 Skill 的契约，而不是修改 iterator 脚本。
- 本仓库不存放历史检修方案原文、生成 DOCX、API Key、`.env` 或运行日志。

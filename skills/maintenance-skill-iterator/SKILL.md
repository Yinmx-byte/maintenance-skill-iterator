---
name: maintenance-skill-iterator
description: 用于检修方案生成 Skill 的质量自迭代：先用某个产品 Skill 生成 DOCX，再将生成结果与高质量历史检修方案对比，最后把生成结果中的缺陷反推为 SKILL.md 修改建议。适用于 检修方案 Skill 自迭代、生成结果评估、对比历史方案、根据生成文档优化 Skill、多轮测试-优化-再测试。
---

# 检修 Skill 自迭代

## 用途

这个 Skill 用于开发期优化项目中的检修方案生成 Skill，不属于业务运行时能力。它的主目标不是“直接评估 SKILL.md 写得好不好”，而是评估 **由 Skill 实际生成出的 DOCX 是否足够接近高质量历史方案**，再把问题反推回产品 Skill。

主闭环：

```text
产品 SKILL.md -> 项目接口生成 DOCX -> 评估生成 DOCX -> 反推 SKILL.md 修改 -> 再生成 -> 再评估
```

静态评估 `SKILL.md` 只能作为预检，不能替代生成结果评估。

## 项目上下文配置

默认优化对象是 plan-generator-demo。不同成员本机路径可能不同，优先使用环境变量或命令参数覆盖：

- `PLAN_GENERATOR_PROJECT`：plan-generator-demo 项目根目录。
- `PLAN_GENERATOR_API_URL`：快速测试接口，默认 `http://127.0.0.1:8000/api/dev/plan-test`。
- `MAINTENANCE_REFERENCE_ROOT`：高质量历史检修方案根目录。
- `MAINTENANCE_ITERATOR_OUTPUT`：生成结果与评估报告输出目录。

本机示例：

- 项目路径：`C:\Users\26441\Documents\New project\plan-generator-demo`
- 后端路径：`C:\Users\26441\Documents\New project\plan-generator-demo\backend`
- 参考文档根目录：`D:\个人工作\2026\微创项目\整理后文档\检修方案-整理`

启动方式：在 plan-generator-demo 的 `backend` 目录执行 `uvicorn main:app --reload --host 0.0.0.0 --port 8000`。

快速测试接口请求要优先传完整 `state`，减少需求抽取对迭代结果的干扰。该接口只负责生成 DOCX，不负责评估。评估由本 Skill 的脚本在项目外部完成。

示例请求体：

```json
{
  "state": {
    "background": "外网统一车辆系统存在空闲 SLB 实例，项目组已确认无业务流量，需通过本次检修回收该空闲 SLB 实例。",
    "maintenance_type": "配置变更",
    "network": "外网",
    "location": "国网亦庄数据中心二期运维专区",
    "instances": "外网统一车辆系统空闲 SLB 实例：slb-unicar-idle-01；组织：总部直属单位；资源集：统一车辆外网资源集；监听：80/443；后端服务器组已清空；DNS/业务入口已解绑。",
    "schedule_year": "2026年",
    "schedule_start": "2026年06月10日 22:00",
    "schedule_end": "2026年06月10日 23:00",
    "provider": "张三",
    "executor": "李四",
    "reviewer": "王五",
    "security_officer": "赵六",
    "ascm_account": "ascm_ops_unicar",
    "bastion_account": "bastion_ops_unicar",
    "ops_detail": "重点写清楚释放前确认监听无流量、DNS 和业务入口已解绑、释放后业务访问无异常。",
    "tech_params": "SLB 实例名称 slb-unicar-idle-01，网络类型外网，监听端口 80/443，目标动作：回收空闲 SLB。"
  },
  "allow_partial": false
}
```

## 输入

- 参考目录：同一检修类型下的高质量历史 `.docx` 检修方案。
- 生成结果：当前产品 Skill 生成出的候选 `.docx`。
- 来源 Skill：生成该候选 DOCX 的产品 `SKILL.md`，用于把问题转成可写入的修改建议。
- 可选动作提示：例如 `SLB 回收`、`ECS 创建`、`RDS 变更`。

## 质量评估契约规范

业务检查项必须写在产品 `SKILL.md` 中，而不是写进 iterator 脚本。iterator 只负责解析并执行契约。

每个产品 Skill 建议包含：

```markdown
## 质量评估契约

### common

- 产品名或核心资源名
- 通用资源对象
- 风险评估
- 实施步骤
- 回滚步骤

### create

- 创建动作必须覆盖的字段或动作
- 创建后验证项
- 创建场景回滚项

### recycle

- 回收动作必须覆盖的字段或动作
- 回收前验证项
- 回收场景回滚项
```

约定：

- `common` 总是参与评估。
- 动作小节使用 `create`、`recycle`、`resize`、`restart`、`drill`、`ipv6` 等短名称。
- 小节内每个列表项都是生成结果 DOCX 必须出现或等价覆盖的检查项。
- 新增 RDS、Redis、OSS、MQ、K8s 等类型时，只改对应产品 Skill 的契约，不改 iterator 脚本。
- 如果源 Skill 没有 `## 质量评估契约`，iterator 只执行通用章节、风险、步骤、回滚和格式检查，并输出契约缺失 finding。

## 多轮迭代方式

可以在单次对话中执行多轮迭代。每一轮都遵循：

1. 调用项目的快速测试接口生成候选 DOCX。
2. 运行 `scripts/evaluate_plan_quality.py`，用参考目录评估候选 DOCX。
3. 阅读评分、findings 和必要的参考文档片段。
4. 将问题归因到来源 Skill：缺章节、缺风险项、步骤空泛、回滚不闭环、格式不稳定等。
5. 修改来源 `SKILL.md`，只写能提升生成结果的规则，不把评估脚本塞回项目运行时。
6. 重新生成 DOCX，再次评估。

建议停止条件：

- 生成结果评分达到目标，例如 `>=90` 或用户指定阈值。
- `high` findings 清零。
- 关键人工检查项通过：风险评估贴合动作、实施步骤可执行、回滚步骤闭环。
- 连续两轮改动后问题不再改善，此时需要更多参考文档、需求字段或模型能力调整。

## 工作流

1. 先确认评估对象是生成结果 DOCX，而不是只评估 Skill 文本。
2. 检查参考目录，判断当前产品/动作类型。
3. 运行评估脚本：
   - 首选 `--candidate-docx`
   - 同时传入 `--source-skill`
4. 结合脚本 findings 和参考文档样本判断：
   - 生成 DOCX 是否保留稳定章节和编号？
   - 风险项是否绑定具体动作，而不是泛泛而谈？
   - 实施步骤是否能指导后续自动化或人工验证？
   - 回滚步骤是否与本次动作一一对应？
   - 表格、封面、标题、人员/窗口信息是否接近参考方案？
5. 如果 finding 来自 `contract`，优先修改源产品 Skill 的生成规则或质量评估契约；如果 finding 来自 `format`，优先修改 composer Skill 或渲染器。
6. 只有用户要求“直接修改”时才写入来源 `SKILL.md`。

## 命令

生成候选 DOCX 并立即评估，主路径：

```powershell
python <skill-dir>\scripts\generate_and_evaluate_plan.py `
  --api-url "$env:PLAN_GENERATOR_API_URL" `
  --state-file "C:\path\to\state.json" `
  --reference-dir "$env:MAINTENANCE_REFERENCE_ROOT\SLB" `
  --source-skill "$env:PLAN_GENERATOR_PROJECT\backend\skills\slb-maintenance-plan\SKILL.md" `
  --output-dir "$env:MAINTENANCE_ITERATOR_OUTPUT"
```

评估生成结果 DOCX，主路径：

```powershell
python <skill-dir>\scripts\evaluate_plan_quality.py `
  --reference-dir "$env:MAINTENANCE_REFERENCE_ROOT\SLB" `
  --candidate-docx "C:\Users\26441\Downloads\检修方案_xxxxxxxx.docx" `
  --source-skill "$env:PLAN_GENERATOR_PROJECT\backend\skills\slb-maintenance-plan\SKILL.md"
```

静态预检 Skill，辅助路径：

```powershell
python <skill-dir>\scripts\evaluate_plan_quality.py `
  --reference-dir "$env:MAINTENANCE_REFERENCE_ROOT\SLB" `
  --candidate-skill "$env:PLAN_GENERATOR_PROJECT\backend\skills\slb-maintenance-plan\SKILL.md"
```

用 `--output path.json` 保存原始报告。

## 报告口径

向用户汇报时优先说明：

- 当前是第几轮迭代。
- 评估模式：`generated_docx` 或 `skill_static_preflight`。
- 生成结果得分和主要扣分维度。
- `high` findings 是否清零。
- 本轮要改的 Skill 规则。
- 是否建议继续下一轮。

不要把本评估当成生产可靠性保证。它是开发期反馈环，用于不断改善 Skill 的生成质量。

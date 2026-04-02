---
name: business-trip-assistant
description: 出差全流程管理助手。当用户提到出差、出差申请、出差日程、出差报告、会议纪要、生成出差ID、记录会议、结束出差、设置出差模版等场景时触发。支持：(1)创建出差ID并初始化出差档案；(2)用自然语言描述出差内容自动生成详细日程；(3)根据文字/音频/docx文件生成会议纪要；(4)出差结束后生成完整出差报告并与原申请日程对比；(5)管理会议纪要和出差报告的Word模版。
---

# Business Trip Assistant — 出差全流程管理助手

## 数据存储结构

所有出差数据存储于 skill 目录下 `data/{trip-id}/`：

```
data/
  TRIP-北京-20260320/
    meta.json          ← 出差元信息（目的地、日期、申请日程）
    schedule.md        ← AI生成的出差日程
    meetings/
      meeting-001.md   ← 会议纪要
      meeting-002.md
    report.md          ← 最终出差报告
templates/             ← 由用户自行上传私人模版（不随 skill 分发）
  meeting-minutes.docx ← 默认会议纪要模版（用户上传后生效）
  trip-report.docx     ← 默认出差报告模版（用户上传后生效）
```

**⚠️ 路径说明：** skill 安装后，其根目录（`SKILL_DIR`）由 OpenClaw 动态确定。
在执行任何文件操作前，AI 应先通过以下方式获取 skill 根目录：
```bash
# 找到本 SKILL.md 所在目录即为 SKILL_DIR
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
```
后续所有路径均基于 `$SKILL_DIR`，例如：
- 数据目录：`$SKILL_DIR/data/`
- 模版目录：`$SKILL_DIR/templates/`
- 用户档案：`$SKILL_DIR/profiles/`

## Trip ID 格式

`TRIP-{目的地城市}-{YYYYMMDD}`

示例：`TRIP-北京-20260320`、`TRIP-上海-20260401`

如同一天同一城市有多次出差，追加序号：`TRIP-北京-20260320-2`

## 功能路由

根据用户意图，选择对应子 skill：

| 用户意图 | 子 Skill |
|---------|---------|
| 首次使用/初始化环境/检查依赖 | [setup](sub-skills/setup/SKILL.md) |
| 设置个人信息、录入档案、我叫XX、切换用户、查看档案 | [user-profile](sub-skills/user-profile/SKILL.md) |
| 新建出差、创建出差ID、开始出差 | [trip-init](sub-skills/trip-init/SKILL.md) |
| 生成/查看出差日程 | [schedule-gen](sub-skills/schedule-gen/SKILL.md) |
| 记录会议、上传录音/文件、生成会议纪要 | [meeting-notes](sub-skills/meeting-notes/SKILL.md) |
| 结束出差、生成出差报告 | [report-gen](sub-skills/report-gen/SKILL.md) |
| 查看/上传/更换模版 | [template-mgr](sub-skills/template-mgr/SKILL.md) |
| 修改/调整/微调已生成内容、记住偏好规则、"这里不对"、"以后要这样" | [content-refine](sub-skills/content-refine/SKILL.md) |

## 首次使用自动检查

**每个子 skill 执行前**，先检查 `profiles/.setup_done` 是否存在：
- **存在** → 直接执行，无额外开销
- **不存在** → 先执行 [setup](sub-skills/setup/SKILL.md) 完成环境初始化，再继续原任务

读取对应子 skill 的 SKILL.md 后再执行。

## 模版使用规则

- 用户**指定了模版**：使用指定模版，走 OOXML 填充流程
- 用户**未指定模版**：
  - 会议纪要 → `templates/meeting-minutes.docx`
  - 出差日程 → `templates/trip-schedule.docx`
  - 出差报告 → `templates/trip-report.docx`
- 若默认模版不存在：使用内置 Markdown/Node.js docx 格式生成，并提醒用户可上传模版

**⚠️ 重要：有模版时必须走 OOXML 深度解析 + 填充流程**
无论是会议纪要、出差日程还是出差报告，只要存在 `.docx` 模版文件，都必须：
1. 解包模版，深度解析 OOXML 结构（占位符、编码类型、字体格式、表格结构）
2. 按模版结构精准填充，保留原始 pPr/rPr 格式，只替换文字内容
3. 生成后强制自检（XML 合法性 → 内容完整性 → 空单元格扫描），问题修复后重新检查，直到通过

## 企微文档同步

每次生成/更新内容后，使用 `wecom-doc` skill 将内容同步到用户企微个人空间文档。
文档命名规则：`{trip-id} - {内容类型}`，例如：
- `TRIP-北京-20260320 - 出差日程`
- `TRIP-北京-20260320 - 会议纪要 001`
- `TRIP-北京-20260320 - 出差报告`

## 当前出差上下文

若用户未指明 trip-id，查找 `data/` 目录下最近修改的出差档案作为默认上下文，并告知用户当前操作针对哪个出差ID。

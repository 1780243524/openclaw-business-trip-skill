---
name: report-gen
description: 出差结束后，根据所有会议纪要和出差日程，生成完整的出差报告，并与出差申请日程进行对比分析。当用户说"出差结束了"、"帮我生成出差报告"、"所有会议开完了"时触发。
---

# Report Gen — 出差报告生成

## 步骤

### 0. 确定路径

```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
DOCX_SKILL=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills /usr/local/lib -name 'SKILL.md' -path '*/docx/SKILL.md' 2>/dev/null | head -1)")
UNPACK="$DOCX_SKILL/ooxml/scripts/unpack.py"
PACK="$DOCX_SKILL/ooxml/scripts/pack.py"
```

### 1. 加载出差数据

读取 `$SKILL_DIR/data/{trip-id}/` 下所有数据：
- `meta.json`（出差元信息，包含 `traveler` 出差人字段）
- `schedule.md`（出发前生成的计划日程）
- `meetings/*.md`（所有会议纪要）

出差人信息（`meta.json` 中的 `traveler`）将自动填入报告的"出差人员"等字段。

### 2. 执行申请 vs 实际对比分析

将 `schedule.md` 中的计划日程与实际会议记录进行逐项对比：

**对比维度：**
- 计划内的会议是否都实际发生
- 实际发生的会议是否有计划外新增
- 会见人员是否与计划一致（或有变动）
- 时间安排是否基本符合

**生成差异清单：**

| 差异类型 | 说明 |
|---------|------|
| ✅ 按计划完成 | 列出符合计划的事项 |
| ⚠️ 有偏差 | 列出有变化的事项（人员变动、时间调整等） |
| ❌ 计划未执行 | 列出计划中未实际发生的事项 |
| ➕ 计划外新增 | 列出临时新增的会议/事项 |

### 3. 生成出差报告 docx

**优先使用模版（⚠️ 有模版时必须走 OOXML 流程，不可跳过）：**
- 用户本次对话提供了模版 → 使用该模版
- `$SKILL_DIR/templates/trip-report.docx` 存在 → 使用该模版
- 无模版 → 使用下方 Markdown 格式生成

#### 有模版时：严格遵循 OOXML 模版填充规范

参见 `meeting-notes` skill 中的完整规范，此处为关键摘要：

**Phase 1：解包并深度解析模版**
```bash
python3 $UNPACK <template.docx> /tmp/tpl_unpacked
# 提取所有占位符
python3 -c "
import re
with open('/tmp/tpl_unpacked/word/document.xml') as f: xml = f.read()
for m in re.findall(r'<w:t[^>]*>([^<]+)</w:t>', xml):
    if m.strip(): print(repr(m))
"
# 检查编码类型
head -c 100 /tmp/tpl_unpacked/word/document.xml
```
必须记录：XML 编码类型、每个占位符的实体编码形式、哪些被拆分为多个 `<w:r>`、每区域的 rPr 格式、表格结构（列数、关键词列/内容列索引）。

**Phase 2：按模版结构填充（不改动 pPr / rPr 格式，只换文字）**
- `encoding="ascii"` 文件：所有插入内容必须用 `to_xml_entities()` 转换
- 用 `replace_t_only()` 保留原 run 格式
- 用 `fill_next_tc()` 填充表格内容列
- 用 `fill_empty_para_after()` 填充标题后的空内容段
- 对每个会议纪要段落，按模版中对应区域逐项填充，不增删模版结构

**Phase 3：强制自检（生成后必须逐项验证，发现问题修复后重新检查）**
1. XML 合法性：`defusedxml.minidom.parse()`，不通过不打包
2. 内容完整性：所有预期字段都已填充（日期、目的地、出差概述、各会议摘要、对比分析、后续行动等）
3. 空单元格扫描：无意外遗漏的内容列
4. occurrence 定位：同名词多次出现时确认指向正确行

**Phase 4：打包**
```bash
python3 $PACK /tmp/output_unpacked <output.docx>
```

#### 无模版时：使用以下 Markdown 格式

```markdown
# 出差报告

**出差ID：** {trip-id}
**目的地：** {城市}
**出差日期：** {开始} 至 {结束}
**报告生成时间：** {时间}

---

## 一、出差概述

{简要说明本次出差背景、目的和整体情况，3-5句话}

## 二、出差日程执行情况

### 计划日程摘要

{引用 schedule.md 中的计划日程}

### 实际执行情况

{按日期列出实际发生的会议/活动}

## 三、会议纪要汇总

{按时间顺序，每次会议一节，包含：主题、参会人、核心结论、Action Items}

### 会议一：{标题}

{摘要}

### 会议二：{标题}

{摘要}

## 四、申请与实际对比分析

{差异清单表格，见步骤2}

**重要提醒：** {如有计划未执行或计划外事项，在此处特别标注，提醒用户在提交报告时说明原因}

## 五、成果与收获

{本次出差达成的主要成果、签订的协议、建立的合作等}

## 六、后续行动计划

{汇总所有会议的 Action Items，统一跟踪}

| 来源会议 | 任务 | 负责人 | 截止时间 |
|---------|------|--------|---------|
| ... | ... | ... | ... |

---
*本报告由 Business Trip Assistant 自动生成*
```

### 4. 保存报告

- Markdown 格式：写入 `$SKILL_DIR/data/{trip-id}/report.md`
- docx 格式：写入 `$SKILL_DIR/data/{trip-id}/report.docx`

更新 `$SKILL_DIR/data/{trip-id}/meta.json`：
- `status`: `"completed"`
- `updatedAt`: 当前时间

### 5. 同步企微文档

调用 `wecom-doc` skill，将报告同步到企微个人空间，文档标题：`{trip-id} - 出差报告`。

### 6. 回复用户

展示报告摘要，**重点突出差异提醒**：

> ✅ 出差报告已生成并同步到企微文档：**{trip-id} - 出差报告**
> 
> ⚠️ **与出差申请不符的内容（需在报告中说明）：**
> - {差异项1}
> - {差异项2}
> 
> 请在提交报告前，补充说明上述差异的原因。

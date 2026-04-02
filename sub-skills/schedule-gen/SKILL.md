---
name: schedule-gen
description: 根据用户的自然语言描述，自动生成详细的出差日程安排。当用户说"帮我生成日程"、"我这次出差要..."、"安排一下日程"时触发。
---

# Schedule Gen — 出差日程生成

## 步骤

### 0. 确定路径

```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
DOCX_SKILL=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills /usr/local/lib -name 'SKILL.md' -path '*/docx/SKILL.md' 2>/dev/null | head -1)")
UNPACK="$DOCX_SKILL/ooxml/scripts/unpack.py"
PACK="$DOCX_SKILL/ooxml/scripts/pack.py"
```

### 1. 确认当前出差

从上下文或用户输入中获取 trip-id，读取 `$SKILL_DIR/data/{trip-id}/meta.json`。

### 2. 收集出差描述

从用户自然语言中提取：
- 需要拜访的客户/团队/人员
- 需要参加的会议/活动
- 需要完成的事项
- 出差天数和城市

若信息不足，主动追问：
- "这次出差几天？"
- "需要见哪些人或参加哪些会议？"

### 3. 生成日程

输出格式（Markdown）：

```markdown
# 出差日程 — {trip-id}

**目的地：** {城市}
**日期：** {开始日期} 至 {结束日期}
**出差目的：** {简要说明}

---

## {日期} 第一天

| 时间 | 事项 | 地点 | 参与人员 | 备注 |
|------|------|------|---------|------|
| 09:00-10:00 | XX会议 | XX大厦会议室 | 张三、李四 | 需提前准备PPT |
| 14:00-16:00 | 客户拜访 | XX公司 | 王五 | |

## {日期} 第二天
...

---

**备注：** {其他注意事项}
```

### 4. 保存日程

将生成的日程写入 `$SKILL_DIR/data/{trip-id}/schedule.md`。

更新 `$SKILL_DIR/data/{trip-id}/meta.json`：
- `plannedSchedule`: `"schedule.md"`
- `status`: `"ongoing"`
- `updatedAt`: 当前时间

### 5. 生成 Word 文档

**优先使用模版（⚠️ 有模版时必须走 OOXML 流程，不可跳过）：**
- 用户本次对话提供了模版 → 使用该模版
- `$SKILL_DIR/templates/trip-schedule.docx` 存在 → 使用该模版
- 无模版 → 使用 Node.js docx 库自由生成

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
必须记录：XML 编码类型、每个占位符的实体编码形式、哪些被拆分为多个 `<w:r>`、每区域的 rPr 格式。

**Phase 2：按模版结构填充（不改动 pPr / rPr 格式，只换文字）**
- `encoding="ascii"` 文件：所有插入内容必须用 `to_xml_entities()` 转换
- 用 `replace_t_only()` 保留原 run 格式
- 用 `fill_next_tc()` 填充表格内容列
- 用 `fill_empty_para_after()` 填充标题后的空内容段

**Phase 3：强制自检（生成后必须逐项验证，发现问题修复后重新检查）**
1. XML 合法性：`defusedxml.minidom.parse()`，不通过不打包
2. 内容完整性：所有预期字段都已填充
3. 空单元格扫描：无意外遗漏的内容列
4. occurrence 定位：同名词多次出现时确认指向正确行

**Phase 4：打包**
```bash
python3 $PACK /tmp/output_unpacked <output.docx>
```

#### 无模版时：使用 Node.js docx 库

输出路径：`/root/.openclaw/{trip-id}-出差日程.docx`

文档结构要求：
- **封面区域**：大标题「出差日程」+ 副标题 trip-id（居中，深蓝色主题）
- **基本信息表**：目的地、日期、出差目的、备注（2列表格，左列浅蓝底色加粗）
- **日程安排表**：按天分节，表头深蓝底白字，列：时间 / 事项 / 地点 / 参与人员 / 备注；行交替底色
- **页眉**：右对齐显示 trip-id + 「出差日程」
- **页脚**：居中显示「第 X 页 共 Y 页」
- 全文字体 Arial，正文 11pt

生成完成后，使用 message 工具将文档发送给用户。

### 6. 同步企微文档

调用 `wecom-doc` skill，将日程内容同步到用户企微个人空间，文档标题：`{trip-id} - 出差日程`。

### 7. 回复用户

展示生成的日程，并提示：
> 日程已保存，Word 文档已发送，并已同步到企微文档。出发后，每次开完会可以发给我会议内容，我帮你生成会议纪要。

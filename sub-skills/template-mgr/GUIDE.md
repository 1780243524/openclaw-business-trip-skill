---
name: template-mgr
description: 管理出差报告和会议纪要的Word模版。当用户说"上传模版"、"查看模版"、"更换模版"、"设置默认模版"时触发。
---

# Template Manager — 模版管理

## 模版目录

模版存储于 skill 根目录下的 `templates/` 目录。

**执行前先确定 SKILL_DIR：**
```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
DOCX_SKILL=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills /usr/local/lib -name 'SKILL.md' -path '*/docx/SKILL.md' 2>/dev/null | head -1)")
UNPACK="$DOCX_SKILL/ooxml/scripts/unpack.py"
```

模版目录：`$SKILL_DIR/templates/`

## 默认模版文件名

| 用途 | 文件名 |
|------|-------|
| 会议纪要 | `meeting-minutes.docx` |
| 出差日程 | `trip-schedule.docx` |
| 出差报告 | `trip-report.docx` |

## 功能

### 查看当前模版

列出 `templates/` 目录下所有 .docx 文件，说明每个模版的用途和最后更新时间。

### 上传/更新模版

用户发送 .docx 文件时：
1. 询问用途（会议纪要 / 出差报告 / 自定义名称）
2. 使用 `docx` skill 读取文件，分析模版结构（字段、格式）
3. 保存到 `templates/` 目录：
   - 会议纪要模版 → 覆盖 `meeting-minutes.docx`（备份旧文件为 `meeting-minutes.bak.docx`）
   - 出差报告模版 → 覆盖 `trip-report.docx`
   - 自定义 → 按用户指定名称保存
4. 总结模版包含的字段/结构，确认已识别

### 模版结构分析（⚠️ 必须深度解析 OOXML）

上传模版后，**必须执行以下 OOXML 深度解析**，结果写入 `templates/{模版名}-structure.md`：

```bash
# 解包模版
python3 /projects/.openclaw/skills/docx/ooxml/scripts/unpack.py <template.docx> /tmp/tpl_inspect

# 提取所有 <w:t> 占位符文字
python3 -c "
import re
with open('/tmp/tpl_inspect/word/document.xml') as f:
    xml = f.read()
matches = re.findall(r'<w:t[^>]*>([^<]+)</w:t>', xml)
for m in matches:
    if m.strip():
        print(repr(m))
"

# 检查 XML 编码
head -c 100 /tmp/tpl_inspect/word/document.xml
```

`{模版名}-structure.md` 必须包含：
- XML 编码类型（ascii / utf-8）
- 所有占位符的原始实体形式
- 被拆分成多个 `<w:r>` 的占位符列表（需用 `replace_split_run_para` 处理）
- 每个填充区域的 rPr 格式（字体名/字号/加粗/颜色）
- 表格结构（列数、关键词列索引、内容列索引）
- 同名词出现多次的情况（需用 `tr_occurrence` 参数区分）

### 删除模版

确认用户意图后删除，不可删除默认模版（`meeting-minutes.docx` / `trip-report.docx`），只能替换。

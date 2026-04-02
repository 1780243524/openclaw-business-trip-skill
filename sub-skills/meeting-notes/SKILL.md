---
name: meeting-notes
description: 根据会议文字记录、音频文件或docx文件，自动生成结构化会议纪要。当用户说"开完会了"、"帮我整理会议纪要"、"记录一下这次会议"，或发送了音频/文档文件时触发。
---

# Meeting Notes — 会议纪要生成

## 输入类型处理

### 类型一：文字描述
用户直接用文字描述会议内容，提取关键信息生成纪要。

### 类型二：音频文件
**优先使用 Whisper 本地转录（自动，无需 API Key）：**

**模型选择策略（固化规则，不随用户描述改变）：**

| 优先级 | 模型 | 适用场景 |
|--------|------|---------|
| 1 | `large-v3` | **默认首选**，效果最佳，中文专业术语识别最准 |
| 2 | `large-v2` | large-v3 不可用时的 fallback |
| 3 | `medium` | 仅在用户明确要求"快速/节省时间"时使用 |
| ❌ | `tiny`/`base`/`small` | 禁止默认使用，准确率不足以用于商业会议纪要 |

1. 若用户发送音频文件路径（mp3/wav/m4a/amr/ogg/flac/mp4等），直接使用 Whisper CLI 转录：
   ```bash
   mkdir -p /tmp/whisper_out
   whisper <audio_file> --model large-v3 --language zh --output_format txt --output_dir /tmp/whisper_out
   ```
   - **始终使用 `--model large-v3`**（最高精度，保证专业术语/人名/机构名识别）
   - **始终使用 `--language zh`**（明确指定中文，避免语言检测错误）
   - 转录完成后读取 `/tmp/whisper_out/<filename>.txt`，按文字描述处理
   - 首次使用时模型需从 HuggingFace 下载（约 3GB），需提前告知用户等待时间
   - 若 HuggingFace 限速（429错误），等待提示的秒数后重试，**不降级模型**

2. 企微语音消息（amr格式）：先转 wav 再转录
   ```bash
   ffmpeg -i <file.amr> /tmp/meeting_audio.wav
   whisper /tmp/meeting_audio.wav --model large-v3 --language zh --output_format txt --output_dir /tmp/whisper_out
   ```
   若 ffmpeg 不可用，提示用户企微的"转为文字"功能后粘贴文字内容

3. 若 whisper 命令不可用（`which whisper` 返回空），提示用户安装：
   > Whisper 未安装，请先告知我帮你安装（brew install openai-whisper），或直接将录音文字粘贴给我

### 类型三：docx 文件
1. 解压读取 XML 内容（`python3 $UNPACK <file> <dir>`）
2. 用正则提取所有 `<w:t>` 文字，拼接成原始文本
3. 提取文字内容后生成/优化纪要

---

## 步骤

### 1. 确认 Trip ID、会议编号和出差人

执行前先确定路径：
```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
DOCX_SKILL=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills /usr/local/lib -name 'SKILL.md' -path '*/docx/SKILL.md' 2>/dev/null | head -1)")
UNPACK="$DOCX_SKILL/ooxml/scripts/unpack.py"
PACK="$DOCX_SKILL/ooxml/scripts/pack.py"
```

- 从上下文获取 trip-id
- 读取 `$SKILL_DIR/data/{trip-id}/meta.json`，统计已有会议数，自动编号（meeting-001、meeting-002...）
- 从 `meta.json` 的 `traveler` 字段获取出差人姓名，用于填充纪要中的"作者"/"我方参会人员"等字段
- 若用户指定了会议名称，在编号后追加（如 `meeting-001-客户拜访`）

### 2. 提取关键信息

从输入内容中提取：
- 会议时间、地点、形式
- 管理人参会人员、我方参会人员
- 会议主题/背景
- 核心讨论内容（按模版字段逐项提取）
- 结论与评级建议
- 后续跟进事项与潜在风险

### 3. 生成会议纪要 docx（优先使用模版）

**模版优先级：**
1. 用户本次对话中提供的模版 → 使用该模版
2. `$SKILL_DIR/templates/meeting-minutes.docx`（若存在）→ 使用该模版
3. 无模版 → 使用 Markdown 格式（见末尾备用格式）

**使用模版时，必须严格遵循以下流程（⚠️ 不可跳过）：**

---

## 🔴 OOXML 模版填充规范（强制执行）

### Phase 1：模版解析（在填充前必须完成）

```bash
# Step 1: 解包模版
python3 $UNPACK <template.docx> /tmp/tpl_unpacked

# Step 2: 提取所有 <w:t> 文字（识别占位符）
python3 -c "
import re
with open('/tmp/tpl_unpacked/word/document.xml') as f:
    xml = f.read()
matches = re.findall(r'<w:t[^>]*>([^<]+)</w:t>', xml)
for m in matches:
    if m.strip():
        print(repr(m))
"

# Step 3: 检查 XML 编码声明
head -c 100 /tmp/tpl_unpacked/word/document.xml
# 如果是 encoding="ascii"，所有插入内容必须使用 XML 数字实体编码
```

**必须记录以下信息：**
- [ ] XML 编码类型（ascii / utf-8）
- [ ] 所有占位符的实体编码形式（如 `&#26085;&#26399;&#65306;` = 日期：）
- [ ] 哪些占位符被拆分为多个 `<w:r>` run（如标题日期"2025年X月XX日"）
- [ ] 每个填充区域的 run 格式（字体/字号/加粗/颜色/pStyle）
- [ ] 表格结构：哪一列是关键词列（tc1），哪一列是内容列（tc2/tc3）

### Phase 2：填充脚本编写规范

```python
import re, shutil
import xml.sax.saxutils as saxutils

# ── 关键常量 ──
FONT = '&#26999;&#20307;_GB2312'  # 楷体_GB2312 实体形式（从模版提取）

def to_xml_entities(s):
    """把非 ASCII 字符转成 XML 数字实体（encoding="ascii" 时必用）"""
    return ''.join(f'&#{ord(c)};' if ord(c) > 127 else c for c in s)

def esc_xml(s):
    """转义 XML 特殊字符 + 非 ASCII 实体化"""
    return to_xml_entities(saxutils.escape(s))

def make_content_run(text, font=FONT, bold=False, color=None, sz=None):
    """生成标准内容 run，严格匹配模版 rPr 格式"""
    b = '<w:b/><w:bCs/>' if bold else ''
    c = f'<w:color w:val="{color}"/>' if color else ''
    s = f'<w:sz w:val="{sz}"/>' if sz else ''
    return (f'<w:r><w:rPr>'
            f'<w:rFonts w:ascii="{font}" w:eastAsia="{font}" w:hint="eastAsia"/>'
            f'{b}{c}{s}'
            f'</w:rPr>'
            f'<w:t xml:space="preserve">{esc_xml(text)}</w:t>'
            f'</w:r>')

def replace_t_only(xml, old_entity, new_text):
    """
    保留原 run 的 rPr（字体/加粗/颜色），只替换 <w:t> 内的文字。
    适用于：占位符文字完整在单个 <w:t> 内的情况。
    """
    tag = f'<w:t>{old_entity}</w:t>'
    if tag not in xml:
        tag = f'<w:t xml:space="preserve">{old_entity}</w:t>'
    if tag not in xml:
        print(f'✗ t未找到: {old_entity[:30]}')
        return xml
    xml = xml.replace(tag, f'<w:t xml:space="preserve">{esc_xml(new_text)}</w:t>', 1)
    print(f'✓ t: {old_entity[:30]}')
    return xml

def replace_split_run_para(xml, anchor_entity, new_text, occurrence=1):
    """
    处理占位符被拆分成多个 <w:r> 的情况（如标题日期）。
    找到包含 anchor 的段落，保留 <w:pPr>，替换所有 runs 为单个新 run。
    new_text 需提供完整文字（如 '2026年3月19日'）。
    注意：new_run 的 rPr 需手动按模版格式指定。
    """
    count = 0
    pos = 0
    while True:
        idx = xml.find(anchor_entity, pos)
        if idx < 0:
            print(f'✗ split_run未找到: {anchor_entity[:30]}')
            return xml
        count += 1
        if count == occurrence:
            p_start = xml.rfind('<w:p ', 0, idx)
            p_end = xml.find('</w:p>', idx) + len('</w:p>')
            orig = xml[p_start:p_end]
            p_tag = re.match(r'<w:p\b[^>]*>', orig).group(0)
            ppr_m = re.search(r'<w:pPr>.*?</w:pPr>', orig, re.DOTALL)
            ppr = ppr_m.group(0) if ppr_m else ''
            new_run = make_content_run(new_text)  # 调用时按需调整 rPr 参数
            xml = xml[:p_start] + p_tag + ppr + new_run + '</w:p>' + xml[p_end:]
            print(f'✓ split_run: {anchor_entity[:30]}')
            return xml
        pos = idx + 1
    return xml

def fill_empty_para_after(xml, anchor_entity, content):
    """
    在 anchor 所在段落之后的第一个空段落（无 <w:r>）中插入内容。
    适用于：标题行后面跟着一个空段落作为内容区的场景。
    """
    idx = xml.find(anchor_entity)
    if idx < 0:
        print(f'✗ empty_para未找到: {anchor_entity[:30]}')
        return xml
    p_end = xml.find('</w:p>', idx) + len('</w:p>')
    next_p_start = xml.find('<w:p ', p_end)
    next_p_end = xml.find('</w:p>', next_p_start) + len('</w:p>')
    next_para = xml[next_p_start:next_p_end]
    if '<w:r>' not in next_para and '<w:r ' not in next_para:
        p_tag = re.match(r'<w:p\b[^>]*>', next_para).group(0)
        ppr_m = re.search(r'<w:pPr>.*?</w:pPr>', next_para, re.DOTALL)
        ppr = ppr_m.group(0) if ppr_m else ''
        new_para = p_tag + ppr + make_content_run(content) + '</w:p>'
        xml = xml[:next_p_start] + new_para + xml[next_p_end:]
        print(f'✓ empty_para: {anchor_entity[:30]}')
    else:
        print(f'- empty_para已有内容: {anchor_entity[:30]}')
    return xml

def fill_next_tc(xml, anchor_entity, content, tr_occurrence=1):
    """
    在表格行中，找到包含 anchor 的单元格（tc1），填充其后第一个空单元格（tc2）。
    适用于：两列表格，关键词列 + 内容列。
    tr_occurrence：anchor 第几次出现对应的行（用于同名词出现多次时区分）。
    """
    count = 0
    pos = 0
    while True:
        idx = xml.find(anchor_entity, pos)
        if idx < 0:
            print(f'✗ tc未找到: {anchor_entity[:30]}')
            return xml
        count += 1
        if count == tr_occurrence:
            tr_start = xml.rfind('<w:tr ', 0, idx)
            tr_end = xml.find('</w:tr>', idx) + len('</w:tr>')
            tr_xml = xml[tr_start:tr_end]
            ain = idx - tr_start
            tc_end = tr_xml.find('</w:tc>', ain) + len('</w:tc>')
            rest = tr_xml[tc_end:]
            ntc_end = rest.find('</w:tc>') + len('</w:tc>')
            tc = rest[:ntc_end]
            p_m = re.search(r'(<w:p\b[^>]*>)(.*?)(</w:p>)', tc, re.DOTALL)
            if p_m:
                inner = p_m.group(2)
                has_run = '<w:r>' in inner or '<w:r ' in inner
                if not has_run:
                    ppr_end = inner.find('</w:pPr>')
                    ins = ppr_end + len('</w:pPr>') if ppr_end >= 0 else len(inner)
                    new_inner = inner[:ins] + make_content_run(content) + inner[ins:]
                    new_p = p_m.group(1) + new_inner + p_m.group(3)
                    new_tc = tc[:p_m.start()] + new_p + tc[p_m.end():]
                    new_tr = tr_xml[:tc_end] + new_tc + rest[ntc_end:]
                    xml = xml[:tr_start] + new_tr + xml[tr_end:]
                    print(f'✓ tc: {anchor_entity[:30]}')
                else:
                    print(f'- tc已有内容: {anchor_entity[:30]}')
            return xml
        pos = idx + 1
    return xml
```

### Phase 3：⚠️ 自检清单（生成脚本后强制执行）

脚本生成后，**必须**执行以下检查，发现问题立即修复，修复后重新检查，直到全部通过：

#### 3.1 XML 合法性检查（最重要）
```python
import defusedxml.minidom
try:
    with open('/tmp/output_unpacked/word/document.xml', 'rb') as f:
        defusedxml.minidom.parse(f)
    print('✓ XML有效')
except Exception as e:
    print(f'✗ XML错误: {e}')
    # 找出错行：
    content = open('/tmp/output_unpacked/word/document.xml').read()
    lines = content.split('\n')
    # 检查 encoding="ascii" 文件是否有非 ASCII 字符直接插入
    for i, line in enumerate(lines):
        for j, c in enumerate(line):
            if ord(c) > 127:
                print(f'  非ASCII字符 行{i+1} 列{j+1}: U+{ord(c):04X} {repr(c)}')
```

**常见错误：**
- `encoding="ascii"` 文件直接插入中文字符 → 必须用 `to_xml_entities()` 转换
- `<w:pPr>` 内部的 `<w:rFonts>` 被当作 `<w:r>` 的起始位置 → 用 `</w:pPr>` 定位插入点
- 段落标签（`<w:p>`）的开头被错误截断 → 检查 `p_tag + ppr + run + </w:p>` 是否完整

#### 3.2 内容完整性检查
```python
# 验证所有预期字段都已填充
with open('/tmp/output_unpacked/word/document.xml') as f:
    xml = f.read()

check_fields = [
    '2026',           # 日期已填
    '合骥基金',        # 机构名已填（或其他管理人）
    '会议形式',        # 基本信息
    # ... 按模版字段逐一添加
]
for field in check_fields:
    encoded = ''.join(f'&#{ord(c)};' if ord(c) > 127 else c for c in field)
    if encoded in xml or field in xml:
        print(f'✓ {field}')
    else:
        print(f'✗ 缺失: {field}')
```

#### 3.3 表格空单元格检查
```python
# 找出所有仍然为空的表格内容列（表示填充遗漏）
import re
tcs = re.findall(r'<w:tc>(.*?)</w:tc>', xml, re.DOTALL)
empty_count = 0
for i, tc in enumerate(tcs):
    p_m = re.search(r'<w:p\b[^>]*>(.*?)</w:p>', tc, re.DOTALL)
    if p_m:
        inner = p_m.group(1)
        if '<w:r>' not in inner and '<w:r ' not in inner:
            empty_count += 1
            # 找到对应的行关键词
print(f'空单元格数量: {empty_count}（模版中合理的空单元格除外）')
```

#### 3.4 tr_occurrence 定位检查
对于同一个 anchor 词在文件中出现多次的情况（如"规模"、"策略"），必须：
1. 先用 Python 查找所有出现位置：`positions = [i for i in range(len(xml)) if xml[i:].startswith(anchor)]`
2. 逐个检查每个位置所在行的 tc1 内容
3. 确认 `tr_occurrence` 数字指向正确的行

### Phase 4：打包与发送

```bash
# 打包
python3 $PACK /tmp/output_unpacked <output.docx>

# 发送
message send --channel openclaw-wecom-bot --target <group_id> --media <output.docx>
```

---

## 文件命名规则

```
ReviewMRDiscovery_{管理人名字}_{策略}_{日期%Y%m%d}.docx
```

---

## 备用格式（无模版时）

```markdown
# 会议纪要

**会议编号：** {meeting-id}
**所属出差：** {trip-id}
**会议时间：** {日期} {时间}
**会议地点：** {地点}
**参会人员：** {人员列表}
**会议主题：** {主题}

---

## 会议背景

{背景说明}

## 主要内容

{分项列出讨论内容}

## 决议事项

| 序号 | 决议内容 | 责任人 | 截止日期 |
|------|---------|--------|---------|
| 1 | ... | ... | ... |

## Action Items

| 序号 | 任务 | 负责人 | 截止时间 | 状态 |
|------|------|--------|---------|------|
| 1 | ... | ... | ... | 待处理 |

## 遗留问题

{如无则填"无"}

---
*会议纪要生成时间：{生成时间}*
```

---

## 保存与归档

- docx 格式：写入 `$SKILL_DIR/data/{trip-id}/meetings/{meeting-id}.docx`
- 更新 `$SKILL_DIR/data/{trip-id}/meta.json`

## 回复用户

展示生成的纪要摘要，提示后续可继续记录下一次会议，或发送"出差结束"生成最终报告。

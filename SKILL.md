---
title: "PDF文档转Skill技能"
version: "3.1.0"
description: "将专业PDF文档（标准/指引/规范/手册）转化为可复用的Skill，实现文档知识的结构化提取、资源文件编写、质量验证与打包分发"
tags: ["PDF转化", "Skill开发", "文档提取", "质量验证", "知识工程"]
---

# PDF文档转Skill技能 v3.1.0

将专业PDF文档转化为结构化、可复用、可分发的Skill。

---

## 适用场景

**适用：** 用户要求将PDF"转成技能/做成Skill/可复用"，文档≥20页且含表格/规则/参数

| 适用文档类型 | 不适用文档类型 |
|------------|-------------|
| 工程造价指引、技术标准规范、操作手册/SOP | 小说/报告/演示文稿（无结构化数据） |
| 法律法规汇编、产品参数手册、财务测算模板 | 纯叙述性文档（无规则/参数） |

---

## 核心原则

1. **数据硬编码** — 指标/参数写入资源文件，AI只调用，不凭记忆补数据
2. **来源可溯源** — 每条数据标注 `资源文件名 + <!-- PDF第XX页 -->`
3. **计算透明** — 展示公式与中间结果，可手工复算
4. **校验内嵌** — 互斥/优先/边界/约束规则写入SKILL.md
5. **相对路径** — 分发版资源路径统一用 `resources/`
6. **渐进交付** — P0最小可用→确认→P1核心→P2扩展

---

## 快速决策树

```
收到"将PDF转成Skill"请求
│
├── 无PDF路径？ → 询问路径 + 目标技能名
│
├── 无技能目录？ → 创建 ~/.workbuddy/skills/<技能名>/
│
├── 文档规模？
│   ├── <50页 → 单Agent串行
│   ├── 50~150页 → 2~4 Agent并行
│   └── >150页 → 必须拆分章节，多Agent并行
│
└── 文档类型？ → 影响资源拆分策略（见§适配指南）
```

---

## 工作流（7个阶段）

> **Python脚本通用约定：** 以下脚本在Windows中文环境下，需在脚本开头加：
> ```python
> import sys, io
> sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
> ```
> 后文脚本中不再重复标注。

---

### 阶段1：文档分析（1轮）

**目标：** 理解文档结构，设计Skill架构，输出方案待用户确认

#### 1.1 PDF双通道提取

```python
# 通道1: fitz — 全文提取（适合文本段落）
import fitz
doc = fitz.open('文档.pdf')
with open('pdf_text_fitz.txt', 'w', encoding='utf-8') as f:
    for i, page in enumerate(doc):
        f.write(f'\n\n===== 第{i+1}页 =====\n')
        f.write(page.get_text())
doc.close()
print(f'总页数: {len(doc)}')
```

```python
# 通道2: pdfplumber — 结构化表格提取
import pdfplumber, json
tables_all = []
with pdfplumber.open('文档.pdf') as pdf:
    for i, page in enumerate(pdf.pages):
        for j, t in enumerate(page.extract_tables()):
            tables_all.append({'page': i+1, 'table_idx': j, 'data': t})
with open('pdf_tables.json', 'w', encoding='utf-8') as f:
    json.dump(tables_all, f, ensure_ascii=False, indent=2)
print(f'提取表格共 {len(tables_all)} 个')
```

**工具盲区（必读）：**

| 工具 | 盲区 | 补救 |
|------|------|------|
| fitz | 跨行截断（数字拆成两行） | pdfplumber补充 |
| pdfplumber | 跨页截断（表格只提取当页） | fitz全文补充 |
| 两者共有 | 合并单元格丢失 | 对照PDF手工标注继承关系 |

#### 1.2 文档结构梳理

| 层级 | 识别内容 | 归属 |
|------|---------|------|
| 数据层 | 含数值指标/参数的章节 | 资源文件核心 |
| 规则层 | 计算/校验/适用条件 | SKILL.md校验规则区 |
| 模板层 | 输出格式/表格样式 | 02-模板.md |
| 说明层 | 术语定义/背景 | 可选提取 |

#### 1.3 资源文件架构设计

**拆分原则：** 独立自包含 | 单文件30~100KB（>150KB必拆） | P0→P1→P2优先级

**命名约定：** `序号-主题.md`（序号按调用频率排列）

| 文件 | 内容 | 优先级 |
|------|------|--------|
| 01-计量规则.md | 计算规则、系数、适用条件 | P0 |
| 02-模板.md | 输出格式、样表 | P0 |
| 03-通用指标.md | 所有项目通用的数据/参数 | P1 |
| 04~0N-专项X.md | 各专项数据 | P1 |
| 0X-快速模式.md | 综合指标/匡算表 | P1 |
| 0Y-使用说明与案例.md | 案例+FAQ+溯源 | P2 |

#### 1.4 交付与确认

输出《文档分析报告与Skill方案.md》，包含：文档规模、章节树形图、资源文件拆分方案（名/内容/对应页/大小/优先级）、工作模式流程设计、复杂问题清单。

> ⚠️ **输出方案后等待用户确认，不要直接开始提取。**

**交付物：** `pdf_text_fitz.txt` + `pdf_tables.json` + 《文档分析报告与Skill方案.md》

---

### 阶段2：资源文件创建（2-3轮）

**目标：** 从PDF提取内容，编写结构化资源文件

#### 2.1 按页码导出原始数据

```python
def extract_pages(pdf_path, start, end, output):
    """提取指定页码范围（1-indexed）到文本文件"""
    import fitz
    doc = fitz.open(pdf_path)
    with open(output, 'w', encoding='utf-8') as f:
        for i in range(start-1, min(end, len(doc))):
            f.write(f'\n\n===== 第{i+1}页 =====\n')
            f.write(doc[i].get_text())
    doc.close()

extract_pages('文档.pdf', 29, 84, 'raw_03_通用指标.txt')
```

#### 2.2 格式规范（每个文件必须遵守）

**文件头（必须）：**
```markdown
# XX-文件名.md
**来源：** 《文档名》（机构/日期）
**对应PDF：** 第XX-XX页 | **提取日期：** YYYY-MM-DD
**说明：** 本文件包含XXX，共X大类指标
```

**内容规范：**

| 规范项 | 要求 | 示例 |
|--------|------|------|
| 区间分隔符 | 统一用 `~` | `60~110 元/m³` |
| 页码标注 | 每个表格/章节前 | `<!-- PDF第29页 -->` |
| 计量单位 | 必须保留 | `元/m²`、`万元/床` |
| 合并单元格 | 标注继承关系 | `（同上，适用A/B/C）` |
| 调整系数 | 完整保留公式 | `运距每增减1km：±2.5元/m³` |
| 注释说明 | 保留原文注释 | `注：XXX时增加XX元/m²` |
| 大类分隔 | `---` | 一级章节之间 |

#### 2.3 并行策略

| 规模 | 策略 |
|------|------|
| ≤3文件且均≤50KB | 单Agent串行 |
| 4~6文件或含>100KB文件 | 2~4 Agent并行 |
| >6文件 | 分批并行，每批≤4 Agent |

**并行时主Agent负责：** 分配任务 → 统一格式规范 → 完成后统一校验一致性

#### 2.4 分阶段交付

```
P0（最小可用）→ 01+02+核心1个数据文件 → 等用户确认方向
P1（核心功能）→ 所有专项+快速模式文件 → 可使用基本功能
P2（完善）    → 使用说明/案例/扩展专项 → 可选
```

**交付物：** 所有资源文件 + raw原始数据文件（不纳入分发包）

---

### 阶段3：SKILL.md编写（1轮）

**目标：** 编写Skill主文件，定义工作流与校验规则

#### 3.1 必须包含的要素

| 要素 | 要求 |
|------|------|
| frontmatter | title / version / description / author / agent_created / tags |
| 适用范围 | 地区/阶段/类型/依据/有效期 |
| 资源目录表 | 文件名/内容/对应PDF页/状态 |
| 工作模式 | 每种模式的完整步骤，每步引用具体资源文件 |
| 调用优先级 | 专项 > 通用（防重复/遗漏） |
| 校验规则 | 文档中所有互斥/约束/边界条件（编号列出） |
| 输出格式 | 含数据来源标注的模板示例 |
| 特别说明 | 超范围提示/不适用情形 |
| 版本记录 | 版本号/时间/变更内容 |

#### 3.2 校验规则编写规范

每条规则必须包含**触发条件**+**处理方式**：

```markdown
**规则N：[规则名称]**
- 触发条件：[什么情况下检查]
- 检查内容：[检查什么]
- 处理方式：[冲突时如何处理]
- 数据来源：[资源文件+页码]
```

#### 3.3 资源路径规范

- 开发版：绝对路径 `D:\...\resources\01-计量规则.md`
- 分发版：相对路径 `resources/01-计量规则.md`
- 打包前必须修改

**交付物：** `SKILL.md`（本地版，绝对路径）

---

### 阶段4：质量验证（2轮）

**目标：** 确保数据准确性 + 功能可用性

#### 4.1 第一轮：数据准确性

**4.1.1 PDF原文抽样验证（≥10项）**

```python
import fitz
doc = fitz.open('文档.pdf')

# 格式：(描述, PDF页0-indexed, 关键词列表, 'all'全匹配/'any'任意匹配)
checks = [
    ('通用指标-土石方', 28, ['60', '110'], 'all'),
    ('已修正-挡土墙', 289, ['2000', '2300'], 'all'),
    # 按需添加
]

for name, page_idx, keywords, mode in checks:
    text = doc[page_idx].get_text()
    ok = all(kw in text for kw in keywords) if mode == 'all' else any(kw in text for kw in keywords)
    print(f'[{"PASS" if ok else "FAIL"}] {name}')
doc.close()
```

> ⚠️ fitz跨行截断会导致数字搜索FAIL，需人工检查PDF原文区分"工具限制"与"数据错误"。

**4.1.2 格式一致性验证**

```python
import re
for f in ['03-通用指标.md', '04-专项A.md', '05-专项B.md']:
    with open(f, encoding='utf-8') as fp:
        matches = re.findall(r'(?<=[0-9])-(?=[0-9])', fp.read())
    print(f'{f}: 残留-分隔符 {len(matches)} 处')
```

**4.1.3 跨文件数据一致性**

```python
shared = [('土石方60~110', '60~110'), ('围蔽费800~950', '800~950')]  # 按实际填写
files = ['03-通用.md', '04-专项A.md', '05-专项B.md']
for name, pattern in shared:
    counts = {f: open(f, encoding='utf-8').read().count(pattern) for f in files}
    ok = all(v > 0 for v in counts.values())
    print(f'[{"PASS" if ok else "FAIL"}] {name}: {counts}')
```

**4.1.4 结构覆盖度**

```python
import os
coverage = {'PDF P1-28': '01-计量规则.md', 'PDF P29-84': '03-通用指标.md'}  # 按实际填写
for chapter, res_file in coverage.items():
    print(f'[{"PASS" if os.path.exists(res_file) else "FAIL"}] {chapter} → {res_file}')
```

**交付物：** `质量验证报告.md`

---

#### 4.2 第二轮：功能可用性（8维度）

| 维度 | 内容 | 最低项数 |
|------|------|---------|
| T1 Skill可用性 | 元数据完整、资源全部可达 | 资源文件数+3 |
| T2 文件完整性 | 行数/大小、格式、页码标注 | 6项 |
| T3 数据抽样 | PDF原文抽样（复用第一轮+新增） | ≥10项 |
| T4 跨文件一致性 | 共享指标数值相同 | 共享指标数 |
| T5 端到端流程 | 每种模式完整走一遍 | 每模式≥3项 |
| T6 校验规则 | 每条规则能正确触发 | 每规则1项 |
| T7 边界条件 | 超范围/缺参数/异常输入 | ≥5项 |
| T8 计算验证 | 案例计算可手工复算 | 每案例≥3项 |

**计算验证模板（容差0.5%）：**

```python
cases = [
    # (描述, 计算值, 期望值, 容差)
    ('医院估算', 80000 * 7590 / 1e8, 6.07, 0.05),
]
for name, calc, expected, tol in cases:
    print(f'[{"PASS" if abs(calc-expected)<=tol else "FAIL"}] {name}: {calc:.2f} vs {expected}')
```

**交付物：** `测试验证报告.md`

---

### 阶段5：使用说明与案例（1轮）

**目标：** 编写用户友好的使用说明

#### 5.1 文件结构

```markdown
# XX-使用说明与案例.md

## 1. 技能简介（一句话适用范围 + 核心能力 + 数据来源）
## 2. 使用方法（每模式：适用场景 + 所需参数 + 输出内容 + 注意事项）
## 3. 典型案例（≥3个，见下方规范）
## 4. FAQ
## 5. 数据溯源说明
```

#### 5.2 案例编写规范

- ≥3个案例，覆盖：快速模式×1 + 详细模式×1 + 校验规则触发×1
- 每个计算项标注数据来源（资源文件+表格）
- 所有乘法/求和用Python验证后写入
- 至少1个案例涉及跨文件调用

**交付物：** 使用说明资源文件 + SKILL.md版本更新

---

### 阶段6：打包前检查复核 ⭐

**目标：** 分发前系统检查完整性+准确性+质量，所有问题必须修复后才可打包

#### 6.1 完整性分析

**三步验证：**

**步骤1：覆盖矩阵** — 确认每个PDF章节有对应资源文件

```python
import fitz, os
doc = fitz.open('文档.pdf')
chapters = {  # 根据阶段1分析结果填写
    '计量规则+模板': (1, 28),
    '通用指标': (29, 84),
    '专项A': (85, 154),
}
mapping = {  # 章节到资源文件的映射
    '计量规则+模板': ['01-计量规则.md', '02-模板.md'],
    '通用指标': ['03-通用指标.md'],
    '专项A': ['04-专项A.md'],
}
resource_dir = r'D:\...\resources'  # 按实际路径修改

for ch, (start, end) in chapters.items():
    chars = sum(len(doc[i].get_text()) for i in range(start-1, min(end, len(doc))))
    files = mapping.get(ch, [])
    status = []
    for f in files:
        p = os.path.join(resource_dir, f)
        if os.path.exists(p):
            status.append(f'{f}({os.path.getsize(p)//1024}KB)')
        else:
            status.append(f'❌{f}不存在')
    print(f'{ch} P{start}-P{end}({end-start+1}页/{chars}字) → {", ".join(status)}')
doc.close()
```

**步骤2：页码标注统计** — 确认资源文件页码范围与方案一致

```python
import re, os
for fname in sorted(os.listdir(resource_dir)):
    if not fname.endswith('.md'): continue
    content = open(os.path.join(resource_dir, fname), encoding='utf-8').read()
    pages = sorted(set(int(p) for p in re.findall(r'PDF第(\d+)页', content)))
    tbl_rows = len([l for l in content.split('\n') if l.strip().startswith('|') and '---' not in l])
    indicators = len([l for l in content.split('\n') if '~' in l])
    rng = f'P{pages[0]}-P{pages[-1]}' if pages else '无标注'
    print(f'{fname:<35} 页码:{rng:<12} 表格行:{tbl_rows:>4} 指标行:{indicators:>4}')
```

**步骤3：章节遗漏检测** — PDF章节标题是否在资源文件中出现

```python
import re, os
chapters_in_pdf = [l.strip()[:50] for l in open('pdf_text_fitz.txt', encoding='utf-8')
                   if re.match(r'^[一二三四五六七八九十]+[、．.]\s*\S', l.strip())]
all_content = ''.join(open(os.path.join(resource_dir, f), encoding='utf-8').read()
                      for f in os.listdir(resource_dir) if f.endswith('.md'))
missing = [ch for ch in chapters_in_pdf if ch[3:13].strip() and ch[3:13].strip() not in all_content]
print(f'⚠️ 可能遗漏{len(missing)}个章节' if missing else '✅ 章节全覆盖')
for ch in missing: print(f'  - {ch}')
```

**结论要求：** 列出所有"PDF有但资源文件无"的内容，区分「刻意不提取」vs「意外遗漏」，意外遗漏必须补充。

---

#### 6.2 准确性分析

**三级抽样：**

| 级别 | 对象 | 数量 | 方法 |
|------|------|------|------|
| L1 | 已修正项 | 全部 | 双向验证：资源文件含正确值且无旧错误值 + PDF原文确认 |
| L2 | 综合指标/专项极值 | ≥3项/文件 | Python脚本+PDF原文 |
| L3 | 通用指标常规抽样 | ≥5项/文件 | Python脚本 |

**L1 双向验证脚本：**

```python
import fitz, os
doc = fitz.open('文档.pdf')
resource_dir = r'D:\...\resources'  # 按实际路径修改

corrections = [
    # (描述, PDF页0-indexed, 旧错误值, 正确值, 资源文件, PDF确认词)
    ('挡土墙', 289, '1100~1200', '2000~2300', '06-住房.md', ['2000','2300']),
    # 从MEMORY.md修正记录填写
]

for desc, pg, old, new, res_file, kws in corrections:
    res = open(os.path.join(resource_dir, res_file), encoding='utf-8').read()
    has_new, has_old = new in res, old in res
    pdf_ok = all(kw in doc[pg].get_text() for kw in kws)
    status = 'PASS' if (has_new and not has_old) else 'FAIL'
    print(f'[{status}] {desc}: 正确值={has_new} 旧值残留={has_old} PDF确认={pdf_ok}')
doc.close()
```

**L2/L3 脚本：** 复用阶段4.1.1的验证框架，按文件分组输出即可。

**FAIL项判断规则：**

| FAIL原因 | 判断方式 | 处理 |
|---------|---------|------|
| fitz跨行截断 | 人工查看PDF，数字存在 | 标注"工具限制"，不改资源文件 |
| 实际数据错误 | 人工查看PDF，数字不存在/不符 | 必须修正，记录对照表 |
| 页码偏差 | 关键词在±2页内找到 | 更新验证脚本参数 |
| 表述差异 | PDF用汉字数字，资源文件用阿拉伯数字 | 验证等价性，不改 |

---

#### 6.3 问题修复流程

**第一步：分类汇总**

| 编号 | 类型 | 文件 | 描述 | 严重度 | 状态 |
|------|------|------|------|--------|------|
| P001 | 数据错误 | 04-医院.md | 挡土墙1100~1200应为2000~2300 | 高 | 待修复 |

- **高**：数值错误，直接导致用户结果偏差（必须修复才能打包）
- **中**：内容遗漏，影响功能完整性（必须修复才能打包）
- **低**：格式/可读性（不修复需在分发说明中注明）

**第二步：修复并记录** — 每处修复追加到 MEMORY.md：

```markdown
| 日期 | 文件 | 原内容 | 修正后 | PDF依据 | 原因 |
```

**第三步：修复后重新验证** — 重跑6.1+6.2脚本，直到：完整性无意外遗漏 + 准确性FAIL项已分类 + L1双向验证全通过

---

#### 6.4 打包前核查清单

**完整性（缺一不可）：**
- [ ] PDF核心章节均有对应资源文件
- [ ] 资源文件头部页码范围与实际内容一致
- [ ] SKILL.md资源目录表与实际文件数一致
- [ ] 快速/详细模式所需文件全部完成

**准确性（缺一不可）：**
- [ ] L1修正项双向验证全通过
- [ ] L2通过率≥95%（FAIL项已分类为工具限制）
- [ ] L3通过率≥90%（FAIL项已分类为工具限制）
- [ ] 跨文件共享指标数值完全一致
- [ ] 分隔符已全部统一为 `~`

**格式规范：**
- [ ] 所有资源文件有标准头部
- [ ] 数值单位已标注
- [ ] 合并单元格继承关系已标注
- [ ] 修正记录已在MEMORY.md更新

**SKILL.md：**
- [ ] 版本号已更新
- [ ] 校验规则均有触发条件+处理方式
- [ ] 特别说明/不适用情形已明确

**交付物：** 《打包前检查复核报告.md》（含覆盖矩阵+验证结果+问题汇总+核查清单+打包建议）

---

#### 6.5 技能质量评估与修复 ⭐

**目标：** 从"数据正确"迈向"技能好用"——检查Skill设计合理性，而非数据对不对

> 6.1~4检查"数据对不对、内容全不全"，6.5检查"好不好用、设计合不合理"。

**第一步：6维度打分（1~5分）**

| 维度 | 评估重点 | 5分标准 |
|------|---------|--------|
| D1 适用性 | 场景覆盖是否全面 | 适用/不适用/文档类型/规模限制均有清晰定义 |
| D2 可操作性 | 步骤是否可执行、脚本是否完整可运行 | 每步有输入→处理→输出，脚本可直接复制运行 |
| D3 校验完整性 | 规则是否覆盖关键风险 | 每条规则有触发条件+处理方式完整闭环 |
| D4 准确性 | 工具/正则/脚本/经验是否正确 | 无工具名错误/正则误判/过时API |
| D5 复用性 | 换同类文档能否直接套用 | 适配指南含资源拆分+特殊处理+校验规则 |
| D6 一致性 | 术语/格式/命名/路径是否统一 | 无同一概念不同术语，版本号一致 |

**评分输出：**

```markdown
| 维度 | 得分 | 扣分项 |
|------|------|--------|
| D1 | _/5 | ... |
| D2~D6 | ... | ... |
| **加权平均** | **_/5** | — |
```

**第二步：逐维度深度检查（24项）**

**D1 适用性：** ①≥3种子类型 ②排除条件明确 ③决策树覆盖大/中/小文档 ④不适用文档回复指引

**D2 可操作性：** ①每阶段输入→处理→输出 ②脚本含完整import+参数+异常处理 ③路径占位符有注释 ④正则有测试用例 ⑤脚本失败有回退方案

**D3 校验完整性：** ①每条业务规则有对应校验 ②校验含触发+检查+处理+提示 ③规则间优先级/互斥说明 ④规则冲突仲裁逻辑

**D4 准确性：** ①工具名拼写正确 ②正则精确（无误匹配/漏匹配） ③经验教训与MEMORY.md一致 ④硬编码信息为当前值 ⑤API为当前版本

**D5 复用性：** ①换同类文档可直接执行 ②适配指南三列齐全 ③结构差异调整指引 ④经验教训为通用原则

**D6 一致性：** ①version与版本记录一致 ②tags与内容匹配 ③术语统一 ④路径格式统一 ⑤章节编号连续

**第三步：按优先级修复**

| 优先级 | 维度 | 要求 | 期限 |
|--------|------|------|------|
| P0 | D4 准确性 | 修复后重跑脚本验证 | 当轮 |
| P1 | D2 可操作性 / D3 校验完整性 | 补全脚本/补写规则 | 当轮 |
| P2 | D1 适用性 / D5 复用性 | 补充场景/适配指南 | 当轮 |
| P3 | D6 一致性 | 统一术语/格式 | 可延后 |

**第四步：复评终止条件**

- D4 = 5 | D2 ≥ 4 | D3 ≥ 4 | D1/D5/D6 ≥ 3 | **加权平均 ≥ 4.0**

未达标维度继续修复，直到满足。

**6.5交付物：** 《Skill质量评估报告.md》+ P0/P1全部修复 + 复评≥4.0

**阶段6总交付物：**
- [ ] 《打包前检查复核报告.md》（6.1~4）
- [ ] 《Skill质量评估报告.md》（6.5）
- [ ] 高/中问题全部修复
- [ ] 质量复评加权平均 ≥ 4.0
- [ ] MEMORY.md修正记录已更新

---

### 阶段7：打包分发（1轮）

**目标：** 生成SkillHub可上传的分发包

```python
import shutil, zipfile, os, re

# ====== 按实际路径修改 ======
skill_name = "技能名称"
src_skill_md = r"C:\Users\...\skills\技能名\SKILL.md"
src_resources = r"D:\...\resources"
output_zip = f"D:\\...\\{skill_name}-SkillHub.zip"
# ============================

# Step1: 绝对路径→相对路径
content = open(src_skill_md, encoding='utf-8').read()
content = re.sub(r'D:\\[^`\n]*resources\\', 'resources/', content)

# Step2: 创建临时目录
tmp = f"D:\\temp\\{skill_name}-SkillHub"
os.makedirs(f"{tmp}/resources", exist_ok=True)

# Step3: 写入分发版SKILL.md
open(f"{tmp}/SKILL.md", 'w', encoding='utf-8').write(content)

# Step4: 复制资源文件（排除过程文件）
exclude = {'测试验证报告.md', '质量验证报告.md', '文档分析报告与Skill方案.md'}
for f in os.listdir(src_resources):
    if f.endswith('.md') and f not in exclude:
        shutil.copy(os.path.join(src_resources, f), os.path.join(tmp, 'resources', f))

# Step5: 打包
os.makedirs(os.path.dirname(output_zip), exist_ok=True)
with zipfile.ZipFile(output_zip, 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.write(f"{tmp}/SKILL.md", "SKILL.md")
    for f in os.listdir(f"{tmp}/resources"):
        zf.write(f"{tmp}/resources/{f}", f"resources/{f}")

# Step6: 验证（用Python zipfile，不用PowerShell—中文文件名乱码）
with zipfile.ZipFile(output_zip) as zf:
    for info in zf.infolist():
        print(f'  {info.filename:<55} {info.file_size/1024:.1f} KB')
    print(f'文件数: {len(zf.namelist())} | 大小: {os.path.getsize(output_zip)/1024:.1f} KB')

# Step7: 清理
shutil.rmtree(tmp)
print(f'✅ 打包完成: {output_zip}')
```

**不可纳入分发包：** `raw_XX_*.txt` / `pdf_text_fitz.txt` / `pdf_tables.json` / 各类报告.md

**交付物：** `{技能名}-SkillHub.zip`

---

## 常见PDF提取错误速查表

| 错误类型 | 表现 | 检测 | 修复 |
|---------|------|------|------|
| OCR数字错误 | 数值超合理范围（12060 vs 1260） | 范围合理性判断 | 对照PDF修正，记录对照表 |
| fitz跨行截断 | 数字拆两行（8~\\n15） | `[digit]\\n[digit]` 模式 | pdfplumber补充；不影响资源文件则标注工具限制 |
| pdfplumber跨页截断 | 表格只提取当页 | 与fitz全文对照 | fitz提取+手工补全 |
| 合并单元格丢失 | 多行/列共用值缺失 | 检查空值行/列 | 标注继承关系（"同上"/"适用A/B/C"） |
| 分隔符混用 | `-` 和 `~` 混用 | `grep '[0-9]-[0-9]'` | 正则批量替换（见下方脚本） |
| 单位截断 | 数值与单位分离 | 数值后无单位 | 对照PDF修复 |
| 负数与范围混淆 | -5 被误解为范围 | 上下文判断 | 加括号或说明（"增减±5元"） |

---

## 格式一致性自动修复脚本

```python
import re, os

def fix_separators(file_path):
    """将价格范围中的-替换为~，保留章节号/页码/比例/负数中的-"""
    with open(file_path, encoding='utf-8') as f:
        content = f.read()
    original = content
    content = re.sub(r'(?<=[0-9])-(?=[0-9](?![-级层m]))', '~', content)
    changes = sum(1 for a, b in zip(original, content) if a != b)
    if changes > 0:
        open(file_path, 'w', encoding='utf-8').write(content)
        print(f'  修复 {file_path}: {changes//2} 处')
    else:
        print(f'  无需修复: {file_path}')

resource_dir = r'D:\...\resources'  # 按实际路径修改
for fname in os.listdir(resource_dir):
    if fname.endswith('.md'):
        fix_separators(os.path.join(resource_dir, fname))
```

---

## 进度跟踪规范

每阶段完成后立即更新：

**日期日志**（`.workbuddy/memory/YYYY-MM-DD.md`，追加）：
```
### 阶段X完成（HH:MM）
- 完成内容：...
- 发现问题：...（含修正：原值→正确值，PDF第X页，原因）
- 关键决策：...
```

**长期记忆**（`.workbuddy/memory/MEMORY.md`，更新）：
- 项目背景 / 文档信息 / 资源文件状态（checkbox）
- **已修正PDF错误清单**（原值→正确值 | PDF第X页 | 原因）
- 测试验证状态 / 开发原则

---

## 经验教训

1. **分阶段交付远优于一次完成** — P0确认方向再P1P2，避免大量返工
2. **双工具交叉验证** — fitz截行、pdfplumber截页各有盲区，关键数据必须交叉验证
3. **格式规范开工前定** — 分隔符/页码/命名后期统一成本极高（实测373处替换）
4. **错误修正必须留痕** — 对照表是后续验证和交付报告的依据
5. **大文件并行Agent** — 4个90KB文件并行快3~4倍，但格式一致性由主Agent统一校验
6. **分发版用相对路径** — 绝对路径在他人机器无效；验证用Python zipfile不用PowerShell（中文乱码）
7. **测试覆盖数据+功能+边界** — 三层缺一不可
8. **fitz验证失败≠数据错误** — 跨行截断导致搜索失败，需人工区分工具限制与真实错误

---

## 文档类型适配指南

| 文档类型 | 资源拆分 | 特殊处理 | 典型校验 |
|---------|---------|---------|---------|
| 工程造价指引 | 按项目类型+工程分类；通用独立 | 计量规则+校验规则+指标三层 | 重复计列、专项优先、超范围 |
| 技术标准规范 | 按专业章节；强制/推荐分开 | 标注强制性（"应/宜/可"） | 强制条文不可跳过 |
| 操作手册/SOP | 按流程；前置条件独立 | 保留步骤序号；标注并行步骤 | 步骤顺序、前置条件缺失 |
| 法律法规汇编 | 按主题/法条；按效力层级 | 生效日期+版本；废止标注 | 上位法优先、时效检查 |
| 产品参数手册 | 按产品系列；参数表独立 | 兼容性矩阵；选型决策树 | 参数越界、兼容冲突 |
| 财务测算模板 | 按模块（收入/成本/融资） | 输入参数与公式严格分离 | 公式逻辑、参数范围 |

---

## 版本记录

| 版本 | 时间 | 说明 |
|-----|------|------|
| v1.0.0 | 2026-05-28 | 初始版本 |
| v2.0.0 | 2026-06-02 | 新增决策树/完整脚本/错误速查表/经验教训/适配指南 |
| v3.0.0 | 2026-06-03 | 新增阶段6打包前检查复核及质量评估；全面结构优化 |
| v3.1.0 | 2026-06-03 | 理顺阶段编号：5.5→6，6→7，同步交叉引用 |

---

© 全网 **AI探子队长**
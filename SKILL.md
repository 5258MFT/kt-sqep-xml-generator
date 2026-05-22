---
name: mindmap-xml-generator
description: Converts mindmaps, outlines, hierarchical content, and teaching syllabi into 教学知识图谱 XML files. ALWAYS trigger when the user provides any tree/outline/hierarchy and asks to generate, create, or convert it to XML — especially knowledge graph XML (知识图谱 XML), teaching knowledge graph (教学知识图谱), or mindmap XML. Trigger when the user mentions any combination of: (1) a mindmap/outline/syllabus/tree structure or hierarchical content, (2) generating/converting/creating an XML file with entities and relations. Also trigger for requests to create knowledge graph files from chapter content, course outlines, subject hierarchies, or nested markdown lists. Do NOT trigger when the user only wants to read, check, validate, merge, parse, or analyze existing XML files; extract data from XML; convert to non-XML formats (JSON, CSV, tables); or ask general XML learning questions.
---

## Purpose

Generate XML files that conform to the "教学知识图谱" (Teaching Knowledge Graph) format. The user provides a mindmap hierarchy — as a tree outline, nested markdown list, indented text, or descriptive content — and you produce a complete, valid XML file with entities and parent-child relations.

## Reference Format

Before generating any XML, always read the reference files in the `references/` directory located in the same folder as this skill file.These files are the ground truth for the exact XML schema, field values, and conventions. Read at least 2-3 reference files to understand the patterns for different document structures.

## XML Structure

```xml
<KG>
    教学知识图谱
    <entities>
        <entity>
            <id>N</id>
            <class_name>节点类型名称</class_name>
            <classification>内容方法型节点</classification>
            <identity>知识</identity>
            <level>层级描述</level>
            <attach>六位数字代码</attach>
            <opentool>无</opentool>
            <content>节点文本内容</content>
            <x>浮点坐标X</x>
            <y>浮点坐标Y</y>
        </entity>
    </entities>
    <relations>
        <relation>
            <name>包含</name>
            <headnodeid>父节点ID</headnodeid>
            <tailnodeid>子节点ID</tailnodeid>
            <class_name>包含关系</class_name>
            <mask>知识连线</mask>
            <classification>包含关系</classification>
            <head_need>内容方法型节点</head_need>
            <tail_need>内容方法型节点</tail_need>
        </relation>
    </relations>
</KG>
```

## Node Type Mapping (class_name)

Map each mindmap node to a class_name based on its hierarchy depth:

| Mindmap Level | class_name | Meaning |
|---|---|---|
| 1 (root) | `知识领域:KA` | Knowledge domain / root topic |
| 2 | `知识单元:KU` | Knowledge unit / major section |
| 3 | `知识点:KP` | Knowledge point / sub-topic |
| 4+ (leaf) | `关键知识细节:KD` | Key knowledge detail / content |

## Level Mapping

| Mindmap Level | level field |
|---|---|
| 1 (root) | `一级` |
| 2 | `二级` |
| 3 (知识点 branch) | `归纳级` |
| 4+ (leaf/detail) | `内容级` |

## Field Rules

### Fixed fields (apply to ALL entities)

- `<classification>`: Always `内容方法型节点`
- `<identity>`: Always `知识`
- `<opentool>`: Always `无`

### attach field

The `attach` field is a 6-digit code. Use these patterns:

- `KD` (关键知识细节) leaf nodes: typically `000000`
- `KP` (知识点) nodes: typically `000100`
- `KU` (知识单元) nodes: typically `001000`
- `KA` (知识领域) root node: typically `010000`

If uncertain, use `000000` for all nodes.

### ID assignment

Assign sequential numeric IDs starting from 1. Process the tree depth-first so parent IDs are always lower than their children's.

## Node Count Management

**Target: 40–150 entities total.** When the source content is rich, you will easily exceed 150 if you create one KD per bullet point. Use these strategies to stay within bounds:

### Merge similar leaf items

When a section has many short bullet points (8+ items), consolidate them into 4–6 grouped KD nodes. For example, instead of 12 separate "complexity/difficulty" nodes like "预测困难", "提前期短", "设计变更难控", merge into 4–5 thematic groups:

```
物料管理的复杂性与难度 (KP)
├── 预测困难，提前期短交货急迫 (KD)
├── 设计变更难以控制，生产计划经常变动 (KD)
├── 制造现场生产进度不明，绩效衡量困难 (KD)
├── 供应商交货期难以控制，品质不稳定 (KD)
└── 解决方法：需要有效的计划与控制 (KD)
```

### Keep formulas on one line

Multiple related formulas (POH, PAB, NR) should each be a single KD node rather than split across multiple nodes.

### Combine related concepts

Two closely related ideas (e.g., "LLC定义" and "低阶码定义") can share one KP with 2–3 KD children rather than separate KPs.

## Coordinate Layout (x, y)

**Use automatic coordinate generation rather than manual assignment.** Build the tree structure first, then compute positions programmatically:

1. Build a `children_of` dictionary from all parent-child relations
2. Assign depth via DFS: root=0, children=depth+1
3. Map depth to x: `{0: 300.0, 1: 750.0, 2: 1200.0, 3: 1600.0}`
4. Assign y via DFS: start at 100.0, add 200.0 per node, then add 400.0 offset

This produces a left-to-right tree layout where siblings stack vertically and each depth level has a consistent x-coordinate.

## Relations

### Parent-child "包含" relations

For every parent → child relationship, create a relation:

- `<name>`: `包含`
- `<class_name>`: `包含关系`
- `<classification>`: `包含关系`

### Sequential "次序关系" between sibling chains

When siblings form a logical sequence (steps, timeline, ordered progression), add sequential relations between consecutive siblings:

- `<class_name>`: `次序：次序关系`
- `<classification>`: `次序关系`

Parallel categories only need 包含关系; ordered sequences need both types.

## XML Formatting Rules

**Proper formatting matters** — the output should match the reference files exactly:

- Use **tab characters (`\t`)** for indentation (not spaces)
- **No blank lines** between elements — every line should contain either a tag or text
- **No extra whitespace** inside tags — `<id>1</id>` not `<id>\n\t\t\t1\n\t\t</id>`
- Each entity and relation should be on contiguous lines
- Escape special XML characters in content: `&` → `&amp;`, `<` → `&lt;`

## Recommended Generation Approach: Python Script

**For complex outputs (many nodes), write a Python script to generate the XML.** This is significantly more reliable and efficient than manually constructing XML text, especially for node counts above 60. Use this pattern:

### 1. Define helper functions

```python
entities = []
relations = []
next_id = 1

def ent(class_name, level, content, attach="000000"):
    global next_id
    eid = next_id; next_id += 1
    entities.append((eid, class_name, level, content, attach))
    return eid

def rel(pid, cid, cn="包含关系", cl="包含关系"):
    relations.append((pid, cid, cn, cl))

def seq(ids):
    for i in range(len(ids)-1):
        rel(ids[i], ids[i+1], "次序：次序关系", "次序关系")
```

### 2. Define the tree using helpers

```python
root = ent("知识领域:KA", "一级", "第6讲 生产计划管理一", "010000")
ku1 = ent("知识单元:KU", "二级", "制造企业物料管理的基本问题", "001000")
rel(root, ku1)
kp = ent("知识点:KP", "归纳级", "物料管理的基本要求", "000100")
rel(ku1, kp)
d1 = ent("关键知识细节:KD", "内容级", "保证需求：正确、准确、及时")
rel(kp, d1)
```

### 3. Auto-compute coordinates

```python
children_of = {}
for pid, cid, _, _ in relations:
    children_of.setdefault(pid, []).append(cid)

# Depth-first depth assignment
depth_map = {root: 0}
def set_depth(pid, d):
    for c in children_of.get(pid, []):
        depth_map[c] = d; set_depth(c, d+1)
set_depth(root, 1)

x_map = {0: 300.0, 1: 750.0, 2: 1200.0, 3: 1600.0}

# Depth-first y assignment
ymap = {}; yc = [100.0]
def set_y(nid):
    ymap[nid] = yc[0]; yc[0] += 200
    for c in children_of.get(nid, []): set_y(c)
set_y(root)
```

### 4. Write XML directly (no minidom — it introduces blank lines)

```python
def esc(s):
    return s.replace("&","&amp;").replace("<","&lt;").replace(">","&gt;")

lines = ["<KG>", "\t教学知识图谱", "\t<entities>"]
for eid, cn, lv, ct, at in entities:
    x = x_map.get(depth_map.get(eid, 0), 300.0)
    y = ymap.get(eid, 100.0) + 400.0
    lines.append(f"\t\t<entity>")
    lines.append(f"\t\t\t<id>{eid}</id>")
    lines.append(f"\t\t\t<class_name>{esc(cn)}</class_name>")
    lines.append(f"\t\t\t<classification>内容方法型节点</classification>")
    lines.append(f"\t\t\t<identity>知识</identity>")
    lines.append(f"\t\t\t<level>{esc(lv)}</level>")
    lines.append(f"\t\t\t<attach>{at}</attach>")
    lines.append(f"\t\t\t<opentool>无</opentool>")
    lines.append(f"\t\t\t<content>{esc(ct)}</content>")
    lines.append(f"\t\t\t<x>{x}</x>")
    lines.append(f"\t\t\t<y>{y}</y>")
    lines.append(f"\t\t</entity>")
# ... relations similarly ...
lines.append("\t</relations>")
lines.append("</KG>")
```

### Why this approach works better

- **`ent()` and `rel()`** are concise — each node/relation is one line
- **Auto-ID** via `next_id` counter eliminates manual counting
- **Auto-coordinates** from tree structure — no manual x,y guessing
- **Direct string output** avoids `minidom` blank lines and extra whitespace
- **`seq()` helper** makes sequential relations trivial

## Step-by-Step Process

1. **Parse the user's input** — identify the tree structure (indentation, markdown lists, or described hierarchy)
2. **Read reference files** — read 2-3 files from the `reference/` directory to confirm conventions
3. **Plan the hierarchy** — decide which items to merge to stay within 40–150 node limit
4. **Build the entity list** — use `ent()` helper, traverse tree depth-first, assign IDs
5. **Build the relation list** — use `rel()` for parent-child, `seq()` for ordered siblings
6. **Auto-compute coordinates** — build `children_of` tree, assign depth→x and DFS order→y
7. **Write XML directly** — string concatenation with `\t` indentation, no blank lines
8. **Save the file** — meaningful name based on content (e.g., `第6讲-生产计划管理一.xml`)

## Input Formats

The skill accepts mindmap structures in these formats:

### Indented text

```
第一章 企业资源规划
    什么是ERP和SCM
        核心概念解析
            ERP定义
            SCM定义
```

### Markdown list

```
- 第一章 引言
  - 背景介绍
    - 研究动机
    - 问题定义
```

### Descriptive content

The user describes the content verbally; extract the hierarchy and infer the logical structure.

## Output

Write the generated XML to a file in the user's working directory. Suggest a filename based on the content (e.g., `第1章.xml`, `L01-计算机.xml`). Confirm the file path with the user if uncertain.


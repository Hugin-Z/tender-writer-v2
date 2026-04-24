# tender-writer 变更日志

## v2.0.0 · 2026-04-23 首发

v2 在 tender-writer v1.1 基础上经过两轮迭代沉淀而来,包含**项目类型识别、AI 输出规则常驻化、docx 渲染质量总攻、非交互批量化、跨章节一致性检查、工具链 bug 清扫**等十项改进。

### v1.1 → v2.0.0 主要变化

| 模块 | v1.1 | v2.0.0 |
|---|---|---|
| outline 模板 | 单一模板,手工重组 | 按 `extracted.project_type` 五选一(工程/平台/研究/规划/其他),自动选骨架 |
| AI 输出规则 | 散落在 SKILL.md 各章节 | 常驻文档 `references/ai_output_rules.md`,R1-R10 十条红线 |
| docx 渲染 | 默认灰色字体,中英混排空格不清理 | 统一黑色 RGB(0,0,0) + 中文宋体 + 相对字号 + `**bold**` 内联解析 + 图占位区块 + 中英空格后处理 |
| C 模式填充 | 交互式 `input()`,多 Part 需 10 次手动回答 | 非交互,变量缺失时写入"【待填:xxx】"红色占位;`c_mode_run.py --all` 一键批量 |
| B 模式组装 | 单 Part 单次手动 | `b_mode_run.py --all` 一键批量 |
| 跨章节一致性 | 无(只能靠 compliance 粗查) | 新增 `check_cross_consistency.py` 抓团队成本 vs 预算、承诺时间 vs 工期等矛盾 |
| 投标主体选择 | 硬编码或无支持 | `select_bidding_entity.py` 显式选择,写入 `extracted.bidding_entity`,多主体场景停下让用户选 |
| v45_merge 合并顺序 | 硬编码某项目 Part 顺序 | 从 `response_file_parts` 动态推导,任何项目通用 |
| export_deliverables 映射 | 硬编码某项目 Part 名 | 动态生成交付物清单 |
| check_chapter 检查项 | 6 项 | 新增 [7] AI 输出空格规范(R3 落地),≥3 处 fail |
| 协作契约 | 隐含在 SKILL.md | `CLAUDE.md` 6 条硬红线,任何 AI 代理工作前必读 |

### 新增文件

- `CLAUDE.md` — Claude Code 协作契约(6 条红线)
- `references/ai_output_rules.md` — AI 输出常驻约束(R1-R10)
- `references/outline_templates/{engineering,platform,research,planning,other}/` — 5 类 outline 模板
- `scripts/c_mode_run.py` / `scripts/b_mode_run.py` — 一键批量运行器
- `scripts/check_cross_consistency.py` — 跨章节一致性检查
- `scripts/c_mode_docx_passthrough.py` — Word 上游原样切用(beta)
- `scripts/select_bidding_entity.py` — 投标主体选择
- `scripts/tests/test_budget_parsing.py` — 预算解析 fixture
- `docs/v2_design_notes.md` — 设计决策记录
- `docs/v2_roadmap.md` — 未来路线图

### 示范项目

`projects/demo_cadre_training/` — 虚构"A 市 2026 年度干部综合能力提升培训服务采购项目",预算 500000 元,工期 60 日历日,Clone 下来可直接跑通端到端。

---

## v1.1 及之前

v1.1 及之前的变更记录未迁入 v2 对外版。v1 是内部迭代期,对外公开从 v2.0.0 起步。

### v2 补丁 2(字体安全回归,2026-04-24)

**背景**:demo 重跑后 WPS 打开 final_response.docx 提示"2 个文档字体缺失:Courier、MS 明朝",标题与正文部分 fallback 到日文字体 MS 明朝。属于 v2(二轮迭代)docx 渲染质量总攻本该覆盖但漏了的回归,补作 v2 补丁 2。

**根因**:

- `c_mode_extract.render_template_docx`、`b_mode_fill.main`、`v45_merge._create_part_divider`、`v45_merge._create_inapplicable_doc` 四处用裸 `Document()` 创建 docx,**未调 `apply_default_styles(doc)`**
- `styles.xml` 的 `Normal` 样式留空,`docDefaults` 里是 `<w:rFonts w:eastAsiaTheme="minorEastAsia">`(主题字体引用)
- Office/WPS 遇到 `minorEastAsia` 主题但主题文件缺失或 `themeFontLang` 为日语时,fallback 到系统日文字体 MS 明朝
- C/B 模式所有片段 run 级 100% 无 `rFonts`,全部 fallback;v45_merge 的 divider 又成为合并 master,把空 Normal 样式传染给 final_response.docx

**修复**(最小改动):

- `scripts/c_mode_extract.py::render_template_docx`:`Document()` 后加 `apply_default_styles(doc)`
- `scripts/b_mode_fill.py::main`:同上
- `scripts/v45_merge.py::_create_inapplicable_doc` + `_create_part_divider`:同上
- 共 4 处 `Document()` 裸调用改为 `Document() + apply_default_styles(doc)`

**回归检查(补强 docx 渲染质量总攻)**:

- `scripts/compliance_check.py` 新增 `check_font_safety(docx_path)`:
  - Normal 样式必须有 `<w:rFonts>` 且 `eastAsia` 在白名单(宋体/仿宋/仿宋_GB2312/黑体/微软雅黑)
  - run 级 `<w:rFonts>` 的 `eastAsia` / `ascii` 属性须在白名单
  - 异常字体(如 `MS Mincho` / `Arial Unicode`)报 `format_issues`
- 接入 `main()` 的 format_issues 扩展,不改报告结构

**回归验证**(demo 项目):

| 指标 | 修复前 | 修复后 |
|---|---|---|
| C 模式 filled.docx(6 个)run 级 rFonts | 0/9 / 0/18 / 0/12 / 0/8 / 0/12 / 0/12 | Normal 样式继承宋体 |
| B 模式 assembled.docx(2 个)run 级 rFonts | 0/19 / 0/9 | Normal 样式继承宋体 |
| final_response.docx Normal 样式 | `<w:style ...>` 内**无** `<w:rFonts>` | `<w:rFonts w:ascii="Times New Roman" w:hAnsi="Times New Roman" w:eastAsia="宋体"/>` |
| compliance_check 字体检查 | (不存在该检查项)| 0 issue |
| WPS 打开 | 提示"2 个文档字体缺失" + 渲染日文 MS 明朝 | (待用户 WPS 重开确认) |

**baseline**:v3 → v4(4 个脚本改动,工具链指纹刷新)。


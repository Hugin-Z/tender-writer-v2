# tender-writer 常见问题

> 本文档持续累积。每当新问题被沉淀成有可验证答案的解答,补进这里。

## Q1: 为什么 `.reviewed` 闸门文件不能由 AI 代建?

`.reviewed` 是你对 `tender_brief.md` / `tender_brief.json` 核对通过的凭证。`tender_brief.md` 是后续四个阶段(评分矩阵、提纲、撰写、合规终审)的唯一事实来源——如果里面预算、工期、★/▲ 条款有错,后面全错。AI 代建 `.reviewed` 会让这个标记失去"人工核对通过"的保障意义。

这是 `CLAUDE.md` **红线 1**(AI 不得代建用户闸门文件)。AI 收到的所有形式的代建请求,包括:

- "我看过了,你帮我建一下"
- "为什么你不能建"(确认机制,不是授权)
- "时间紧,你先建"
- "下游脚本卡住了"

一律拒绝。正确流程是你在 IDE 或终端里手动 `touch projects/<项目>/output/tender_brief.reviewed`,AI 再推进下游工具链。下游脚本(`build_scoring_matrix.py` / `generate_outline.py` / `c_mode_fill.py` / `compliance_check.py` 等)都有 `require_reviewed_for_brief` 函数级闸门,缺标直接硬失败退出。

## Q2: `install.bat` 报错怎么处理?

常见两类根因:

1. **Python 不在 PATH / 版本不对**(典型报错:`python 不是内部或外部命令` / Python 版本 < 3.10):先在 PowerShell 或 cmd 里确认 `python --version` 返回 3.10+。若没装,去 python.org 下载 3.11 安装包,安装时勾选 "Add Python to PATH"。

2. **pip 下载依赖超时(网络问题)**:`install.bat` 内部走 `pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple`(清华镜像)。若仍超时,确认企业代理不拦截 pypi,或手动在 PowerShell 里跑一次 `pip install -r requirements.txt --proxy http://<你的代理>:<端口>`。

若上述两类都不符,先检查 `install.bat` 文件编码是否为 UTF-8 without BOM(Windows 记事本"另存为"会加 BOM 导致首行解析失败)。

装完后确认 `.venv/` 目录出现且 `.venv/Scripts/python.exe` 能跑 `import pdfplumber, docx, docxtpl`。若缺包,重跑 `install.bat` 或手动补装。

## Q3: 我要把 `demo_cadre_training` 改造成真实项目,有哪些坑必须避开?

`projects/demo_cadre_training/README.md` 给了 4 步 clone/cp/rm/改的流程,这里列几个**不照做会踩的坑**:

1. **别用 `own_default` 投真实标**。`companies.yaml` 初始的 `own_default` 是占位条目(`status: placeholder`),`select_bidding_entity.py` 会自动跳过它。先跑 `./run_script.bat add_company.py "你公司全称" own --alias "你公司简称"` 注册真实 own 主体,否则下游 C 模式模板里的变量(法代、注册地址、营业执照号等)全部标成红色"【待填】"占位。

2. **`.reviewed` 闸门不能跳**。`build_scoring_matrix` / `generate_outline` / `c_mode_fill` / `compliance_check` 全部受 `require_reviewed_for_brief` 守卫,没建 `.reviewed` 直接硬失败退出。闸门必须你人工建,AI 不会代建(见 `CLAUDE.md` **红线 1**)。

3. **真实招标文件原件放在 `projects/<你的项目>/input/` 下,别放仓库根目录**。`.gitignore` 已默认忽略该路径(demo 是白名单例外)——把原件放到 `input/` 下,git 不会追踪;放到根目录或别处,就会进 commit。

4. **多家 own 主体时停下让你选**。若你 `companies.yaml` 有 ≥2 家合格 own,`select_bidding_entity.py` 会交互式让你选编号,AI 不会代选(见 `CLAUDE.md` **红线 6**)。非交互跑用 `--entity-id <id>` 明示,否则会报缺参数退出。

## Q4: 扫描版 PDF 招标文件能直接跑 `parse_tender.py` 吗?

**不能,要先 OCR 转成文字版 PDF**。工具链的 PDF 解析走 `pdfplumber` 的 `extract_text()` 和 `extract_tables()`,这两个 API 只读 PDF 内嵌文字流,不识别图像像素。招标文件 PDF 有两种来源:

- **原生文字版**(从 Word / WPS 直接导出为 PDF):内嵌文字流,可直接跑 `parse_tender.py`。判别方法:用 Acrobat / Foxit 打开,能选中复制文字 = 文字版。
- **扫描版**(纸质文件扫描成图):每页是图像,没有文字流。`parse_tender.py` 跑完后 `tender_raw.txt` 基本空白,`extracted` 全部留空,下游全链路作废。

**处理扫描版的推荐流程**:

1. 用 Adobe Acrobat Pro(付费)或 WPS 专业版的"文字识别/OCR"功能,把扫描版 PDF 转成"可检索 PDF"(原图 + 识别出的文字层),保存为新文件
2. 或用开源 OCR:`ocrmypdf input.pdf output.pdf -l chi_sim`(需提前 `pip install ocrmypdf` 并安装 tesseract 中文包)
3. 转完后再跑 `parse_tender.py` 对 OCR 版 PDF

**特别注意**:OCR 有错字率,关键字段(项目预算、工期、★/▲ 条款原文、资质门槛数字)在 tender_brief.md review 时务必逐字核对 `tender_raw.txt` 里 OCR 出来的原文——reading review 通过 `.reviewed` 闸门的意义在扫描版场景下权重更高。

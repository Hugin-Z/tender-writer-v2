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

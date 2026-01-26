# AGENTS.md
# 面向本仓库的智能代理协作指南（约 150 行）

## 仓库概览
- 项目：A 股自选股智能分析系统（Python 3.10+）
- 主要入口：`main.py`
- WebUI：`webui.py` + `web/`
- 配置：`.env` / `.env.example` + `config.py`
- 数据：`data/`、`logs/`、`reports/`

## 规则来源说明
- 未发现 Cursor 规则：`.cursor/rules/`、`.cursorrules`
- 未发现 Copilot 规则：`.github/copilot-instructions.md`
- 现有格式化/检查配置：`pyproject.toml`、`setup.cfg`

## 常用命令
### 依赖与环境
- 创建虚拟环境：`python -m venv venv`
- 激活（Windows）：`venv\Scripts\activate`
- 安装依赖：`pip install -r requirements.txt`
- 复制配置：`cp .env.example .env`

### 运行
- 默认运行：`python main.py`
- 仅股票分析：`python main.py --no-market-review`
- 仅大盘复盘：`python main.py --market-review`
- 启用 WebUI：`python main.py --webui`
- 仅 WebUI：`python main.py --webui-only`

### 测试（pytest）
- 运行全部测试：`pytest`
- 单文件：`pytest path/to/test_file.py`
- 单用例：`pytest path/to/test_file.py::test_case_name`
- 关键词过滤：`pytest -k "keyword"`
- 失败即停：`pytest -x`

### Lint / 格式化 / 安全
- Black 格式化：`black .`
- isort 排序：`isort .`
- Flake8 静态检查：`flake8 .`
- Bandit 扫描：`bandit -r . -x ./test_*.py`

### CI 对齐（本地模拟）
- 语法检查：
  `python -m py_compile main.py config.py analyzer.py notification.py`
  `python -m py_compile storage.py scheduler.py search_service.py`
  `python -m py_compile market_analyzer.py stock_analyzer.py`
  `python -m py_compile data_provider/*.py`
- Flake8 严重错误：`flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics`

### Docker
- 构建镜像：`docker build -t stock-analysis:test .`
- 运行镜像：`docker run --rm stock-analysis:test python -c "print('Docker OK')"`
- Compose（定时）：`docker-compose up -d`
- Compose（仅分析器）：`docker-compose up -d analyzer`
- Compose（仅 WebUI）：`docker-compose up -d webui`
- Compose（重建镜像）：`docker-compose up -d --build`

## 代码风格与工程约定
### 总体风格
- 遵循 PEP 8；行宽 120（Black/Flake8/isort 已配置）
- 模块/类/函数均使用清晰的中文 docstring
- 重要流程用日志而非 print，日志级别合理分层
- 结构优先：配置、数据访问、分析、通知、WebUI 分层清晰

### 导入顺序（由 isort + black 维护）
- 标准库
- 第三方库
- 本地模块（如 `config`、`storage`、`data_provider`）
- 避免循环导入；必要时局部导入并注明原因

### 格式化与静态检查
- Black：`line-length = 120`，保持一致性
- Flake8：忽略 `E501/W503/E203/E402`（与 black 冲突或有意为之）
- isort：`profile = black`，`known_first_party` 已在 `setup.cfg`

### 类型与数据结构
- 重要数据结构使用 `@dataclass`（如 `AnalysisResult`）
- 公共接口尽量显式类型注解（`Optional` / `List` / `Dict`）
- 返回值含状态与错误信息时使用 `Tuple[bool, Optional[str]]` 等组合
- 配置集中在 `Config`，避免散落的全局常量

### 命名规范
- 模块/函数/变量：`snake_case`
- 类名：`PascalCase`
- 常量：`UPPER_SNAKE_CASE`
- 日志器：`logger = logging.getLogger(__name__)`
- 文件与模块命名保持语义清晰（如 `market_analyzer.py`）

### 错误处理与日志
- 关键流程使用 `try/except` 捕获并记录异常
- 单个股票失败不能影响整体执行（流水线鲁棒性）
- 记录可追踪的上下文：股票代码、阶段、异常信息
- 使用 `logging`，避免裸 `print`
- 避免静默失败，必要时返回错误信息并向上层汇报

### 配置与环境变量
- `.env` 为本地配置来源；`config.py` 统一读取并提供默认值
- 新增配置时：
  - 更新 `Config` dataclass
  - 更新 `.env.example`
  - 如涉及文档，补充 `docs/full-guide.md`
- 不在代码中硬编码 API Key

### 业务逻辑建议
- 分析逻辑：优先可解释性，包含结论、理由、风险提示
- 数据获取：注意限流和重试（已有 `max_retries` / `retry_delay`）
- 结果输出：保证字段完整，不遗漏 JSON 必要字段
- 通知内容长度受限（飞书约 20KB，企业微信约 4KB）

### 文件与目录约定
- `data/` 数据库与缓存
- `logs/` 运行日志
- `reports/` 分析结果输出
- `sources/` 仅存示例图片/素材，避免写入动态内容

### 注释与文档
- 非显而易见的算法或策略写清原因
- 中文注释优先，保持简洁
- 面向用户的改动同步到 `README.md` 或 `docs/full-guide.md`

### WebUI 约定
- `web/` 内为轻量 HTTP 服务，无重型框架依赖
- HTML 模板集中在 `web/templates.py`
- API 端点与行为需同步到 README 的接口表

## 变更建议与自查清单
- 新增依赖需更新 `requirements.txt`
- 运行 `black .` + `isort .` 后再提交
- 核心改动至少跑一次 `pytest`（单用例优先：`pytest path/to/test_file.py::test_case_name`）
- 变更通知/配置时，注意同步 `.env.example`
- 避免提交敏感文件（`.env`、密钥、token）

### Git 同步与维护规范
- **保持线性历史**：同步上游（upstream）时，始终使用 `rebase` 而非 `merge`。
  - 命令：`git fetch upstream && git rebase upstream/main`
- **避免冗余合并提交**：除非存在必须手动解决的冲突，否则提交历史中不应出现 "Merge remote-tracking branch..." 字样。
- **强制推送策略**：在本地 `rebase` 成功后，若远程仓库（origin）历史已分叉，需使用 `git push origin main --force` 同步。
- **保持领先状态**：确保仓库在 GitHub 上显示为“领先 N 个提交”（N 为 `AGENTS.md` 等本地特有修改），而不是同时“领先且落后”。

## 提交信息（供参考）

- 采用 Conventional Commits（见 `CONTRIBUTING.md`）
- 示例：`feat: 添加飞书文档推送`

## 常见问题速查
- 语法错误：先跑 `python -m py_compile ...`
- 模块导入失败：检查 `PYTHONPATH` 或相对导入
- 配置不生效：确认 `.env` 在仓库根目录并被加载
- 数据库锁：停止进程后删除 `data/*.lock`

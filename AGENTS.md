# AGENTS.md
# 面向本仓库的智能代理协作指南

## 仓库概览
- 项目：A 股自选股智能分析系统（Python 3.11+）
- 核心代码：`src/` 目录下（`analyzer.py`, `config.py`, `notification.py` 等）
- 主要入口：`main.py`（位于根目录，调用 `src` 模块）
- WebUI：`webui.py` + `web/` + `src/`
- 配置：`.env` (本地) + `src/config.py` (配置类)
- 数据：`data/` (SQLite, Lockfiles), `logs/`, `reports/`

## 规则来源
- 未发现 Cursor/Copilot 特定规则文件。
- 遵循 `pyproject.toml` (Black/Isort) 和 `setup.cfg` (Flake8/Pytest) 配置。

## 常用命令

### 依赖与环境
- 创建虚拟环境：`python -m venv venv`
- 激活（Windows）：`venv\Scripts\activate`
- 安装依赖：`pip install -r requirements.txt`
- 复制配置：`cp .env.example .env`

### 运行
- **标准运行**：`python main.py`
- **仅股票分析**：`python main.py --no-market-review`
- **仅大盘复盘**：`python main.py --market-review`
- **启用 WebUI**：`python main.py --webui` (或 `python main.py --webui-only`)

### 测试 (pytest)
- 运行所有测试：`pytest`
- 运行单文件：`pytest test_env.py`
- 运行单用例：`pytest test_env.py::test_config_loading`
- 失败即停：`pytest -x`
- 详细输出：`pytest -v`

### Lint / 格式化 / 静态检查
- **Black 格式化**：`black .`
- **Isort 排序**：`isort .`
- **Flake8 检查**：`flake8 .`
- **语法自检 (CI 模拟)**：
  ```bash
  python -m py_compile main.py webui.py
  python -m py_compile src/config.py src/analyzer.py src/notification.py src/storage.py
  python -m py_compile src/core/pipeline.py src/core/market_review.py
  python -m py_compile data_provider/*.py
  ```

### Docker 操作
> 注意：Docker 配置位于 `docker/` 目录，建议使用根目录下的脚本或指定文件路径
- **构建并启动**：
  ```bash
  docker compose --env-file .env -f docker/docker-compose.yml -f docker-compose.override.yml up -d --build
  ```
- **仅构建镜像**：
  ```bash
  docker compose --env-file .env -f docker/docker-compose.yml -f docker-compose.override.yml build
  ```
- **启动定时任务（analyzer）**：
  ```bash
  docker compose --env-file .env -f docker/docker-compose.yml -f docker-compose.override.yml up -d analyzer
  ```
- **启动 WebUI（webui）**：
  ```bash
  docker compose --env-file .env -f docker/docker-compose.yml -f docker-compose.override.yml up -d webui
  ```
- **启动 API 服务（server）**：
  ```bash
  docker compose --env-file .env -f docker/docker-compose.yml -f docker-compose.override.yml up -d server
  ```
- **同时启动 analyzer + webui**：
  ```bash
  docker compose --env-file .env -f docker/docker-compose.yml -f docker-compose.override.yml up -d analyzer webui
  ```
- **查看日志**：`docker logs -f stock-analyzer`
- **清理旧镜像**：`docker image prune -f`

## 代码风格与工程约定

### 1. 目录结构与导入
- **核心逻辑**：全部位于 `src/` 目录下。
- **导入规范**：
  - 根目录脚本（`main.py`）导入：`from src.config import Config`
  - `src` 内部相互导入：使用绝对导入 `from src.storage import Storage` 或相对导入 `from .enums import ...`
- **数据源**：`data_provider/` 独立于 `src/`，作为基础组件被 `src` 调用。

### 2. 格式化规范
- **行宽**：120 字符（Black/Flake8 已配置）。
- **引号**：双引号 `"` 优先（Black 默认）。
- **导入排序**：标准库 -> 第三方库 -> 本地模块 (`src`, `data_provider`)。由 `isort` 自动维护。

### 3. 类型与命名
- **类型注解**：公共函数必须包含类型注解。
  - 推荐：`def analyze(code: str) -> Optional[AnalysisResult]:`
- **命名**：
  - 变量/函数：`snake_case`
  - 类名：`PascalCase`
  - 常量：`UPPER_SNAKE_CASE`
- **配置**：不要使用全局变量，统一通过 `src.config.Config` 单例获取。

### 4. 错误处理与日志
- **异常捕获**：
  - 避免裸 `try...except`，尽量捕获具体异常。
  - 单个股票分析失败 **不应** 中断整个流程，应记录 ERROR 日志并继续。
- **日志**：
  - 使用 `logger = logging.getLogger(__name__)`。
  - 关键路径使用 `INFO`，调试信息使用 `DEBUG`，错误堆栈使用 `logger.error(..., exc_info=True)`。

### 5. Git 协作规范
- **同步上游**：使用 `rebase` 保持提交历史线性。
  - `git fetch upstream && git rebase upstream/main`
- **提交信息**：遵循 Conventional Commits (e.g., `feat: add new signal`, `fix: resolve timeout`).
- **文件变更**：
  - 优先修改 `src/` 下的现有文件。
  - `AGENTS.md` 为代理指南，若项目结构变更需同步更新此文件。

## 常见问题
- **ImportError**: 确认是否正确使用了 `from src.xxx`。
- **代理报错**: 检查 `.env` 中的 `HTTP_PROXY`，Docker 容器内需检查 `docker-compose.override.yml` 的环境变量映射。
- **依赖缺失**: 新增库后记得更新 `requirements.txt`。

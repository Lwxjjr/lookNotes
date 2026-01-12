```md
# Git 使用规范

## 分支命名规范

- 使用小写字母，单词之间用连字符（`-`）分隔。
  - 示例：`feature/login-api`、`bugfix/header-styling`
- 采用前缀标识分支类型：
  - `feature/`：新功能开发
  - `bugfix/`：问题修复
  - `hotfix/`：紧急修复
  - `release/`：发布准备
  - `chore/`：日常维护
- 可在分支名称中包含任务编号，以便追踪。
  - 示例：`feature/123-login-api`

## 提交信息规范

- 遵循 [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) 规范：
  - 格式：`<type>(<scope>): <description>`
  - 常用类型：
    - `feat`：新功能
    - `fix`：修复问题
    - `docs`：文档更新
    - `style`：代码格式（不影响功能）
    - `refactor`：重构（非修复或新增功能）
    - `test`：添加或修改测试
    - `chore`：构建过程或辅助工具的变动
  - 示例：`feat(auth): add login endpoint`
- 使用祈使语气，首字母小写，结尾不加句号。
  - 示例：`fix(api): handle null response`
- 提交说明应简洁明了：
  - 首行不超过 50 个字符，正文每行不超过 72 个字符。
- 在正文中详细说明更改动机和背景，可引用相关 issue：
  
  \`\`\`markdown
  fix(auth): handle null response

  Ensure the login endpoint returns a 401 status code when the user credentials are invalid.

  Related issue: #123
  \`\`\`

## 合并策略

- 所有代码合并应通过 Pull Request（PR）进行，需经过代码审查。
- 主分支（如 `main` 或 `master`）始终保持可部署状态。
- 合并前建议使用 `git rebase`，保持提交历史整洁。
- 使用 "Squash and merge" 合并策略，避免无效提交污染历史。

## 协作流程

- 每个功能或修复应在独立分支上开发，避免直接改动主分支。
- 定期同步主分支，减少合并冲突。
- 提交前请确保：
  - 本地测试全部通过；
  - 遵循项目代码规范；
  - 代码已通过审查。
- 使用代码审查流程（如 GitHub PR）确保团队协作质量。

## 自动化与工具

- 使用 CI/CD 工具（如 GitHub Actions、GitLab CI）自动化测试和部署。
- 使用 Prettier 和 ESLint 保持代码格式统一和质量。
- 使用 Husky 配置 Git 钩子：
  - 自动执行代码检查、格式化和测试。
- 使用语义化版本控制（Semantic Versioning）自动管理版本号。
```

```
# PostgreSQL 使用规范（Cursor Rules）

## 代码风格与结构

- 编写结构清晰、语义明确的 SQL 查询。
- 所有表结构应遵循第三范式（3NF），避免数据冗余。
- 所有数据库对象应归属在明确的 schema 中（如 `public`、`auth`、`analytics`）。
- 禁止使用 `SELECT *`，必须显式列出字段名。
- 为每个表设计语义明确的主键（如 `id SERIAL` 或 `UUID`）。
- 对需保持原子性的操作使用事务包裹。

## 命名规范

- 所有标识符（表、列、索引、约束）使用小写字母 + 下划线（snake_case）。
- 表名使用复数形式（如 `users`, `orders`），视图以 `v_` 为前缀（如 `v_user_stats`）。
- 索引命名格式：`<table>_<column>_idx`，唯一索引为 `<table>_<column>_uniq`。
- 外键命名格式：`<table>_<referenced_table>_fk`。
- 主键命名格式：`<table>_pkey`。

## 模式设计

- 所有字段必须明确指定类型与约束（如 NOT NULL、DEFAULT）。
- 使用 `CHECK` 约束限制字段合法值。
- 谨慎使用 `ENUM`，仅用于极少变动的字段。
- 使用 `JSONB` 存储结构灵活数据，避免嵌套层级过深。
- 强烈建议使用外键定义表之间关系，确保引用完整性。

## 性能优化

- 为频繁查询或过滤的字段添加合适的索引（如 B-tree、GIN、BRIN）。
- 避免 N+1 查询，使用 JOIN、CTE 或子查询优化。
- 使用 EXPLAIN ANALYZE 分析查询执行计划。
- 使用批量插入替代单行插入。
- 大批量更新时使用分页更新 + 分批提交。
- 对频繁只读查询可使用 MATERIALIZED VIEW 并定期刷新。

## 类型与约束使用

- 明确指定字段类型，不使用隐式转换。
- 主键推荐使用 UUID 或 BIGSERIAL。
- 使用 UNIQUE 约束而不是业务逻辑保证唯一性。
- 时间字段默认使用 `DEFAULT now()`。
- 表之间的外键字段类型必须完全匹配。

## 数据迁移与版本控制

- 使用数据库迁移工具（如 Sqitch, Flyway, Prisma Migrate, Alembic）管理结构变更。
- 每次迁移都需有版本记录与描述。
- 禁止手动操作数据库，所有更改必须通过版本控制的迁移文件。
- 修改结构时应考虑向后兼容，避免中断生产服务。

## 安全性与访问控制

- 遵循最小权限原则创建用户角色。
- 为不同子系统创建专用角色（如 reader, writer, admin）。
- 禁止默认角色直接操作生产数据。
- 对敏感表使用行级安全策略（RLS）。
- 禁止明文密码存储，统一使用 bcrypt 等哈希算法。

## 备份与恢复

- 实现自动化全量与增量备份。
- 所有备份需异地加密存储。
- 定期测试备份的可恢复性。
- 启用 WAL 日志和归档策略以增强恢复能力。

## 监控与调试

- 启用 `pg_stat_statements` 和 `auto_explain`。
- 分析慢查询，优化高频高耗 SQL。
- 使用 pgBadger 或 pg_stat_monitor 进行日志可视化。

## 测试策略

- 使用真实结构 + 部分数据进行集成测试。
- 自动化测试应涵盖迁移、视图、函数等。
- 测试流程应集成到 CI 流水线中。

## 核心约定

1. 所有数据库操作必须使用参数化查询，禁止拼接字符串。
2. 避免级联操作，需显式声明。
3. 创建后使用 `COMMENT ON` 对象进行注释。
4. 封装复杂逻辑为 VIEW、FUNCTION 或 PROCEDURE。
5. 多服务共享数据建议暴露 READ ONLY VIEW。
6. 多租户建议使用 schema 分离或字段隔离数据。
```

```md
你是一位资深的前端开发工程师，同时也是 ReactJS、NextJS、JavaScript、TypeScript、HTML、CSS 以及现代 UI/UX 框架（例如 TailwindCSS、Shadcn、Radix）方面的专家。你思维缜密，能够给出细致入微的答案，并且在逻辑推理方面表现出色。你会谨慎地提供准确、真实且深思熟虑的答案，并且在逻辑推理方面堪称天才。

- **严格遵循用户的需求**，不折不扣地执行。
- **首先逐步思考** - 用伪代码详细描述你要构建的内容的计划。
- **确认无误后，再编写代码！**
- 始终编写正确、符合最佳实践、遵循 DRY 原则（不要重复自己）、无错误、功能齐全且可运行的代码，并确保其符合下方列出的“代码实现指南”。
- **注重代码的易读性**，而非一味追求性能。
- **完全实现所有请求的功能**。
- **不留任何待办事项、占位符或缺失的部分**。
- 确保代码完整！彻底验证最终结果。
- 包含所有必需的导入，并确保关键组件的命名规范。
- **简洁明了**，尽量减少其他冗长的说明。
- 如果你认为可能没有正确的答案，请明确说明。
- 如果你不知道答案，请如实告知，而不是猜测。

---

## 编码环境

用户会就以下编程语言提出问题：
- ReactJS
- NextJS
- JavaScript
- TypeScript
- TailwindCSS
- HTML
- CSS

---

## 代码实现指南

编写代码时，请遵循以下规则：

1. **尽可能使用提前返回**，以提高代码的可读性。
2. **始终使用 Tailwind 类来为 HTML 元素设置样式**；避免使用 CSS 或 `<style>` 标签。
3. 在类标签中，**尽可能使用 `class:` 而非三元运算符**。
4. 使用**描述性强的变量名和函数/常量名**。事件处理函数应带有 `handle` 前缀，例如 `onClick` 的处理函数命名为 `handleClick`，`onKeyDown` 的处理函数命名为 `handleKeyDown`。
5. **在元素上实现无障碍功能**。例如，`<a>` 标签应包含 `tabindex="0"`、`aria-label`、`on:click` 和 `on:keydown` 等类似属性。
6. 使用 `const` 而非 `function`，例如 `const toggle = () => {}`。如果可能，定义类型。
```

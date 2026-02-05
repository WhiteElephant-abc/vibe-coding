### 完整架构概括 L4→L3→L2→L1→极简 L0

**L4 组织层 Roles Boundaries**  
七个角色：**Researcher, External Tester, Architect, Coder, Reviewer, Internal Tester, Documenter PM**。每个角色独立会话、独立职责、不得越界。

**L3 流程层 Pipelines**  
三条单向流水线：**外部依赖验证**（Researcher → External Tester → Researcher → Architect）、**内部实现**（Architect → Coder → Reviewer → Internal Tester）、**文档管理**（Internal Tester → Documenter PM → Architect）。

**L2 存储层 Repo Worktrees**  
仓库为协议载体。通过 role access map 定义角色可读写路径。每角色一个 worktree 或隔离视图，角色间仅通过文件通信。

**L1 执行层 Sessions**  
每个角色对应一个 Claude Code 会话或终端标签页。角色不共享上下文、记忆或 prompt。角色切换由 prompt 驱动。

**L0 极简自动化 Minimal L0（职责层面定义）**  
管理 Claude Code 实例、Claude 配置、提示词模板、pipeline 编排元数据、worktree 生命周期与并行安全检查；不包含具体实现细节或脚本。

---

### 1 Roles Definition and Contracts

| **Role** | **核心职责** | **主要输入** | **主要输出** | **禁止行为** |
|---|---:|---|---|---|
| **Researcher** | 识别并规范外部依赖与不确定行为 | 现有源码、第三方文档、依赖清单 | 外部依赖规范草案（spec draft） | 不实现代码、不修改 src |
| **External Tester** | 在真实或沙箱环境验证外部行为 | Researcher 草案、测试脚本 | 外部测试结果报告 | 不改架构文档、不实现内部逻辑 |
| **Architect** | 将外部规范封装为内部接口与模块边界 | 最终外部规范、系统目标 | 接口契约、模块图、错误映射 | 不实现业务代码 |
| **Coder** | 在内部接口上实现功能模块 | 接口契约、contracts | 实现代码、单元测试桩 | 直接调用外部依赖或未授权库 |
| **Reviewer** | 审查实现是否遵守契约与边界 | 实现代码、接口契约 | 审查报告、修正清单 | 跳过契约检查或运行外部测试 |
| **Internal Tester** | 在隔离环境验证实现正确性 | 实现代码、测试用例 | 内部测试报告、覆盖率摘要 | 访问真实外部服务 |
| **Documenter PM** | 汇总产物、维护 backlog 与 roadmap | 测试与审查结果 | README、roadmap、backlog | 直接实现代码或改接口契约（需 Architect 批准） |

**每个角色必须在其契约内工作，所有交付物包含 role 元数据与 task-id。**

---

### 2 L0 Responsibilities High Level

**L0 是运行时管理层，职责只定义不实现细节：**

- **Claude Code Instance Management**：记录并管理每个角色对应的 Claude 实例或会话模板（实例标识、会话配置、并行配额）。  
- **Claude Configuration**：集中定义会话超时、温度、token 限制、重试策略等高阶配置（以文档形式管理）。  
- **Prompt Template Registry**：维护角色级提示词模板的版本与变更历史（模板为契约参考，不包含具体文本）。  
- **Pipeline Orchestration Metadata**：维护 pipeline 定义、任务依赖图、并行安全规则与任务状态模型（状态机定义而非实现）。  
- **Worktree Lifecycle**：定义 worktree 的创建、快照、回滚、清理策略与权限模型（文档化操作步骤）。  
- **Concurrency Safety Engine（概念）**：提供并行判定规则（基于 role access map）与冲突解决策略（优先级、merge policy）。  
- **Observability and Audit Surface**：定义审计日志格式、artifact 索引、变更事件模型与回溯流程。  
- **Evolution Hooks**：定义如何从手工满血流程演进到脚本化、再到 orchestrator（阶段化里程碑与验收标准）。

---

### 3 File Protocol and Artifact Contracts

**文件即协议，必须标准化但不实现细节：**

- **Artifact Metadata**（每个文件头或伴随文件）  
  - **role**：产出角色  
  - **module**：目标模块名  
  - **task-id**：唯一任务标识  
  - **status**：draft | final | reviewed | tested  
  - **timestamp**：UTC 时间戳  
- **命名约定**：`<docs|src>/<category>/<module>.<role>.<task-id>.<status>.md|py`  
- **交付物类型**：spec draft, spec final, interface contract, impl, review report, test results, roadmap entry  
- **变更流程**：任何从 draft→final→reviewed 的状态变更必须在 audit index 中记录并 commit

---

### 4 Worktree and Session Management Principles

- **Worktree per Role**：每角色独立工作目录或隔离视图，包含该角色可读写的 artifact 子集。  
- **Session Binding**：每个终端/Claude 会话绑定到一个 worktree，session 生命周期与 task-id 绑定。  
- **Minimal Local State**：会话内不保留跨任务记忆，所有跨角色信息通过 artifact 文件与 audit index 传递。  
- **Manual Full-Fidelity Mode**：满血 L0 初期以手工方式运行：为每角色打开固定终端标签页并 `cd` 到对应 worktree，按文件协议交付与 commit。  
- **Conflict Handling**：并行写冲突由 Documenter PM 协调，优先保留早期提交并创建合并任务。

---

### 5 Pipelines, Parallelism and Scheduling Rules

**Pipeline primitives**  
- **Task**：最小工作单元，绑定 role、module、task-id、read/write paths。  
- **Dependency Graph**：任务依赖由 artifact read/write 决定，Architect→Coder 等为强依赖。  
- **Parallelism Rule**：若两个任务的写路径无交集且读路径不依赖对方写入，则可并行。  
- **Scheduling Modes**  
  - **Manual Parallel**：用户在多个终端并行启动会话（满血 L0 初期推荐）。  
  - **Advisory Scheduler**：基于 role access map 提供并行建议（后期脚本化）。  
- **Precedence and Escalation**：若并行导致冲突，Documenter PM 发起合并任务并记录决策。

---

### 6 Governance, Onboarding and Evolution Roadmap

**Governance**  
- **Role Manifest**：每个仓库必须包含 role-manifest 文档，列出角色契约与 artifact 类型。  
- **Audit Index**：docs/audit_index.md 记录所有 role 交付的摘要与 task-id。  
- **Reviewer Gate**：任何 impl 合并到主分支前必须有 Reviewer 报告与 Internal Tester 结果。

**Onboarding**  
- **Quickstart Steps**：创建 7 个 worktrees、打开 7 个终端、加载 role-manifest、运行一次 end-to-end 流程。  
- **Checklist**：每个新模块必须完成 Researcher spec draft、External Tester 验证、Architect 接口定义，方可进入实现阶段。

**Evolution Roadmap**  
- **Phase 0** 手工满血 L0：手动会话、worktree、文件协议、audit index。  
- **Phase 1** 半自动化：脚本化 worktree 创建、并行安全检查脚本、prompt 模板 registry（文档化）。  
- **Phase 2** 轻量 orchestrator：任务队列、并行建议、状态机、CI gate（静态检查、未授权 import 检查）。  
- **Phase 3** 完整 orchestrator：可选 Web 控制台、恢复与回滚、细粒度权限、自动并行调度。

---

### 立刻可用的行动项（三条）
1. 在 GitHub 建仓并推送本架构设计文档与 role-manifest（作为元老级文档）。  
2. 本地手动创建 7 个 worktrees，打开 7 个终端标签页，按文件协议跑一次完整 Pipeline，记录痛点到 `docs/project/backlog.md`。  
3. 在 backlog 中列出首批 L0 自动化需求（worktree init、并行安全检查、审查静态检查），作为满血 L0 的首批实现目标。

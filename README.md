# 基于Claude Code Subagent的多角色vibe coding架构

## 一 概要目标与核心原则

- **目标**：保证主 agent 上下文极小且安全，避免上下文窗口爆炸，同时支持可插拔的 subagent 流水线。
- **核心原则**：
  - **主 agent 是唯一调度者**，负责决策与全局目标判断。
  - **subagent 只产出详细 artifact 到外部存储并返回结构化摘要**，不得污染主 agent 上下文。
  - **最小公共协议加可扩展 extra 字段**，保证稳定解析与灵活扩展。
  - **主 agent 以事件日志和任务目标驱动调度**，而非依赖 subagent 的建议。

## 二 最小协议规范

### 核心 summary（所有 subagent 必须返回）

```markdown
## metadata

- action: "generate_impl"             # subagent 执行的动作
- status: "success" | "partial" | "failed" | "blocked"
- artifact_path: "path/to/artifact"   # 详细任务报告存放位置
- file_change_count: 0
- file_extensions:
  - ".py"
  - ".md"
- errors:                             # 简短错误列表或省略
```

### 扩展 metadata（可选，role 特定）

```
## summary

	# 简短任务总结以及关键说明
```

**用途**：供特定 subagent 报告角色相关信息，不作为主调度的必需段落。

## 三 全局状态机

- **状态驱动**：`status` 决定分支方向（success → 下一阶段；partial → 补充；failed → 修复；blocked → 解锁）。
- **目标驱动完成判定**：主 agent 根据 **上下文 + 用户目标** 判断任务是否完成，单个 subagent 无权判定完成。

## 四 主 agent 调度逻辑规范

### 调度输入优先级

1. **action**（必须）  
2. **status**（必须）  
3. **errors**（必须）  
5. **file_extensions**（参考）  
6. **summary**（参考）

### 调度规则风格

- **定义规范，不写死决策**：在主 agent 的提示词中写入流程规范和约束（例如：写完代码必须 review；review 通过后必须测试；测试失败必须修复），但不硬编码 if/else 的具体阈值或策略。
- **决策由主 agent 在运行时推理**：主 agent 根据当前上下文与目标动态选择下一个 subagent 或动作。
- **避免 subagent 建议干扰**：subagent 可以提供 suggestions 但主 agent 默认忽略，除非提示词明确允许采纳建议。

### 示例调度策略（伪逻辑）

- 若 `action = generate_impl` 且 `status = success` 且 `file_extensions` 包含 `.py` → 调用 reviewer。  
- 若 `status = failed` 或 `errors` 非空 → 调用错误处理。  
- 若事件日志显示所有必需阶段已完成且与用户目标匹配 → 标记任务完成，主 agent 输出总结。

## 五 实现与安全注意事项

- **访问权限**：subagent 写入权限受限，主 agent 仅有 agent 调用权限，不加载大文件。

[README.md](https://github.com/user-attachments/files/31510495/README.md)
# diet‑agent
> 面向日常就餐决策场景的智能饮食推荐Agent，基于编排+Worker多智能体协作，实现对话式膳食推荐，内置Trace全链路监控、离线评估体系，打通**使用‑反馈‑标注‑评估‑迭代**完整工程闭环。

## 📖 项目简介
解决用户“不知道吃什么”就餐选择焦虑，结合用户**餐次、心情、场景、健康诉求、菜系、口味、便捷性**7维槽位信息，完成多轮对话饮食推荐。
区别普通Demo级Agent项目，本项目重点落地Agent工程化能力：**全链路可观测、量化评估闭环、LLM全链路降级兜底、会话状态机管理**，并非简单调用大模型做问答。

**业务链路：意图识别 → 槽位澄清 → 标签检索重排 → LLM生成推荐理由返回结果**

- 用户侧：双数据源支持`PUBLIC公共餐食库` / `PERSONAL个人餐食库`，个人菜单CRUD，多轮对话推荐，用户反馈点赞/差评
- 研发侧：Trace链路查看、人工标注、批量离线评估报告；可本地完整运行前端页面演示效果

## ✨ 核心亮点
1. **Orchestrator+Worker多Agent协作架构**
   编排中枢负责会话流转、状态机管控；拆解`IntentAgent`、`ClarifyAgent`、`RecommendResponseAgent`、`EvaluationJudgeAgent`专职Worker；LLM只负责结构化推理，业务决策全部由Java规则层控制，避免大模型自由流转带来不可控风险。

2. **Trace全链路可观测能力**
   基于ThreadLocal实现Trace上下文，记录每轮请求全部事件：Agent入参出参、token消耗、各阶段耗时、路由决策、检索排序结果；落库`diet_request_trace`，支持链路回放排查问题，也是后续评估闭环的数据源。

3. **完整离线评估闭环（本项目最核心工程价值）**
- 数据源复用线上Trace记录，支持人工标注标准Gold‑Label
- 三维加权打分：**后端规则指标60% + LLM‑as‑Judge Judge10% + 用户真实反馈30%**
- 覆盖意图准确率、槽位准确率、token成本、端到端耗时、幻觉检测、多轮一致性等十余项指标，输出百分制评估报告；低分样本驱动Prompt、规则迭代优化，形成「Trace→标注→评估→迭代」闭环。

4. **LLM全链路Fallback降级设计**
   意图识别、槽位澄清、推荐生成各个Agent调用节点，针对超时、JSON解析失败、模型限流异常提供Java规则+模板兜底；**即使大模型完全不可用，业务流程不会中断，依然可以返回可用餐食卡片与模板话术**，规避对话直接报错。

5. **意图识别三层兜底 + 词典约束防幻觉**
   识别6类业务意图，7维槽位抽取；叠加「意图状态矫正、低置信降级、关键词Fallback」三层兜底；槽位输出受数据库词典强约束，禁止模型编造不存在标签，降低幻觉。

6. **混合推荐流水线，抑制餐食幻觉**
   `检索(MySQL JSON_OVERLAPS宽召回) → Java层打分精排 → LLM仅生成推荐理由`；LLM只能基于已检索候选输出，不允许编造不存在餐食。

7. **RiskGuard健康合规守卫**
   意图层+输出层双拦截，识别医疗、极端节食、绝对化健康风险文案，返回保守提示文案，规避Agent输出风险内容。

8. **多轮会话状态管理**
   自研SessionState会话状态机，MySQL持久化结构化槽位、历史推荐ID、会话阶段；多轮对话自动增量合并用户偏好；实现换一批排除历史推荐、会话锁防止并发请求状态错乱。

## 🛠️ 技术栈
| 模块 | 技术选型 |
|---|---|
| 后端 | Java21、SpringBoot 3.3.x |
| Agent框架 | AgentScope‑spring‑boot‑starter 1.0.11 |
| 大模型 | 阿里云百炼 DashScope(qwen‑max / qwen‑turbo) |
| 数据库 | MySQL8.0（JSON_OVERLAPS JSON字段） |
| ORM | MyBatis |
| 前端 | 原生静态页面，打包在resources/static，SPA哈希路由 |

## 📂 数据库表说明
- `diet_sessions`：会话状态，存储多轮累积槽位、会话阶段、历史推荐ID
- `diet_messages`：对话消息流水
- `diet_request_trace`：全链路Trace存储，同时保存人工标注gold label
- `meal_item`：餐食表，区分PUBLIC公共库、PERSONAL用户个人库
- `diet_slot_option`：7维槽位合法标签词典，约束LLM输出，防幻觉
- `recommend_feedback`：用户对推荐结果的反馈数据，用于评估打分

## 🚀 快速启动
### 1. 数据库准备
1. 创建数据库：`diet_db`，排序规则：`utf8mb4_unicode_ci`
2. 执行SQL脚本：`src/main/resources/db/diet_db.sql`，自动建表+初始化种子数据

### 2. 修改配置文件
打开 `src/main/resources/application.yml`
填入阿里云百炼 `agentscope.dashscope.api‑key`
```yaml
agentscope:
  dashscope:
    api‑key: "你的dashscope api‑key"
diet:
  llm:
    main‑model: qwen‑max
    light‑model: qwen‑turbo

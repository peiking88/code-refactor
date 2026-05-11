# 工作摘要

**时间:** 2026-05-11 11:45:00
**版本:** 1.2.0

## 本次升级概要

### v1.2.0 变更（相比 v1.1.0）

1. **新增异常审计方法论**
   - SKILL.md 新增 Exception Audit 章节：三类异常问题的决策框架（该用未用、不该用却用、异常类使用不当）
   - 每类包含操作风险表和决策表，直接指导诊断和修复

2. **语言级 Before/After 示例**
   - references/python.md：4 组异常示例（文件读取、异常驱动控制流、吞没异常、错误类型/丢失原因链）
   - references/typescript.md：4 组异常示例（网络请求、异常驱动控制流、不必要的 try/catch、错误类型/丢失原因链）
   - references/java.md：4 组异常示例（文件读取、空指针替代、批量吞没异常、错误类型/丢失原因链）

3. **技术债框架增强**
   - techdebt.md 债务清单新增 Exception 维度：裸 catch 计数、泛型异常计数、未处理失败点计数
   - 质量门禁新增 3 项异常检查：裸/泛型 catch 零容忍、吞没异常零容忍、I/O 网络路径错误边界

4. **异常反模式条目**
   - anti-patterns.md 新增 7 条异常反模式：裸 catch、异常驱动控制流、过度宽泛 catch、错误异常类型、丢失原因链、构造函数吞没、无退避重试

5. **触发词更新**
   - SKILL.md description 新增异常审计触发词：audit exception handling、check exception usage、find missing try/catch

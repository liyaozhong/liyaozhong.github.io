---
layout: post
title: "Multi-Agent Systems"
subtitle: "到底要不要Multi-agent?"
tags: [RAG, Vector Searh]
comments: false
---

最近读了两篇关于多代理系统（Multi-Agent Systems, MAS）的文章，视角不同但都很有启发。以下是原文链接，建议大家直接阅读原文：

- [Cognition.ai: Don’t Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents#a-theory-of-building-long-running-agents)
- [Anthropic: How we built our multi-agent research system ](https://www.anthropic.com/engineering/built-multi-agent-research-system)

两篇文章看似观点冲突，但其实各有侧重，下面我来简单总结一下。

## 文章总结

### Cognition.ai: Don’t Build Multi-Agents

这篇文章的核心观点是：在需要共享上下文和处理依赖关系的任务中，多代理系统可能会导致不可靠的结果。他们用一个“Flappy Bird 克隆”任务举例，指出子代理可能因误解任务而生成不一致的输出。文章认为，目前的技术条件下，多代理系统在上下文共享和协调上存在根本性挑战，建议开发者谨慎使用。

### Anthropic: How we built our multi-agent research system

Anthropic 的文章则聚焦于多代理系统在研究场景中的应用，详细介绍了他们如何利用 Claude 构建一个高效的多代理研究系统。以下是几个重点内容：

- **系统目标**：提升研究效率，特别是在信息搜索和广度优先的查询任务中。
- **架构设计**：采用 orchestrator-worker 模式，由一个领导代理（orchestrator）协调多个子代理（workers）并行执行任务。
- **技术亮点**：
    - 集成了网页搜索和 Google Workspace 工具，支持动态、多步骤的信息检索。
    - 通过任务委派和并行工具调用，研究时间最多可减少 90%。
    - Claude Opus 4 和 Sonnet 4 的组合在广度优先任务中比单代理系统性能提升了 90.2%。
- **局限性**：文章也提到，在编码等需要深度上下文和依赖管理的任务中，多代理系统的表现不如单代理。
- **开发方法**：强调评估驱动开发（Evaluation-Driven Development），通过不断测试优化系统。

Anthropic 的实践表明，多代理系统在并行处理和信息密集型任务中有显著优势，但并非万能。

## 我的看法：不冲突，任务决定一切

我觉得这两篇文章并不矛盾，而是从不同角度探讨了多代理架构的适用性。关键在于任务类型：

- **广度优先任务**（如信息搜索）：多代理并行处理效率高，Anthropic 的实践证明了这一点。
- **深度优先任务**（如编码）：需要共享上下文和强一致性，多代理容易出问题，Cognition.ai 的担忧很有道理。

两篇文章其实有共识：编码这类任务因上下文依赖重，目前的多代理架构不太适合。归根结底，用什么架构，得看具体需求。

## 读后感

1. **多代理要干高价值的事**
    
    多代理系统适合复杂、可拆分的任务，尤其是能发挥并行优势的场景，比如大规模信息处理。简单任务用它反而浪费资源。
    
2. **Prompt Engineering 是核心**
    
    不管多代理还是单代理，Prompt 设计直接决定效果。多代理协作时，任务分配的 Prompt 尤为关键，得清晰到让每个代理都知道自己干啥。
    
3. **架构和评估的重要性**
    
    复杂任务需要好的系统架构和评估方法。Anthropic 的 orchestrator-worker 模式和评估驱动开发给了我很大启发，值得借鉴。
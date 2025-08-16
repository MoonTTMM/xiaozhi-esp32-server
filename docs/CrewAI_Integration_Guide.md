# 🤖 xiaozhi-server 集成 crewAI 多智能体系统最佳实践指南

## 📋 目录

1. [概述与背景](#概述与背景)
2. [架构设计方案](#架构设计方案)
3. [集成策略](#集成策略)
4. [详细实现指南](#详细实现指南)
5. [配置管理](#配置管理)
6. [性能优化](#性能优化)
7. [测试与部署](#测试与部署)
8. [最佳实践](#最佳实践)
9. [故障排除](#故障排除)

---

## 🌟 概述与背景

### 为什么集成 crewAI？

crewAI 是一个强大的多智能体协作框架，能够让多个AI智能体协同工作来解决复杂任务。在 xiaozhi-server 中集成 crewAI 可以：

- **增强复杂任务处理能力**: 通过多个专业化agent协作
- **提升决策质量**: 不同视角的agent提供更全面的分析
- **支持复杂工作流**: 处理需要多步骤、多角色的任务
- **保持架构灵活性**: 可以根据任务复杂度动态选择单agent或多agent模式

### 集成原则

1. **向后兼容**: 保持现有单agent模式的正常工作
2. **智能路由**: 根据任务复杂度自动选择处理模式
3. **性能优先**: 确保实时语音交互的流畅性
4. **配置灵活**: 支持动态配置不同的agent团队

---

## 🏗️ 架构设计方案

### 1. 整体架构图

```
xiaozhi-server (现有架构)
├── core/providers/llm/         # 现有LLM providers
├── core/providers/crewai/      # 新增: crewAI多agent支持
│   ├── base.py                 # crewAI基类
│   ├── multi_agent.py          # 多agent实现
│   └── agents/                 # 预定义agent配置
└── core/providers/tools/
    └── crewai_tools/           # crewAI专用工具
        ├── crewai_executor.py  # crewAI执行器
        └── agent_coordinator.py # agent协调器
```

### 2. 核心组件设计

#### MultiAgent Provider

```python
# core/providers/crewai/multi_agent.py
class MultiAgentProvider(LLMProviderBase):
    """crewAI多智能体LLM提供者"""
    
    def __init__(self, config):
        self.crew_configs = config.get("crews", {})
        self.default_crew = config.get("default_crew", "general")
        self.crews = {}
        self.initialize_crews()
    
    async def response(self, session_id, dialogue, **kwargs):
        """智能路由到合适的crew处理"""
        task_complexity = self.analyze_task_complexity(dialogue)
        
        if task_complexity > self.crew_threshold:
            return await self.multi_agent_response(session_id, dialogue)
        else:
            return await self.single_agent_response(session_id, dialogue)
```

#### Agent协调器

```python
# core/providers/tools/crewai_tools/agent_coordinator.py
class AgentCoordinator:
    """管理多个agent的协调和状态同步"""
    
    def __init__(self, config):
        self.agents = {}
        self.active_crews = {}
        self.task_queue = asyncio.Queue()
    
    async def execute_crew_task(self, crew_name, task, context):
        """执行crew任务"""
        crew = self.get_or_create_crew(crew_name)
        
        # 设置任务上下文
        crew.context = context
        
        # 异步执行任务
        result = await asyncio.create_task(
            crew.kickoff_async(task)
        )
        
        return result
```

### 3. 集成点设计

#### 在UnifiedToolHandler中集成

```python
# core/providers/tools/unified_tool_handler.py (修改)
class UnifiedToolHandler:
    def __init__(self, conn):
        # ... 现有代码 ...
        self.crewai_executor = CrewAIExecutor(conn.config) if self._should_enable_crewai() else None
    
    async def handle_llm_function_call(self, conn, function_call_data):
        function_name = function_call_data.get("name")
        
        # 检查是否需要crewAI处理
        if self._requires_multi_agent(function_name, function_call_data):
            return await self.crewai_executor.execute(conn, function_call_data)
        
        # 现有的单agent处理逻辑
        return await self._handle_single_agent_call(conn, function_call_data)
```

---

## 🎯 集成策略

### 1. 渐进式集成策略

#### 阶段一：基础框架集成
- 创建crewAI Provider基类
- 实现基本的多agent调用机制
- 保持与现有系统的完全兼容

#### 阶段二：智能路由实现
- 实现任务复杂度分析
- 添加智能路由逻辑
- 支持动态切换单/多agent模式

#### 阶段三：高级功能扩展
- 支持自定义agent团队配置
- 实现agent间状态共享
- 添加performance监控和优化

### 2. 任务路由策略

```python
class TaskRouter:
    """任务路由器：决定使用单agent还是多agent"""
    
    MULTI_AGENT_KEYWORDS = [
        "分析", "规划", "研究", "比较", "评估", 
        "设计", "策略", "多步骤", "协作"
    ]
    
    def should_use_multi_agent(self, user_input: str, context: dict) -> bool:
        """判断是否需要使用多agent模式"""
        
        # 1. 关键词检测
        if any(keyword in user_input for keyword in self.MULTI_AGENT_KEYWORDS):
            return True
        
        # 2. 输入长度检测
        if len(user_input) > 100:  # 复杂任务通常描述较长
            return True
        
        # 3. 历史复杂度检测
        if context.get("recent_complex_tasks", 0) > 2:
            return True
        
        # 4. 显式请求检测
        if "团队" in user_input or "多个" in user_input:
            return True
        
        return False
```

---

## 💻 详细实现指南

### 1. 环境准备

#### 安装依赖

```bash
# 在xiaozhi-server根目录执行
pip install crewai crewai-tools

# 添加到requirements.txt
echo "crewai>=0.22.0" >> requirements.txt
echo "crewai-tools>=0.12.0" >> requirements.txt
```

#### 目录结构创建

```bash
# 创建crewAI相关目录
mkdir -p core/providers/crewai/agents
mkdir -p core/providers/tools/crewai_tools
mkdir -p config/crewai
```

### 2. 核心实现代码

#### MultiAgent Provider基类

```python
# core/providers/crewai/base.py
from abc import ABC, abstractmethod
from typing import Dict, List, Any, Optional
from crewai import Agent, Task, Crew
from core.providers.llm.base import LLMProviderBase

class CrewAIProviderBase(LLMProviderBase):
    """crewAI多智能体基类"""
    
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.crews: Dict[str, Crew] = {}
        self.agents: Dict[str, Agent] = {}
        self.active_sessions: Dict[str, str] = {}  # session_id -> crew_name
    
    @abstractmethod
    def create_agents(self) -> Dict[str, Agent]:
        """创建智能体"""
        pass
    
    @abstractmethod
    def create_crews(self) -> Dict[str, Crew]:
        """创建crew团队"""
        pass
    
    async def response(self, session_id: str, dialogue: List[Dict], **kwargs):
        """主要响应方法"""
        user_message = dialogue[-1]["content"] if dialogue else ""
        
        # 选择合适的crew
        crew_name = self.select_crew(user_message, **kwargs)
        crew = self.crews.get(crew_name)
        
        if not crew:
            raise ValueError(f"Crew '{crew_name}' not found")
        
        # 创建任务
        task = Task(
            description=user_message,
            agent=list(crew.agents)[0],  # 指定主要负责的agent
            expected_output="对用户请求的详细回复"
        )
        
        # 执行任务
        try:
            result = await self.execute_crew_task(crew, task, session_id)
            return self.format_response(result)
        except Exception as e:
            return f"多智能体处理出错: {str(e)}"
    
    def select_crew(self, user_message: str, **kwargs) -> str:
        """选择合适的crew"""
        # 实现智能选择逻辑
        if "分析" in user_message or "研究" in user_message:
            return "research_crew"
        elif "创意" in user_message or "设计" in user_message:
            return "creative_crew"
        else:
            return "general_crew"
    
    async def execute_crew_task(self, crew: Crew, task: Task, session_id: str):
        """异步执行crew任务"""
        import asyncio
        
        def run_crew():
            return crew.kickoff()
        
        # 在线程池中执行，避免阻塞
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(None, run_crew)
        return result
```

#### 具体MultiAgent实现

```python
# core/providers/crewai/multi_agent.py
from crewai import Agent, Task, Crew
from crewai_tools import SerperDevTool, WebsiteSearchTool
from core.providers.crewai.base import CrewAIProviderBase

class MultiAgentProvider(CrewAIProviderBase):
    """多智能体提供者实现"""
    
    def __init__(self, config):
        super().__init__(config)
        self.agents = self.create_agents()
        self.crews = self.create_crews()
    
    def create_agents(self):
        """创建专业化智能体"""
        
        # 获取LLM配置
        llm_config = self.config.get("base_llm", {})
        
        # 研究分析师
        researcher = Agent(
            role='研究分析师',
            goal='深入研究和分析用户提出的问题',
            backstory="""你是一位经验丰富的研究分析师，擅长收集信息、
            分析数据和提供深入见解。你总是确保信息的准确性和全面性。""",
            verbose=True,
            allow_delegation=False,
            tools=[SerperDevTool(), WebsiteSearchTool()],
            llm=self._create_llm_instance(llm_config)
        )
        
        # 创意策划师
        creative_planner = Agent(
            role='创意策划师',
            goal='为用户需求提供创新和创意的解决方案',
            backstory="""你是一位富有创意的策划师，善于跳出常规思维，
            为复杂问题提供创新解决方案。你结合艺术和科学的思维方式。""",
            verbose=True,
            allow_delegation=False,
            llm=self._create_llm_instance(llm_config)
        )
        
        # 技术专家
        tech_expert = Agent(
            role='技术专家',
            goal='提供技术实现方案和最佳实践建议',
            backstory="""你是一位资深技术专家，在软件开发、系统架构
            和技术选型方面有丰富经验。你善于将复杂技术转化为易懂的建议。""",
            verbose=True,
            allow_delegation=False,
            llm=self._create_llm_instance(llm_config)
        )
        
        # 综合顾问
        advisor = Agent(
            role='综合顾问',
            goal='综合各方面信息，提供最终建议和决策支持',
            backstory="""你是一位经验丰富的综合顾问，善于整合不同专家的
            意见，权衡利弊，为用户提供最适合的建议和行动方案。""",
            verbose=True,
            allow_delegation=True,
            llm=self._create_llm_instance(llm_config)
        )
        
        return {
            "researcher": researcher,
            "creative_planner": creative_planner,
            "tech_expert": tech_expert,
            "advisor": advisor
        }
    
    def create_crews(self):
        """创建不同类型的crew团队"""
        
        # 研究分析团队
        research_crew = Crew(
            agents=[self.agents["researcher"], self.agents["advisor"]],
            verbose=True,
            process="sequential"
        )
        
        # 创意策划团队
        creative_crew = Crew(
            agents=[self.agents["creative_planner"], self.agents["advisor"]],
            verbose=True,
            process="sequential"
        )
        
        # 技术咨询团队
        tech_crew = Crew(
            agents=[self.agents["tech_expert"], self.agents["advisor"]],
            verbose=True,
            process="sequential"
        )
        
        # 综合团队（所有专家）
        full_crew = Crew(
            agents=list(self.agents.values()),
            verbose=True,
            process="hierarchical",
            manager_llm=self._create_llm_instance(self.config.get("manager_llm", {}))
        )
        
        return {
            "research_crew": research_crew,
            "creative_crew": creative_crew,
            "tech_crew": tech_crew,
            "general_crew": full_crew
        }
    
    def _create_llm_instance(self, llm_config):
        """创建LLM实例"""
        # 根据配置创建相应的LLM实例
        from core.utils import llm as llm_utils
        
        llm_type = llm_config.get("type", "openai")
        return llm_utils.create_instance(llm_type, llm_config)
    
    def select_crew(self, user_message: str, **kwargs) -> str:
        """智能选择crew"""
        message_lower = user_message.lower()
        
        # 研究分析类任务
        research_keywords = ["分析", "研究", "调查", "数据", "报告", "市场"]
        if any(keyword in message_lower for keyword in research_keywords):
            return "research_crew"
        
        # 创意策划类任务
        creative_keywords = ["创意", "设计", "策划", "创新", "方案", "营销"]
        if any(keyword in message_lower for keyword in creative_keywords):
            return "creative_crew"
        
        # 技术咨询类任务
        tech_keywords = ["技术", "开发", "编程", "系统", "架构", "算法"]
        if any(keyword in message_lower for keyword in tech_keywords):
            return "tech_crew"
        
        # 复杂任务或明确要求团队协作
        if len(user_message) > 100 or "团队" in message_lower:
            return "general_crew"
        
        # 默认使用研究团队
        return "research_crew"
```

#### CrewAI工具执行器

```python
# core/providers/tools/crewai_tools/crewai_executor.py
import asyncio
from typing import Dict, Any
from core.providers.tools.base.tool_executor import ToolExecutor
from plugins_func.register import ActionResponse, Action

class CrewAIExecutor(ToolExecutor):
    """crewAI多智能体工具执行器"""
    
    def __init__(self, config):
        super().__init__()
        self.config = config
        self.multi_agent_provider = None
        self.initialize_provider()
    
    def initialize_provider(self):
        """初始化多智能体提供者"""
        try:
            from core.providers.crewai.multi_agent import MultiAgentProvider
            self.multi_agent_provider = MultiAgentProvider(
                self.config.get("crewai", {})
            )
        except Exception as e:
            self.logger.error(f"初始化crewAI提供者失败: {e}")
    
    async def execute(self, conn, function_call_data: Dict[str, Any]) -> ActionResponse:
        """执行crewAI多智能体任务"""
        try:
            function_name = function_call_data.get("name", "")
            arguments = function_call_data.get("arguments", "{}")
            
            # 解析参数
            import json
            try:
                args = json.loads(arguments) if isinstance(arguments, str) else arguments
            except json.JSONDecodeError:
                args = {"task": arguments}
            
            # 构建任务描述
            task_description = args.get("task", args.get("query", ""))
            if not task_description:
                return ActionResponse(
                    action=Action.RESPONSE,
                    response="请提供具体的任务描述"
                )
            
            # 构建对话上下文
            dialogue = []
            if hasattr(conn, 'dialogue') and conn.dialogue:
                dialogue = conn.dialogue.get_llm_dialogue()
            
            # 添加当前任务
            dialogue.append({
                "role": "user",
                "content": task_description
            })
            
            # 执行多智能体任务
            self.logger.info(f"执行crewAI任务: {task_description}")
            
            result = await self.multi_agent_provider.response(
                session_id=conn.session_id,
                dialogue=dialogue,
                context=args
            )
            
            return ActionResponse(
                action=Action.RESPONSE,
                response=result
            )
            
        except Exception as e:
            self.logger.error(f"crewAI执行失败: {e}")
            return ActionResponse(
                action=Action.ERROR,
                response=f"多智能体协作出现错误: {str(e)}"
            )
    
    def get_functions(self) -> Dict[str, Dict]:
        """获取crewAI相关的函数定义"""
        return {
            "multi_agent_analysis": {
                "name": "multi_agent_analysis",
                "description": "使用多个AI智能体协作来分析复杂问题和提供综合解决方案",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "task": {
                            "type": "string",
                            "description": "需要多智能体协作完成的任务描述"
                        },
                        "crew_type": {
                            "type": "string",
                            "description": "指定使用的团队类型：research_crew(研究分析), creative_crew(创意策划), tech_crew(技术咨询), general_crew(综合团队)",
                            "enum": ["research_crew", "creative_crew", "tech_crew", "general_crew"]
                        },
                        "priority": {
                            "type": "string",
                            "description": "任务优先级：high(高), medium(中), low(低)",
                            "enum": ["high", "medium", "low"]
                        }
                    },
                    "required": ["task"]
                }
            },
            "collaborative_planning": {
                "name": "collaborative_planning",
                "description": "多智能体协作制定计划和策略",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "goal": {
                            "type": "string",
                            "description": "要实现的目标"
                        },
                        "constraints": {
                            "type": "array",
                            "items": {"type": "string"},
                            "description": "约束条件列表"
                        },
                        "timeline": {
                            "type": "string",
                            "description": "时间要求"
                        }
                    },
                    "required": ["goal"]
                }
            }
        }
```

### 3. 配置集成

#### crewAI配置文件

```yaml
# config/crewai/crews.yaml
crewai:
  enabled: true
  
  # 基础LLM配置
  base_llm:
    type: "openai"
    api_key: "your-openai-api-key"
    model: "gpt-4-turbo-preview"
    temperature: 0.7
  
  # 管理者LLM配置（用于hierarchical模式）
  manager_llm:
    type: "openai"
    api_key: "your-openai-api-key"
    model: "gpt-4-turbo-preview"
    temperature: 0.3
  
  # 任务路由配置
  routing:
    complexity_threshold: 0.7
    multi_agent_keywords:
      - "分析"
      - "研究"
      - "规划"
      - "设计"
      - "策略"
      - "协作"
      - "团队"
    
    # 自动路由规则
    auto_routing_rules:
      - pattern: ".*分析.*市场.*"
        crew: "research_crew"
      - pattern: ".*设计.*方案.*"
        crew: "creative_crew"
      - pattern: ".*技术.*实现.*"
        crew: "tech_crew"
  
  # Agent配置
  agents:
    researcher:
      role: "研究分析师"
      max_execution_time: 300
      tools:
        - "SerperDevTool"
        - "WebsiteSearchTool"
    
    creative_planner:
      role: "创意策划师"
      max_execution_time: 240
      tools: []
    
    tech_expert:
      role: "技术专家"
      max_execution_time: 180
      tools: []
    
    advisor:
      role: "综合顾问"
      max_execution_time: 120
      tools: []
  
  # Crew配置
  crews:
    research_crew:
      agents: ["researcher", "advisor"]
      process: "sequential"
      max_execution_time: 600
    
    creative_crew:
      agents: ["creative_planner", "advisor"]
      process: "sequential"
      max_execution_time: 480
    
    tech_crew:
      agents: ["tech_expert", "advisor"]
      process: "sequential"
      max_execution_time: 360
    
    general_crew:
      agents: ["researcher", "creative_planner", "tech_expert", "advisor"]
      process: "hierarchical"
      max_execution_time: 900
  
  # 性能配置
  performance:
    max_concurrent_crews: 3
    task_timeout: 900
    agent_timeout: 300
    enable_caching: true
    cache_ttl: 3600
```

#### 主配置文件修改

```yaml
# config.yaml (添加crewAI配置)
selected_module:
  LLM: "LLM_crewai"  # 新增crewAI LLM模块
  # ... 其他配置保持不变

LLM:
  # 现有LLM配置保持不变
  LLM_openai:
    type: "openai"
    # ...
  
  # 新增crewAI配置
  LLM_crewai:
    type: "crewai_multi_agent"
    enabled: true
    config_file: "config/crewai/crews.yaml"
    fallback_to_single: true  # 出错时回退到单agent模式
    fallback_llm: "LLM_openai"

# Intent配置修改，支持crewAI路由
Intent:
  Intent_function_call:
    type: "function_call"
    functions:
      - "multi_agent_analysis"
      - "collaborative_planning"
      # ... 其他现有函数
```

---

## ⚡ 性能优化

### 1. 异步执行优化

```python
# 优化的异步执行方案
class OptimizedCrewAIExecutor:
    """优化的crewAI执行器"""
    
    def __init__(self, config):
        self.config = config
        self.crew_pool = {}  # crew实例池
        self.task_queue = asyncio.Queue(maxsize=100)
        self.worker_pool = []
        self.initialize_workers()
    
    def initialize_workers(self):
        """初始化工作线程池"""
        max_workers = self.config.get("max_concurrent_crews", 3)
        for i in range(max_workers):
            worker = asyncio.create_task(self.crew_worker(f"worker-{i}"))
            self.worker_pool.append(worker)
    
    async def crew_worker(self, worker_name: str):
        """crew工作线程"""
        while True:
            try:
                task_data = await self.task_queue.get()
                if task_data is None:  # 停止信号
                    break
                
                crew_name = task_data["crew_name"]
                task = task_data["task"]
                future = task_data["future"]
                
                try:
                    crew = self.get_or_create_crew(crew_name)
                    result = await self.execute_with_timeout(crew, task)
                    future.set_result(result)
                except Exception as e:
                    future.set_exception(e)
                finally:
                    self.task_queue.task_done()
                    
            except Exception as e:
                self.logger.error(f"Worker {worker_name} error: {e}")
    
    async def execute_with_timeout(self, crew, task):
        """带超时的执行"""
        timeout = self.config.get("task_timeout", 900)
        
        try:
            result = await asyncio.wait_for(
                self.run_crew_task(crew, task),
                timeout=timeout
            )
            return result
        except asyncio.TimeoutError:
            raise Exception(f"任务执行超时 ({timeout}秒)")
    
    def get_or_create_crew(self, crew_name: str):
        """获取或创建crew实例（带缓存）"""
        if crew_name not in self.crew_pool:
            self.crew_pool[crew_name] = self.create_crew(crew_name)
        return self.crew_pool[crew_name]
```

### 2. 缓存策略

```python
# 智能缓存机制
class CrewAICache:
    """crewAI结果缓存"""
    
    def __init__(self, config):
        self.config = config
        self.cache = {}
        self.cache_ttl = config.get("cache_ttl", 3600)
        self.max_cache_size = config.get("max_cache_size", 1000)
    
    def get_cache_key(self, task_description: str, crew_name: str) -> str:
        """生成缓存键"""
        import hashlib
        content = f"{task_description}:{crew_name}"
        return hashlib.md5(content.encode()).hexdigest()
    
    async def get(self, task_description: str, crew_name: str):
        """获取缓存结果"""
        key = self.get_cache_key(task_description, crew_name)
        
        if key in self.cache:
            entry = self.cache[key]
            if time.time() - entry["timestamp"] < self.cache_ttl:
                return entry["result"]
            else:
                del self.cache[key]
        
        return None
    
    async def set(self, task_description: str, crew_name: str, result: str):
        """设置缓存结果"""
        key = self.get_cache_key(task_description, crew_name)
        
        # LRU清理
        if len(self.cache) >= self.max_cache_size:
            oldest_key = min(self.cache.keys(), 
                           key=lambda k: self.cache[k]["timestamp"])
            del self.cache[oldest_key]
        
        self.cache[key] = {
            "result": result,
            "timestamp": time.time()
        }
```

---

## 🧪 测试与部署

### 1. 单元测试

```python
# tests/test_crewai_integration.py
import pytest
import asyncio
from unittest.mock import Mock, patch
from core.providers.crewai.multi_agent import MultiAgentProvider

class TestCrewAIIntegration:
    """crewAI集成测试"""
    
    @pytest.fixture
    def config(self):
        return {
            "base_llm": {
                "type": "openai",
                "api_key": "test-key",
                "model": "gpt-3.5-turbo"
            },
            "crews": {
                "research_crew": {
                    "agents": ["researcher", "advisor"]
                }
            }
        }
    
    @pytest.fixture
    def provider(self, config):
        return MultiAgentProvider(config)
    
    @pytest.mark.asyncio
    async def test_basic_response(self, provider):
        """测试基本响应功能"""
        dialogue = [
            {"role": "user", "content": "请分析一下AI市场的发展趋势"}
        ]
        
        with patch.object(provider, 'execute_crew_task') as mock_execute:
            mock_execute.return_value = "AI市场分析报告..."
            
            result = await provider.response("test-session", dialogue)
            
            assert result is not None
            assert "AI市场" in result
            mock_execute.assert_called_once()
    
    def test_crew_selection(self, provider):
        """测试crew选择逻辑"""
        # 研究类任务
        crew = provider.select_crew("请分析市场数据")
        assert crew == "research_crew"
        
        # 创意类任务
        crew = provider.select_crew("设计一个营销方案")
        assert crew == "creative_crew"
        
        # 技术类任务
        crew = provider.select_crew("如何实现这个算法")
        assert crew == "tech_crew"
    
    @pytest.mark.asyncio
    async def test_error_handling(self, provider):
        """测试错误处理"""
        dialogue = [{"role": "user", "content": "测试错误"}]
        
        with patch.object(provider, 'execute_crew_task', side_effect=Exception("测试错误")):
            result = await provider.response("test-session", dialogue)
            assert "错误" in result
```

### 2. 集成测试

```python
# tests/test_crewai_end_to_end.py
import pytest
from core.connection import ConnectionHandler
from core.providers.tools.crewai_tools.crewai_executor import CrewAIExecutor

class TestCrewAIEndToEnd:
    """端到端集成测试"""
    
    @pytest.mark.asyncio
    async def test_full_integration_flow(self):
        """测试完整集成流程"""
        # 模拟配置
        config = {
            "crewai": {
                "enabled": True,
                "base_llm": {"type": "mock"}
            }
        }
        
        # 创建模拟连接
        conn = Mock()
        conn.session_id = "test-session"
        conn.config = config
        
        # 创建执行器
        executor = CrewAIExecutor(config)
        
        # 测试函数调用
        function_call_data = {
            "name": "multi_agent_analysis",
            "arguments": '{"task": "分析AI发展趋势"}'
        }
        
        with patch.object(executor.multi_agent_provider, 'response') as mock_response:
            mock_response.return_value = "详细的AI趋势分析..."
            
            result = await executor.execute(conn, function_call_data)
            
            assert result.action == "RESPONSE"
            assert "AI趋势" in result.response
```

### 3. 性能测试

```bash
# 性能测试脚本
# tests/performance/test_crewai_performance.py

import time
import asyncio
import pytest
from concurrent.futures import ThreadPoolExecutor

class TestCrewAIPerformance:
    """性能测试"""
    
    @pytest.mark.asyncio
    async def test_concurrent_requests(self):
        """测试并发请求性能"""
        async def make_request():
            # 模拟crewAI请求
            start_time = time.time()
            await asyncio.sleep(0.1)  # 模拟处理时间
            return time.time() - start_time
        
        # 并发测试
        tasks = [make_request() for _ in range(10)]
        results = await asyncio.gather(*tasks)
        
        # 性能断言
        avg_time = sum(results) / len(results)
        assert avg_time < 1.0  # 平均响应时间小于1秒
        
    def test_memory_usage(self):
        """测试内存使用"""
        import psutil
        import gc
        
        process = psutil.Process()
        initial_memory = process.memory_info().rss
        
        # 创建多个crew实例
        crews = []
        for i in range(100):
            crew = Mock()  # 模拟crew
            crews.append(crew)
        
        final_memory = process.memory_info().rss
        memory_increase = final_memory - initial_memory
        
        # 清理
        del crews
        gc.collect()
        
        # 内存增长应该在合理范围内
        assert memory_increase < 100 * 1024 * 1024  # 小于100MB
```

---

## 🎯 最佳实践

### 1. Agent设计最佳实践

```python
# 最佳实践：Agent角色定义
class AgentRoleDesigner:
    """Agent角色设计器"""
    
    @staticmethod
    def create_specialized_agent(role_type: str, domain: str, llm_config: dict):
        """创建专业化agent"""
        
        role_templates = {
            "researcher": {
                "role": f"{domain}研究专家",
                "goal": f"深入研究{domain}领域的问题并提供专业分析",
                "backstory": f"""你是一位在{domain}领域有着丰富经验的研究专家。
                你善于收集和分析相关信息，能够发现其他人容易忽略的重要细节。
                你的分析总是基于事实和数据，具有很高的可信度。"""
            },
            
            "creative": {
                "role": f"{domain}创意顾问",
                "goal": f"为{domain}相关问题提供创新解决方案",
                "backstory": f"""你是一位富有创意的{domain}顾问，善于跳出传统思维，
                为复杂问题提供独特的视角和解决方案。你结合理性分析和感性洞察，
                能够产生既实用又有创新性的想法。"""
            },
            
            "executor": {
                "role": f"{domain}执行专家",
                "goal": f"将{domain}相关的计划转化为可执行的具体步骤",
                "backstory": f"""你是一位经验丰富的{domain}执行专家，擅长将抽象的想法
                转化为具体的行动计划。你考虑问题全面，能够预见执行过程中的潜在问题
                并提前准备解决方案。"""
            }
        }
        
        template = role_templates.get(role_type, role_templates["researcher"])
        
        return Agent(
            role=template["role"],
            goal=template["goal"],
            backstory=template["backstory"],
            verbose=True,
            allow_delegation=False,
            llm=create_llm_instance(llm_config)
        )
```

### 2. 任务分解策略

```python
class TaskDecomposer:
    """任务分解器"""
    
    def decompose_complex_task(self, task_description: str) -> List[Dict]:
        """将复杂任务分解为子任务"""
        
        # 使用LLM分析任务复杂度
        complexity_analysis = self.analyze_task_complexity(task_description)
        
        if complexity_analysis["complexity"] < 0.5:
            return [{"description": task_description, "agent": "general"}]
        
        # 复杂任务分解
        subtasks = []
        
        # 1. 信息收集阶段
        if complexity_analysis["requires_research"]:
            subtasks.append({
                "description": f"收集关于'{task_description}'的相关信息和数据",
                "agent": "researcher",
                "priority": "high"
            })
        
        # 2. 分析阶段
        subtasks.append({
            "description": f"分析'{task_description}'的核心要求和约束条件",
            "agent": "analyst",
            "priority": "high"
        })
        
        # 3. 方案设计阶段
        if complexity_analysis["requires_creativity"]:
            subtasks.append({
                "description": f"为'{task_description}'设计创新解决方案",
                "agent": "creative",
                "priority": "medium"
            })
        
        # 4. 实施规划阶段
        subtasks.append({
            "description": f"制定'{task_description}'的具体实施计划",
            "agent": "executor",
            "priority": "medium"
        })
        
        return subtasks
```

### 3. 错误恢复策略

```python
class CrewAIErrorRecovery:
    """crewAI错误恢复策略"""
    
    def __init__(self, config):
        self.config = config
        self.fallback_llm = config.get("fallback_llm", "LLM_openai")
        self.max_retries = config.get("max_retries", 3)
    
    async def execute_with_recovery(self, crew, task, session_id):
        """带错误恢复的执行"""
        
        for attempt in range(self.max_retries):
            try:
                result = await self.execute_crew_task(crew, task, session_id)
                return result
                
            except Exception as e:
                self.logger.warning(f"Attempt {attempt + 1} failed: {e}")
                
                if attempt < self.max_retries - 1:
                    # 尝试恢复策略
                    if "timeout" in str(e).lower():
                        # 超时错误：简化任务
                        task = self.simplify_task(task)
                    elif "api" in str(e).lower():
                        # API错误：切换到备用LLM
                        crew = self.switch_to_fallback_llm(crew)
                    else:
                        # 其他错误：等待后重试
                        await asyncio.sleep(2 ** attempt)
                else:
                    # 最后尝试：回退到单agent模式
                    return await self.fallback_to_single_agent(task, session_id)
    
    async def fallback_to_single_agent(self, task, session_id):
        """回退到单agent模式"""
        try:
            from core.utils import llm as llm_utils
            fallback_llm = llm_utils.create_instance(
                self.fallback_llm, 
                self.config["LLM"][self.fallback_llm]
            )
            
            dialogue = [{"role": "user", "content": task.description}]
            result = await fallback_llm.response(session_id, dialogue)
            
            return f"[单智能体模式] {result}"
            
        except Exception as e:
            return f"很抱歉，处理您的请求时遇到了问题：{str(e)}"
```

---

## 🔧 故障排除

### 常见问题及解决方案

#### 1. crewAI依赖安装问题

```bash
# 问题：安装crewai时出现依赖冲突
# 解决方案：使用虚拟环境
python -m venv crewai_env
source crewai_env/bin/activate  # Linux/Mac
# 或 crewai_env\Scripts\activate  # Windows

pip install crewai crewai-tools
```

#### 2. 内存使用过高

```python
# 问题：多个crew同时运行导致内存不足
# 解决方案：实现crew实例池和限制

class CrewResourceManager:
    def __init__(self, max_crews=3):
        self.max_crews = max_crews
        self.active_crews = {}
        self.crew_semaphore = asyncio.Semaphore(max_crews)
    
    async def acquire_crew(self, crew_name):
        await self.crew_semaphore.acquire()
        # 获取或创建crew实例
        return self.get_crew(crew_name)
    
    def release_crew(self, crew_name):
        # 清理crew状态
        if crew_name in self.active_crews:
            self.active_crews[crew_name].cleanup()
        self.crew_semaphore.release()
```

#### 3. 响应时间过长

```python
# 问题：crewAI响应时间超过实时语音交互要求
# 解决方案：混合模式 + 超时控制

class HybridLLMProvider:
    def __init__(self, config):
        self.single_llm = create_single_llm(config)
        self.multi_agent_llm = create_multi_agent_llm(config)
        self.quick_response_threshold = config.get("quick_response_threshold", 5)
    
    async def response(self, session_id, dialogue, **kwargs):
        # 快速路径：简单问题用单LLM
        if self.is_simple_query(dialogue[-1]["content"]):
            return await self.single_llm.response(session_id, dialogue)
        
        # 复杂路径：尝试多agent处理
        try:
            result = await asyncio.wait_for(
                self.multi_agent_llm.response(session_id, dialogue),
                timeout=self.quick_response_threshold
            )
            return result
        except asyncio.TimeoutError:
            # 超时回退到单LLM
            return await self.single_llm.response(session_id, dialogue)
```

#### 4. Agent协作失效

```python
# 问题：Agent之间无法有效协作
# 解决方案：明确的协作协议

class AgentCollaborationProtocol:
    def __init__(self):
        self.shared_context = {}
        self.collaboration_rules = {
            "research_before_create": True,
            "validate_before_execute": True,
            "share_findings": True
        }
    
    def update_shared_context(self, agent_name, findings):
        """更新共享上下文"""
        self.shared_context[agent_name] = {
            "timestamp": time.time(),
            "findings": findings
        }
    
    def get_collaboration_prompt(self, current_agent, task):
        """生成协作提示"""
        context_summary = self.summarize_shared_context()
        return f"""
        当前任务：{task}
        你的角色：{current_agent}
        团队发现：{context_summary}
        
        请基于团队已有的发现来完成你的部分工作，并确保与其他专家的结论保持一致。
        """
```

### 监控和日志

```python
# crewAI专用监控
class CrewAIMonitor:
    def __init__(self):
        self.metrics = {
            "total_requests": 0,
            "successful_requests": 0,
            "failed_requests": 0,
            "avg_response_time": 0,
            "active_crews": 0
        }
    
    def log_request_start(self, crew_name, task):
        self.metrics["total_requests"] += 1
        self.metrics["active_crews"] += 1
        
        logger.info(f"crewAI请求开始 - Crew: {crew_name}, Task: {task[:100]}...")
    
    def log_request_end(self, crew_name, success, duration):
        self.metrics["active_crews"] -= 1
        
        if success:
            self.metrics["successful_requests"] += 1
            logger.info(f"crewAI请求成功 - Crew: {crew_name}, Duration: {duration:.2f}s")
        else:
            self.metrics["failed_requests"] += 1
            logger.error(f"crewAI请求失败 - Crew: {crew_name}, Duration: {duration:.2f}s")
        
        # 更新平均响应时间
        self.update_avg_response_time(duration)
```

---

## 📚 总结

通过本指南，您可以在xiaozhi-server中成功集成crewAI多智能体系统。关键要点：

1. **渐进式集成**: 从基础功能开始，逐步添加高级特性
2. **智能路由**: 根据任务复杂度选择单agent或多agent模式
3. **性能优化**: 通过异步执行、缓存和资源池化提升性能
4. **错误恢复**: 实现完善的错误处理和回退机制
5. **监控诊断**: 建立完整的监控和日志系统

这种集成方案既保持了xiaozhi-server原有的实时性能，又为复杂任务提供了强大的多智能体协作能力。
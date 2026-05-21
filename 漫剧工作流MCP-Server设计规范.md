# 漫剧工作流系统 MCP Server 设计规范

**文档版本**：v2.1（基于实际API对齐）  
**编写日期**：2025年5月  
**适用系统**：漫剧工作流系统 v1.2+  
**文档状态**：正式版

---

## 目录

1. [概述](#1-概述)
2. [MCP架构设计总览](#2-mcp架构设计总览)
3. [Server 1：剧本清洗服务 (Script Cleaning Server)](#3-server-1剧本清洗服务script-cleaning-server)
4. [Server 2：剧本分析服务 (Script Analysis Server)](#4-server-2剧本分析服务script-analysis-server)
5. [Server 3：图片生成服务 (Image Generation Server)](#5-server-3图片生成服务image-generation-server)
6. [Server 4：视频生成服务 (Video Generation Server)](#6-server-4视频生成服务video-generation-server)
7. [通用规范](#7-通用规范)
8. [部署建议](#8-部署建议)
9. [前端集成方案](#9-前端集成方案)
10. [附录](#10-附录)

---

## 1. 概述

### 1.1 文档目的

本规范文档定义了漫剧工作流系统中MCP（Model Context Protocol）Server的设计标准，旨在为技术开发团队提供完整、可落地的接口规范和技术指导。文档涵盖四个核心Server的功能定义、接口设计、参数schema以及集成方案。

**本版本重点更新**：Video Generation Server已基于火山引擎Seedance、阿里云百炼万相、生数科技Vidu的实际API接口进行完全对齐设计。

### 1.2 适用范围

- 漫剧工作流系统后端开发
- MCP Server实现与部署
- 前端与MCP Server的集成对接
- 第三方系统对接评估

### 1.3 术语定义

| 术语 | 定义 |
|------|------|
| MCP | Model Context Protocol，模型上下文协议，用于AI服务与外部系统的标准通信协议 |
| 剧本清洗 | 将原始剧本文本转换为结构化标准格式的过程 |
| 分镜 | 将剧本内容拆分为可执行的视觉单元 |
| 融图 | 将角色、场景、道具等元素合成为完整画面的技术 |
| 图生视频 | 基于单张或多张图片生成动态视频的技术 |
| 多参图片 | 支持1-9张图片输入，每张图片可指定角色（首帧/尾帧/参考图），用于灵活的视频生成控制 |
| 参考生视频 | 基于参考视频/图片+描述信息生成新视频的技术 |
| 口型驱动 | 基于音频驱动图片角色口型动作的技术 |
| 首帧/尾帧 | 视频的首帧图片和尾帧图片，用于控制视频起止 |
| 多镜头叙事 | 在单个视频片段中切换多个镜头/场景 |

### 1.4 支持的实际API平台

| 平台 | 模型 | API类型 |
|------|------|---------|
| 火山引擎Seedance | doubao-seedance-2-0-260128, doubao-seedance-1-5-pro-251215, doubao-seedance-1-0-pro, doubao-seedance-1-0-pro-fast | REST API |
| 阿里云百炼万相 | wan2.7-i2v, wan2.6-i2v, wan2.6-t2v, wan2.6-r2v, wan2.2-s2v | REST API |
| 生数科技Vidu | 解说剧Agent | Agent API |

### 1.5 版本历史

| 版本 | 日期 | 修改内容 | 作者 |
|------|------|----------|------|
| v1.0 | 2025-01 | 初始版本 | - |
| v2.0 | 2025-05 | 基于实际API对齐Video Generation Server，新增口型驱动、多镜头叙事能力 | - |
| v2.1 | 2025-05 | 合并图生视频Tool：generate_video_from_image + generate_video_first_last_frame → generate_video_from_images（多参图片生视频） | - |

---

## 2. MCP架构设计总览

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              漫剧工作流系统                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│   │  剧本列表     │────▶│  剧本解析     │────▶│  解析详情    │                │
│   │  (Step 1)    │     │  (Step 2)     │     │  (Step 3)    │                │
│   └──────────────┘     └──────────────┘     └──────────────┘                │
│                                                        │                     │
│                                                        ▼                     │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│   │  生成视频     │◀────│  融图管理     │◀────│  分镜列表    │                │
│   │  (Step 6)    │     │  (Step 5)     │     │  (Step 4)    │                │
│   └──────────────┘     └──────────────┘     └──────────────┘                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            MCP Server Layer                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Script Cleaning Server（剧本清洗服务）                   │    │
│  │  Tools: clean_script | validate_script | detect_script_type         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Script Analysis Server（剧本分析服务）                   │    │
│  │  Tools: analyze_characters | analyze_scenes | analyze_props          │    │
│  │         generate_storyboard_prompts | calculate_duration            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Image Generation Server（图片生成服务）                 │    │
│  │  Tools: generate_character_image | generate_scene_image             │    │
│  │         generate_composite_image | generate_reference_image         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Video Generation Server（视频生成服务）                 │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  Tools:                                                              │    │
│  │    - generate_video_from_images (多参图片生视频)                        │    │
│  │    - generate_video_with_reference (参考生视频)                       │    │
│  │    - generate_video_text2video (文生视频)                            │    │
│  │    - generate_lip_sync_video (口型驱动)                              │    │
│  │    - query_video_task | download_video | retry_generation           │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  Providers: 火山引擎Seedance | 阿里云百炼万相 | 生数科技Vidu         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Server调用关系说明

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Server调用依赖关系                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Script Cleaning Server                                                     │
│         │                                                                    │
│         │ 产出: 结构化YAML剧本                                                │
│         ▼                                                                    │
│   Script Analysis Server                                                     │
│         │                                                                    │
│         │ 产出: 角色列表、场景列表、道具列表、分镜prompt                       │
│         ▼                                                                    │
│   Image Generation Server                                                    │
│         │                                                                    │
│         │ 产出: 角色图、场景图、融图                                           │
│         ▼                                                                    │
│   Video Generation Server                                                    │
│         │                                                                    │
│         │ 产出: 最终视频                                                      │
│         ▼                                                                    │
│   [Vidu Agent] ──▶ 全自动成片（可选流程）                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 数据流转说明

| 流程阶段 | 输入 | 处理 | 输出 | 对应Server |
|---------|------|------|------|------------|
| 剧本清洗 | 原始剧本文本 | 规则解析+AI清洗 | 结构化YAML | Script Cleaning |
| 剧本分析 | 清洗后剧本 | 实体提取+分镜生成 | 角色/场景/道具/分镜prompt | Script Analysis |
| 图片生成 | 分镜prompt+角色/场景描述 | 图像生成模型 | 角色图/场景图/融图 | Image Generation |
| 视频生成 | 融图+描述/参考视频 | 视频生成模型 | 最终视频 | Video Generation |

### 2.4 异步任务处理流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           异步任务处理流程                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   客户端发起请求                                                             │
│         │                                                                    │
│         ▼                                                                    │
│   Server创建任务 ──▶ 返回task_id                                              │
│         │                                                                    │
│         ▼                                                                    │
│   轮询查询状态 ──▶ query_video_task(task_id)                                  │
│         │                                                                    │
│         ├────────────────────────────┐                                       │
│         ▼                            ▼                                       │
│    succeeded ◀───────────────── failed（可重试）                               │
│         │                            │                                       │
│         ▼                            ▼                                       │
│  download_video(t_id) ◀── retry_generation(t_id)                            │
│                                                                              │
│  ⚠️ 注意：视频URL有效期24小时，需及时转存                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Server 1：剧本清洗服务 (Script Cleaning Server)

### 3.1 Server概述

**Server名称**: `script-cleaning-server`  
**核心职责**: 接收原始剧本文本，根据剧本类型（解说漫/精品漫）应用不同的清洗规则，输出结构化的标准格式剧本数据。  
**职责边界**: 本服务仅负责剧本的清洗和格式化，不负责角色/场景/道具的提取分析。  
**适用场景**: 剧本列表→剧本解析的工作流阶段。

### 3.2 工具定义 (Tools)

#### 3.2.1 clean_script（剧本清洗）

**功能描述**: 接收原始剧本文本，根据指定类型进行清洗，输出结构化标准格式。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "clean_script_input",
  "required": ["raw_script", "script_type"],
  "properties": {
    "raw_script": {
      "type": "string",
      "description": "原始剧本文本内容",
      "minLength": 1,
      "maxLength": 50000
    },
    "script_type": {
      "type": "string",
      "enum": ["narrative", "premium"],
      "description": "剧本类型：narrative-解说漫，premium-精品漫"
    },
    "options": {
      "type": "object",
      "description": "可选参数",
      "properties": {
        "preserve_original": {
          "type": "boolean",
          "default": true,
          "description": "是否保留原始剧本片段"
        },
        "language": {
          "type": "string",
          "enum": ["zh-CN", "en-US"],
          "default": "zh-CN",
          "description": "输出语言"
        },
        "custom_rules": {
          "type": "array",
          "items": {"type": "string"},
          "description": "自定义清洗规则ID列表"
        }
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "clean_script_output",
  "required": ["success", "data", "metadata"],
  "properties": {
    "success": {"type": "boolean"},
    "data": {
      "type": "object",
      "description": "清洗后的结构化数据",
      "required": ["script_id", "script_type", "scenes"],
      "properties": {
        "script_id": {"type": "string", "description": "剧本唯一标识符(UUID)"},
        "script_type": {"type": "string", "enum": ["narrative", "premium"]},
        "title": {"type": "string", "description": "剧本标题"},
        "total_duration": {"type": "number", "description": "预估总时长(秒)"},
        "scenes": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "scene_id": {"type": "string"},
              "scene_name": {"type": "string"},
              "scene_description": {"type": "string"},
              "time_setting": {"type": "string", "enum": ["day", "night", "morning", "evening", "dawn", "dusk"]},
              "location": {"type": "string"},
              "paragraphs": {
                "type": "array",
                "items": {
                  "type": "object",
                  "properties": {
                    "paragraph_id": {"type": "string"},
                    "order": {"type": "integer"},
                    "characters": {"type": "array", "items": {"type": "string"}},
                    "action": {"type": "string"},
                    "emotion": {"type": "string"},
                    "voice_type": {"type": "string", "enum": ["narration", "dialogue", "sound_effect", "silence"]},
                    "mouth_state": {"type": "string"},
                    "dialogue": {"type": "string"},
                    "camera_suggestion": {"type": "string"},
                    "shot_type": {"type": "string"},
                    "original_text": {"type": "string"}
                  }
                }
              }
            }
          }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "cleaned_at": {"type": "string", "format": "date-time"},
        "rules_version": {"type": "string"},
        "processing_time_ms": {"type": "integer"},
        "character_count": {"type": "integer"},
        "scene_count": {"type": "integer"}
      }
    },
    "errors": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "code": {"type": "string"},
          "message": {"type": "string"},
          "position": {"type": "string"}
        }
      }
    }
  }
}
```

**错误码**:

| 错误码 | 描述 | 处理建议 |
|--------|------|----------|
| SC001 | 剧本内容为空 | 检查输入内容 |
| SC002 | 剧本内容过长 | 分段处理或缩短 |
| SC003 | 剧本类型不支持 | 使用narrative或premium |
| SC004 | 清洗规则加载失败 | 检查服务端配置 |
| SC005 | AI服务调用超时 | 重试请求 |
| SC006 | 清洗结果格式异常 | 检查清洗规则版本 |

**示例请求**:

```json
{
  "raw_script": "【场景：城市街道·日】\n\n小明走在街上，心情低落。他看着手机上的消息，叹了口气。\n\n旁白：又失败了，这已经是第三次了。\n\n小明：（摇头）算了，还是回家吧。\n\n【场景：家中·夜】\n\n小明躺在床上，回想着今天发生的一切。",
  "script_type": "narrative",
  "options": {
    "preserve_original": true,
    "language": "zh-CN"
  }
}
```

**示例响应**:

```json
{
  "success": true,
  "data": {
    "script_id": "550e8400-e29b-41d4-a716-446655440000",
    "script_type": "narrative",
    "title": "未命名剧本",
    "total_duration": 45.0,
    "scenes": [
      {
        "scene_id": "scene_001",
        "scene_name": "城市街道",
        "scene_description": "繁华的城市街道，阳光明媚",
        "time_setting": "day",
        "location": "outdoor",
        "paragraphs": [
          {
            "paragraph_id": "para_001",
            "order": 1,
            "characters": ["小明"],
            "action": "人物行走，低头看手机，表情失落，叹气",
            "emotion": "失落",
            "voice_type": "narration",
            "mouth_state": "closed",
            "original_text": "小明走在街上，心情低落。他看着手机上的消息，叹了口气。"
          }
        ]
      }
    ]
  },
  "metadata": {
    "cleaned_at": "2025-05-11T10:30:00+08:00",
    "rules_version": "narrative_v2.1",
    "processing_time_ms": 2350,
    "character_count": 156,
    "scene_count": 2
  },
  "errors": []
}
```

---

#### 3.2.2 validate_script（剧本校验）

**功能描述**: 校验清洗后的剧本是否符合标准格式要求。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "validate_script_input",
  "required": ["script_data"],
  "properties": {
    "script_data": {"type": "object", "description": "清洗后的剧本数据"},
    "validation_level": {
      "type": "string",
      "enum": ["strict", "normal", "relaxed"],
      "default": "normal"
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "validate_script_output",
  "properties": {
    "success": {"type": "boolean"},
    "is_valid": {"type": "boolean"},
    "errors": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "code": {"type": "string"},
          "message": {"type": "string"},
          "field": {"type": "string"},
          "severity": {"type": "string", "enum": ["error", "warning", "info"]}
        }
      }
    },
    "summary": {
      "type": "object",
      "properties": {
        "total_errors": {"type": "integer"},
        "total_warnings": {"type": "integer"},
        "passed_checks": {"type": "integer"},
        "failed_checks": {"type": "integer"}
      }
    }
  }
}
```

---

#### 3.2.3 detect_script_type（剧本类型检测）

**功能描述**: 自动检测原始剧本的类型（解说漫/精品漫）。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "detect_script_type_input",
  "required": ["raw_script"],
  "properties": {
    "raw_script": {
      "type": "string",
      "minLength": 1,
      "maxLength": 50000
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "detect_script_type_output",
  "properties": {
    "success": {"type": "boolean"},
    "detected_type": {"type": "string", "enum": ["narrative", "premium", "unknown"]},
    "confidence": {"type": "number", "minimum": 0, "maximum": 1},
    "reasons": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "indicator": {"type": "string"},
          "weight": {"type": "number"}
        }
      }
    },
    "suggestion": {"type": "string"}
  }
}
```

---

### 3.3 资源定义 (Resources)

#### 3.3.1 cleaning_rules_narrative

**URI**: `script-cleaning://rules/narrative`  
**描述**: 解说漫剧本清洗规则  
**数据格式**: JSON

```json
{
  "rule_version": "narrative_v2.1",
  "rule_name": "解说漫清洗规则",
  "rules": {
    "scene_detection": {"enabled": true, "patterns": ["【场景：{name}】", "【{time}·{location}】"]},
    "character_extraction": {"enabled": true},
    "action_visualization": {"enabled": true},
    "emotion_tagging": {"enabled": true, "emotion_categories": ["开心", "悲伤", "愤怒", "恐惧", "惊讶", "平静", "失落", "兴奋", "紧张", "释然"]},
    "narration_handling": {"enabled": true, "narration_markers": ["旁白：", "（旁白）", "【旁白】"]},
    "psychological_removal": {"enabled": true}
  }
}
```

#### 3.3.2 cleaning_rules_premium

**URI**: `script-cleaning://rules/premium`  
**描述**: 精品漫剧本清洗规则  
**数据格式**: JSON

```json
{
  "rule_version": "premium_v2.1",
  "rule_name": "精品漫清洗规则",
  "rules": {
    "scene_detection": {"enabled": true},
    "character_state": {"enabled": true},
    "action_breakdown": {"enabled": true, "granularity": "shot_level"},
    "dialogue_extraction": {"enabled": true},
    "shot_suggestion": {"enabled": true, "shot_types": ["远景(EWS)", "全景(WS)", "中景(MS)", "近景(CU)", "特写(ECU)"]},
    "camera_movement": {"enabled": true, "movements": ["固定", "推", "拉", "摇", "移", "跟", "手持"]},
    "timeline_split": {"enabled": true, "unit": "second"},
    "mouth_state_annotation": {"enabled": true}
  }
}
```

---

### 3.4 提示模板定义 (Prompts)

#### 3.4.1 narrative_cleaning_prompt

**名称**: `narrative_cleaning_prompt`  
**描述**: 解说漫剧本清洗专用提示词模板

```markdown
# 解说漫剧本清洗任务

## 任务
请将下面的原始剧本按照解说漫清洗规则进行清洗。

## 输入
原始剧本：
{{raw_script}}

## 解说漫清洗规则

### 1. 场景检测与标注
- 识别剧本中的场景转换标记
- 提取场景名称、时间设定（day/night/morning/evening）、地点（indoor/outdoor）

### 2. 角色提取
- 从对白、动作描写、括号指示中提取角色名称
- 归一化角色名称

### 3. 动作具象化
将抽象的心理描写转化为具体的、可视化的动作描述

### 4. 情绪标注
为每个段落标注主要情绪

### 5. 旁白处理
识别旁白标记，将旁白单独作为段落处理

### 6. 去心理描写
移除或转化心理描写为可观察的动作或表情

### 7. 嘴巴状态规则
- dialogue → speaking
- narration → no_mouth
- silence → closed

## 输出要求
1. 输出格式：结构化JSON或YAML
2. 保持原始剧本片段（original_text字段）
3. 每个段落必须有action字段，不能为空
```

---

## 4. Server 2：剧本分析服务 (Script Analysis Server)

### 4.1 Server概述

**Server名称**: `script-analysis-server`  
**核心职责**: 对清洗后的结构化剧本进行深度分析，提取角色、场景、道具等实体信息，并生成分镜prompt。  
**适用场景**: 剧本解析→解析详情的工作流阶段。

### 4.2 工具定义 (Tools)

#### 4.2.1 analyze_characters（角色分析）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "analyze_characters_input",
  "required": ["script_data"],
  "properties": {
    "script_data": {"type": "object"},
    "options": {
      "type": "object",
      "properties": {
        "generate_appearance": {"type": "boolean", "default": true},
        "style_reference": {"type": "string", "default": "二次元插画风格"}
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "analyze_characters_output",
  "properties": {
    "success": {"type": "boolean"},
    "characters": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "character_id": {"type": "string"},
          "name": {"type": "string"},
          "role_type": {"type": "string", "enum": ["protagonist", "supporting", "minor", "background"]},
          "description": {"type": "string"},
          "appearance": {
            "type": "object",
            "properties": {
              "age_estimate": {"type": "string"},
              "gender": {"type": "string"},
              "hair": {"type": "string"},
              "eyes": {"type": "string"},
              "clothing": {"type": "string"}
            }
          },
          "appearance_description": {"type": "string"},
          "personality": {"type": "string"},
          "emotions": {"type": "array", "items": {"type": "string"}}
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "total_characters": {"type": "integer"},
        "analyzed_at": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

**示例请求**:

```json
{
  "script_data": {
    "script_id": "550e8400-e29b-41d4-a716-446655440000",
    "scenes": [
      {
        "scene_id": "scene_001",
        "paragraphs": [
          {
            "paragraph_id": "para_001",
            "characters": ["小明"],
            "action": "人物行走，低头看手机，表情失落，叹气",
            "emotion": "失落"
          }
        ]
      }
    ]
  },
  "options": {
    "generate_appearance": true,
    "style_reference": "二次元插画风格"
  }
}
```

**示例响应**:

```json
{
  "success": true,
  "characters": [
    {
      "character_id": "char_001",
      "name": "小明",
      "role_type": "protagonist",
      "description": "一个普通的大学生，正在经历人生低谷",
      "appearance": {
        "age_estimate": "20-25岁",
        "gender": "男",
        "hair": "黑色短发，刘海微长",
        "eyes": "棕色眼睛，眼神略显疲惫",
        "clothing": "休闲装，深色卫衣"
      },
      "appearance_description": "20岁左右的男生，黑色短发，刘海微长，棕色眼睛，穿深色卫衣，表情略显疲惫但不失帅气，二次元插画风格",
      "personality": "内向但善良，有点固执",
      "emotions": ["失落", "释然", "沉思"]
    }
  ],
  "metadata": {
    "total_characters": 1,
    "analyzed_at": "2025-05-11T10:35:00+08:00"
  }
}
```

---

#### 4.2.2 analyze_scenes（场景分析）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "analyze_scenes_input",
  "required": ["script_data"],
  "properties": {
    "script_data": {"type": "object"},
    "options": {
      "type": "object",
      "properties": {
        "generate_lighting": {"type": "boolean", "default": true},
        "generate_atmosphere": {"type": "boolean", "default": true}
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "analyze_scenes_output",
  "properties": {
    "success": {"type": "boolean"},
    "scenes": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "scene_id": {"type": "string"},
          "name": {"type": "string"},
          "description": {"type": "string"},
          "type": {"type": "string", "enum": ["interior", "exterior"]},
          "time_setting": {"type": "string"},
          "lighting": {
            "type": "object",
            "properties": {
              "main_light": {"type": "string"},
              "quality": {"type": "string"},
              "description": {"type": "string"}
            }
          },
          "atmosphere": {"type": "string"},
          "color_palette": {"type": "array", "items": {"type": "string"}},
          "scene_description_for_image": {"type": "string"},
          "elements": {"type": "array", "items": {"type": "string"}}
        }
      }
    }
  }
}
```

---

#### 4.2.3 analyze_props（道具分析）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "analyze_props_input",
  "required": ["script_data"],
  "properties": {
    "script_data": {"type": "object"}
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "analyze_props_output",
  "properties": {
    "success": {"type": "boolean"},
    "props": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "prop_id": {"type": "string"},
          "name": {"type": "string"},
          "description": {"type": "string"},
          "category": {"type": "string"},
          "usage": {"type": "string"},
          "visual_importance": {"type": "string"}
        }
      }
    }
  }
}
```

---

#### 4.2.4 generate_storyboard_prompts（分镜Prompt生成）

**功能描述**: 根据清洗后的剧本、角色、场景、道具信息，生成完整的分镜prompt列表。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_storyboard_prompts_input",
  "required": ["script_data", "characters", "scenes"],
  "properties": {
    "script_data": {"type": "object"},
    "characters": {"type": "array"},
    "scenes": {"type": "array"},
    "props": {"type": "array"},
    "options": {
      "type": "object",
      "properties": {
        "default_style": {"type": "string", "default": "二次元插画风格"},
        "aspect_ratio": {"type": "string", "enum": ["9:16", "16:9", "1:1"], "default": "9:16"},
        "include_mouth_control": {"type": "boolean", "default": true},
        "mouth_control_prefix": {"type": "string", "default": "全程人物无对白口型，嘴巴不动"},
        "prompt_language": {"type": "string", "enum": ["zh-CN", "en-US"], "default": "zh-CN"}
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_storyboard_prompts_output",
  "properties": {
    "success": {"type": "boolean"},
    "storyboards": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "storyboard_id": {"type": "string"},
          "scene_id": {"type": "string"},
          "paragraph_id": {"type": "string"},
          "order": {"type": "integer"},
          "time_segment": {"type": "string"},
          "scene_prompt": {"type": "string"},
          "character_prompt": {"type": "string"},
          "action_prompt": {"type": "string"},
          "camera_prompt": {"type": "string"},
          "combined_prompt": {"type": "string"},
          "video_prompt": {"type": "string"},
          "mouth_state": {"type": "string"},
          "voice_type": {"type": "string"},
          "emotion": {"type": "string"},
          "characters_in_frame": {"type": "array", "items": {"type": "string"}},
          "props_in_frame": {"type": "array", "items": {"type": "string"}},
          "style": {"type": "string"},
          "aspect_ratio": {"type": "string"},
          "estimated_duration": {"type": "number"},
          "reference_image_needed": {"type": "boolean"},
          "shot_type": {"type": "string", "description": "景别：multi-多镜头叙事，single-单镜头"}
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "total_storyboards": {"type": "integer"},
        "total_duration": {"type": "number"},
        "style": {"type": "string"},
        "generated_at": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

**示例请求**:

```json
{
  "script_data": {
    "scenes": [
      {
        "scene_id": "scene_001",
        "paragraphs": [
          {
            "paragraph_id": "para_001",
            "order": 1,
            "characters": ["小明"],
            "action": "人物行走，低头看手机，表情失落，叹气",
            "emotion": "失落",
            "voice_type": "narration"
          }
        ]
      }
    ]
  },
  "characters": [
    {
      "character_id": "char_001",
      "name": "小明",
      "appearance_description": "20岁左右的男生，黑色短发，刘海微长，棕色眼睛，穿深色卫衣，表情略显疲惫但不失帅气，二次元插画风格"
    }
  ],
  "scenes": [
    {
      "scene_id": "scene_001",
      "name": "城市街道",
      "scene_description_for_image": "繁华的城市街道，阳光明媚，温暖的日光透过树叶洒下斑驳光影，浅灰色路面，蓝灰色天空，街道两旁有绿色植物点缀，画面温暖明亮，二次元插画风格"
    }
  ],
  "options": {
    "default_style": "二次元插画风格",
    "aspect_ratio": "9:16",
    "include_mouth_control": true
  }
}
```

**示例响应**:

```json
{
  "success": true,
  "storyboards": [
    {
      "storyboard_id": "sb_001",
      "scene_id": "scene_001",
      "paragraph_id": "para_001",
      "order": 1,
      "time_segment": "00:00-00:05",
      "scene_prompt": "繁华的城市街道，阳光明媚，温暖的日光透过树叶洒下斑驳光影，浅灰色路面，蓝灰色天空，街道两旁有绿色植物点缀，画面温暖明亮，二次元插画风格",
      "character_prompt": "20岁左右的男生，黑色短发，刘海微长，棕色眼睛，穿深色卫衣，表情略显疲惫但不失帅气，二次元插画风格",
      "action_prompt": "人物侧身行走，低头看着手机，步伐缓慢，肩膀微微下垂，时不时叹气，身体随叹气轻微起伏",
      "camera_prompt": "中景跟随镜头，平视角度，轻微跟拍",
      "combined_prompt": "繁华的城市街道，阳光明媚，温暖的日光透过树叶洒下斑驳光影，浅灰色路面，蓝灰色天空，街道两旁有绿色植物点缀，画面温暖明亮，二次元插画风格。人物：20岁左右的男生，黑色短发，刘海微长，棕色眼睛，穿深色卫衣，表情略显疲惫但不失帅气，二次元插画风格。动作：人物侧身行走，低头看着手机，步伐缓慢，肩膀微微下垂，时不时叹气，身体随叹气轻微起伏。视角：中景，平视",
      "video_prompt": "人物侧身行走，低头看着手机，步伐缓慢，肩膀微微下垂，时不时叹气，身体随叹气轻微起伏。全程人物无对白口型，嘴巴不动。二次元插画风格，动作流畅自然",
      "mouth_state": "no_mouth",
      "voice_type": "narration",
      "emotion": "失落",
      "characters_in_frame": ["小明"],
      "props_in_frame": ["手机"],
      "style": "二次元插画风格",
      "aspect_ratio": "9:16",
      "estimated_duration": 5.0,
      "reference_image_needed": false,
      "shot_type": "single"
    }
  ],
  "metadata": {
    "total_storyboards": 1,
    "total_duration": 5.0,
    "style": "二次元插画风格",
    "generated_at": "2025-05-11T10:40:00+08:00"
  }
}
```

---

#### 4.2.5 calculate_duration（时长计算）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "calculate_duration_input",
  "required": ["storyboard_data"],
  "properties": {
    "storyboard_data": {
      "type": "object",
      "properties": {
        "action": {"type": "string"},
        "dialogue": {"type": "string"},
        "voice_type": {"type": "string"},
        "camera_movement": {"type": "string"}
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "calculate_duration_output",
  "properties": {
    "success": {"type": "boolean"},
    "estimated_duration": {"type": "number"},
    "breakdown": {
      "type": "object",
      "properties": {
        "base_duration": {"type": "number"},
        "dialogue_duration": {"type": "number"},
        "action_adjustment": {"type": "number"}
      }
    },
    "recommended_duration": {"type": "string", "description": "如4s/5s/6s/8s/10s"}
  }
}
```

---

## 5. Server 3：图片生成服务 (Image Generation Server)

### 5.1 Server概述

**Server名称**: `image-generation-server`  
**核心职责**: 生成角色图片、场景图片、融图（角色+场景+分镜描述合成）以及参考图片。  
**适用场景**: 分镜列表→融图管理的工作流阶段。

### 5.2 支持的模型

| 模型ID | 适用场景 | 特点 |
|--------|----------|------|
| `doubao-widebeam-0.5` | 角色图、融图 | 擅长宽画面构图 |
| `doubao-sdstream-4-5` | 场景图、细节图 | 高细节表现 |
| `happyhorse-1-0` | 风格化图片 | 插画风格优化 |

### 5.3 工具定义 (Tools)

#### 5.3.1 generate_character_image（生成角色图片）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_character_image_input",
  "required": ["character_data"],
  "properties": {
    "character_data": {
      "type": "object",
      "properties": {
        "character_id": {"type": "string"},
        "name": {"type": "string"},
        "appearance_description": {"type": "string"},
        "emotion": {"type": "string"},
        "action": {"type": "string"}
      }
    },
    "model": {
      "type": "string",
      "enum": ["doubao-widebeam-0.5", "doubao-sdstream-4-5", "happyhorse-1-0"],
      "default": "doubao-widebeam-0.5"
    },
    "style": {"type": "string", "default": "二次元插画风格"},
    "aspect_ratio": {
      "type": "string",
      "enum": ["9:16", "16:9", "1:1"],
      "default": "9:16"
    },
    "advanced_params": {
      "type": "object",
      "properties": {
        "negative_prompt": {"type": "string"},
        "strength_ratio": {"type": "number", "minimum": 0, "maximum": 1, "default": 0.7},
        "blur_ratio": {"type": "number", "minimum": 0, "maximum": 1, "default": 0}
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_character_image_output",
  "properties": {
    "success": {"type": "boolean"},
    "task_id": {"type": "string"},
    "status": {"type": "string", "enum": ["pending", "processing", "completed", "failed"]},
    "metadata": {
      "type": "object",
      "properties": {
        "model": {"type": "string"},
        "created_at": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

**任务完成后响应**:

```json
{
  "success": true,
  "task_id": "img_task_001",
  "status": "completed",
  "result": {
    "character_id": "char_001",
    "image_url": "https://cdn.example.com/characters/char_001_001.png",
    "width": 1024,
    "height": 1792,
    "file_size": 2048000,
    "format": "PNG"
  },
  "metadata": {
    "model": "doubao-widebeam-0.5",
    "created_at": "2025-05-11T10:45:00+08:00"
  }
}
```

---

#### 5.3.2 generate_scene_image（生成场景图片）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_scene_image_input",
  "required": ["scene_data"],
  "properties": {
    "scene_data": {
      "type": "object",
      "properties": {
        "scene_id": {"type": "string"},
        "name": {"type": "string"},
        "scene_description_for_image": {"type": "string"},
        "lighting_description": {"type": "string"},
        "atmosphere": {"type": "string"}
      }
    },
    "model": {
      "type": "string",
      "enum": ["doubao-widebeam-0.5", "doubao-sdstream-4-5", "happyhorse-1-0"],
      "default": "doubao-sdstream-4-5"
    },
    "style": {"type": "string", "default": "二次元插画风格"},
    "aspect_ratio": {
      "type": "string",
      "enum": ["9:16", "16:9", "1:1"],
      "default": "16:9"
    },
    "advanced_params": {
      "type": "object",
      "properties": {
        "negative_prompt": {"type": "string"},
        "blur_ratio": {"type": "number", "minimum": 0, "maximum": 1, "default": 0.2}
      }
    }
  }
}
```

---

#### 5.3.3 generate_composite_image（生成融图）

**功能描述**: 将角色、场景、道具等元素合成为完整的分镜画面。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_composite_image_input",
  "required": ["storyboard_data"],
  "properties": {
    "storyboard_data": {
      "type": "object",
      "properties": {
        "storyboard_id": {"type": "string"},
        "combined_prompt": {"type": "string"},
        "characters": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "character_id": {"type": "string"},
              "character_image_url": {"type": "string"},
              "position": {
                "type": "object",
                "properties": {
                  "x": {"type": "number"},
                  "y": {"type": "number"},
                  "scale": {"type": "number"}
                }
              }
            }
          }
        },
        "scene_image_url": {"type": "string"},
        "scene_prompt": {"type": "string"}
      }
    },
    "model": {"type": "string", "default": "doubao-widebeam-0.5"},
    "style": {"type": "string", "default": "二次元插画风格"},
    "aspect_ratio": {"type": "string", "enum": ["9:16", "16:9", "1:1"], "default": "9:16"},
    "composition_mode": {
      "type": "string",
      "enum": ["character_first", "scene_first", "balance"],
      "default": "balance"
    },
    "advanced_params": {
      "type": "object",
      "properties": {
        "character_strength": {"type": "number", "minimum": 0, "maximum": 1, "default": 0.6},
        "scene_strength": {"type": "number", "minimum": 0, "maximum": 1, "default": 0.8},
        "edge_feathering": {"type": "number", "minimum": 0, "maximum": 1, "default": 0.3}
      }
    }
  }
}
```

**任务完成后响应**:

```json
{
  "success": true,
  "task_id": "composite_task_001",
  "status": "completed",
  "result": {
    "storyboard_id": "sb_001",
    "composite_image_url": "https://cdn.example.com/composite/sb_001_001.png",
    "width": 1024,
    "height": 1792,
    "file_size": 3072000,
    "format": "PNG"
  }
}
```

---

#### 5.3.4 generate_reference_image（生成参考图）

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_reference_image_input",
  "required": ["description"],
  "properties": {
    "description": {"type": "string"},
    "model": {"type": "string", "default": "doubao-sdstream-4-5"},
    "style": {"type": "string", "default": "二次元插画风格"},
    "aspect_ratio": {"type": "string", "enum": ["9:16", "16:9", "1:1"], "default": "9:16"},
    "reference_type": {
      "type": "string",
      "enum": ["character", "scene", "action", "pose", "expression", "general"],
      "default": "general"
    }
  }
}
```

---

## 6. Server 4：视频生成服务 (Video Generation Server)

### 6.1 Server概述

**Server名称**: `video-generation-server`  
**核心职责**: 基于图片或参考视频生成动态视频内容，支持多种生成模式。  
**适用场景**: 融图管理→生成视频的工作流阶段。  
**重要更新**: 本版本已完全对齐火山引擎Seedance、阿里云百炼万相、生数科技Vidu的实际API接口。

### 6.2 支持的实际API模型

#### 6.2.1 火山引擎 Seedance

| 模型ID | 支持模式 | 时长范围 | 分辨率 | 有声 |
|--------|----------|----------|--------|------|
| `doubao-seedance-2-0-260128` | 多模态参考生视频、图生视频(首帧/首尾帧)、文生视频 | 4-15s 或 -1(智能) | 480p/720p/1080p | ✅ |
| `doubao-seedance-1-5-pro-251215` | 图生视频(首帧/首尾帧)、文生视频、draft模式 | 4-12s 或 -1 | 480p/720p/1080p | ✅ |
| `doubao-seedance-1-0-pro` | 图生视频(首帧/首尾帧)、文生视频 | 2-12s | 480p/720p/1080p | ❌ |
| `doubao-seedance-1-0-pro-fast` | 图生视频(首帧)、文生视频 | 2-12s | 480p/720p/1080p | ❌ |

#### 6.2.2 阿里云百炼 万相

| 模型ID | 支持模式 | 时长范围 | 分辨率 | 特殊能力 |
|--------|----------|----------|--------|----------|
| `wan2.7-i2v` | 首帧生视频、首尾帧生视频、视频续写 | 2-15s | 720P/1080P | 多镜头叙事 |
| `wan2.6-i2v` | 图生视频(首帧)、文生视频、参考生视频(r2v) | 5/10/15s | 480P/720P/1080P | prompt智能改写 |
| `wan2.6-r2v` | 参考生视频 | 5/10s | 720P/1080P | 多角色(最多3个) |
| `wan2.2-s2v` | 数字人口型驱动 | - | 480P/720P | 音频驱动口型 |

#### 6.2.3 模型选择建议

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 普通图生视频 | Seedance 2.0 / wan2.7-i2v | 质量好，支持有声 |
| 需要精确控制起止 | Seedance 2.0 / wan2.7-i2v (首尾帧) | 控制首尾帧 |
| 参考视频保持风格 | Seedance 2.0 (多模态) / wan2.6-r2v | 继承参考视频风格 |
| 口型驱动 | wan2.2-s2v | 专业口型驱动 |
| 多镜头叙事 | wan2.7-i2v (shot_type=multi) | 单片段多镜头 |
| 快速预览 | Seedance 1.5 pro (draft模式) | 速度快，50%价格 |

### 6.3 工具定义 (Tools)

---

#### 6.4.1 generate_video_from_images（多参图片生视频）

**功能描述**: 基于多张图片和提示词生成视频。支持图生视频-首帧（1张）、图生视频-首尾帧（2张，指定first_frame和last_frame）、多模态参考生视频（1-9张参考图+可选参考视频/音频）三种模式。对齐 Seedance 的 image_url content 结构和万相的 media 结构。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_video_from_images_input",
  "required": ["images", "prompt"],
  "properties": {
    "images": {
      "type": "array",
      "description": "图片列表，支持1-9张。每张图片可指定角色：first_frame（首帧）、last_frame（尾帧）、reference_image（参考图）。仅传入1张first_frame时为图生视频-首帧模式；传入first_frame+last_frame时为首尾帧模式；传入reference_image时为多模态参考模式",
      "minItems": 1,
      "maxItems": 9,
      "items": {
        "type": "object",
        "required": ["url"],
        "properties": {
          "url": {
            "type": "string",
            "description": "图片URL，支持：http://, https://, Base64(data:image/xxx;base64,...), 素材ID(asset://ASSET_ID)"
          },
          "role": {
            "type": "string",
            "enum": ["first_frame", "last_frame", "reference_image"],
            "default": "first_frame",
            "description": "图片角色：首帧/尾帧/参考图。图生视频-首帧模式仅需1张first_frame；首尾帧模式需1张first_frame+1张last_frame；多模态参考模式可传1-9张reference_image"
          }
        }
      }
    },
    "prompt": {
      "type": "string",
      "description": "视频生成提示词，描述期望生成的视频内容"
    },
    "provider": {
      "type": "string",
      "enum": ["volcengine", "alibaba"],
      "default": "volcengine",
      "description": "视频服务提供商"
    },
    "model": {
      "type": "string",
      "description": "模型ID",
      "examples": ["doubao-seedance-2-0-260128", "doubao-seedance-1-5-pro-251215", "wan2.7-i2v", "wan2.6-i2v"]
    },
    "resolution": {
      "type": "string",
      "enum": ["480p", "720p", "1080p"],
      "default": "720p",
      "description": "输出分辨率"
    },
    "ratio": {
      "type": "string",
      "enum": ["16:9", "4:3", "1:1", "3:4", "9:16", "21:9", "adaptive"],
      "default": "9:16",
      "description": "画面比例"
    },
    "duration": {
      "type": "integer",
      "description": "视频时长（秒），范围因模型而异。Seedance 2.0: [4,15]或-1(智能); 1.5 pro: [4,12]或-1; 1.0: [2,12]; wan2.7: [2,15]; wan2.6: 5/10/15",
      "examples": [4, 5, 6, 8, 10]
    },
    "generate_audio": {
      "type": "boolean",
      "default": true,
      "description": "是否生成音频（仅Seedance 2.0/1.5 pro、wan2.6/2.7支持）"
    },
    "mouth_state": {
      "type": "string",
      "enum": ["no_mouth", "closed", "speaking", "laughing", "shouting"],
      "description": "嘴巴状态，用于口型控制提示词自动注入"
    },
    "seed": {
      "type": "integer",
      "description": "随机种子，-1表示随机"
    },
    "camera_fixed": {
      "type": "boolean",
      "default": false,
      "description": "是否固定镜头（参考图场景不支持，Seedance 2.0暂不支持）"
    },
    "watermark": {
      "type": "boolean",
      "default": false,
      "description": "是否添加水印"
    },
    "callback_url": {
      "type": "string",
      "format": "uri",
      "description": "生成完成后的回调通知地址"
    },
    "return_last_frame": {
      "type": "boolean",
      "default": false,
      "description": "是否返回尾帧图片URL（可用于连续视频生成：上一个视频的尾帧作为下一个的首帧）"
    },
    "service_tier": {
      "type": "string",
      "enum": ["default", "flex"],
      "default": "default",
      "description": "服务等级（flex为离线模式，50%价格，Seedance 2.0不支持）"
    },
    "reference_videos": {
      "type": "array",
      "description": "参考视频列表（仅Seedance 2.0支持，最多3个，总时长≤15s）。当images中包含reference_image时可搭配使用",
      "maxItems": 3,
      "items": {
        "type": "object",
        "required": ["url"],
        "properties": {
          "url": {
            "type": "string",
            "description": "参考视频URL或素材ID"
          }
        }
      }
    },
    "reference_audios": {
      "type": "array",
      "description": "参考音频列表（仅Seedance 2.0支持，最多3段，总时长≤15s）。不可单独使用，需配合图片或视频",
      "maxItems": 3,
      "items": {
        "type": "object",
        "required": ["url"],
        "properties": {
          "url": {
            "type": "string",
            "description": "参考音频URL，格式wav/mp3"
          }
        }
      }
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_video_from_images_output",
  "properties": {
    "success": {"type": "boolean"},
    "task_id": {"type": "string", "description": "任务ID（火山引擎格式：cgt-xxxxxxxxxx，阿里云格式：task_xxx）"},
    "status": {"type": "string", "enum": ["pending", "processing", "completed", "failed"]},
    "metadata": {
      "type": "object",
      "properties": {
        "provider": {"type": "string"},
        "model": {"type": "string"},
        "resolution": {"type": "string"},
        "ratio": {"type": "string"},
        "duration": {"type": "integer"},
        "created_at": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

**任务完成后响应**:

```json
{
  "success": true,
  "task_id": "cgt-20260414114820-*****",
  "status": "completed",
  "result": {
    "video_url": "https://ark.doubao.com/api/v3/files/xxxxx/video.mp4",
    "last_frame_url": null,
    "duration": 5,
    "width": 720,
    "height": 1280,
    "format": "MP4"
  },
  "metadata": {
    "provider": "volcengine",
    "model": "doubao-seedance-2-0-260128",
    "resolution": "720p",
    "ratio": "9:16",
    "duration": 5,
    "generate_audio": true,
    "created_at": "2025-05-11T10:55:00+08:00",
    "processing_time_ms": 45000
  }
}
```

**错误码**:

| 错误码 | 描述 | 处理建议 |
|--------|------|----------|
| VID001 | 模型不支持 | 检查model参数 |
| VID002 | 图片URL无效或无法访问 | 检查images参数，支持Base64 |
| VID003 | 比例不支持该模型 | 更换ratio参数 |
| VID004 | 时长超出范围 | 检查duration参数（Seedance 2.0: 4-15s） |
| VID005 | generate_audio不支持该模型 | 仅Seedance 2.0/1.5 pro支持 |
| VID006 | 服务不可用 | 稍后重试 |
| VID007 | 任务超时 | 使用retry_generation |
| VID008 | 生成失败 | 检查prompt或重试 |
| VID101 | 请求体超过64MB | 压缩图片 |
| VID102 | 单图片超过30MB | 压缩图片 |

**示例请求（模式1：图生视频-首帧，1张首帧图）**:

```json
{
  "images": [
    {
      "url": "https://cdn.example.com/bedroom_scene.png",
      "role": "first_frame"
    }
  ],
  "prompt": "阿橘躺在床上，闭着眼睛，表情平静，进入梦乡。二次元插画风格",
  "provider": "volcengine",
  "model": "doubao-seedance-2-0-260128",
  "resolution": "720p",
  "ratio": "9:16",
  "duration": 4,
  "generate_audio": false,
  "mouth_state": "no_mouth"
}
```

**示例请求（模式2：图生视频-首尾帧，首帧+尾帧）**:

```json
{
  "images": [
    {
      "url": "https://cdn.example.com/standing.png",
      "role": "first_frame"
    },
    {
      "url": "https://cdn.example.com/sitting.png",
      "role": "last_frame"
    }
  ],
  "prompt": "人物从站立姿势缓缓坐下，表情从平静变为沉思，环境光线逐渐变暗，二次元插画风格",
  "provider": "volcengine",
  "model": "doubao-seedance-2-0-260128",
  "resolution": "720p",
  "ratio": "9:16",
  "duration": 8,
  "generate_audio": false
}
```

**示例请求（模式3：多模态参考生视频，多张参考图+参考视频+参考音频）**:

```json
{
  "images": [
    {
      "url": "https://cdn.example.com/character_ref1.png",
      "role": "reference_image"
    },
    {
      "url": "https://cdn.example.com/scene_ref.png",
      "role": "reference_image"
    },
    {
      "url": "https://cdn.example.com/first_frame.png",
      "role": "reference_image"
    }
  ],
  "reference_videos": [
    {
      "url": "https://cdn.example.com/ref_motion.mp4"
    }
  ],
  "reference_audios": [
    {
      "url": "https://cdn.example.com/bg_music.mp3"
    }
  ],
  "prompt": "角色在梦境空间中行走，手持帕子，光线朦胧昏暗，二次元插画风格",
  "provider": "volcengine",
  "model": "doubao-seedance-2-0-260128",
  "resolution": "720p",
  "ratio": "9:16",
  "duration": 10,
  "generate_audio": true
}
```

---

#### 6.4.2 generate_video_with_reference（参考生视频）

**功能描述**: 基于参考视频和场景/角色/道具/描述信息生成新视频。对齐 Seedance 多模态参考生视频 和 wan2.6-r2v。

**功能说明**: 参考生视频模式通过提供参考视频/图片，继承其运动风格、节奏、镜头语言等特征，生成与参考视频风格一致的新视频内容。支持多模态输入（参考图片0-9张+参考视频0-3个+参考音频0-3段）。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_video_with_reference_input",
  "required": ["prompt"],
  "properties": {
    "prompt": {
      "type": "string",
      "description": "视频生成提示词"
    },
    "provider": {
      "type": "string",
      "enum": ["volcengine", "alibaba"],
      "default": "volcengine"
    },
    "model": {
      "type": "string",
      "examples": ["doubao-seedance-2-0-260128", "wan2.6-r2v"]
    },
    "resolution": {
      "type": "string",
      "enum": ["480p", "720p", "1080p"],
      "default": "720p"
    },
    "ratio": {
      "type": "string",
      "enum": ["16:9", "4:3", "1:1", "3:4", "9:16", "21:9", "adaptive"],
      "default": "9:16"
    },
    "duration": {
      "type": "integer",
      "description": "视频时长（秒）"
    },
    "reference_content": {
      "type": "object",
      "description": "参考内容（对齐Seedance的content数组结构）",
      "properties": {
        "reference_images": {
          "type": "array",
          "description": "参考图片列表（0-9张）",
          "maxItems": 9,
          "items": {
            "type": "object",
            "properties": {
              "url": {
                "type": "string",
                "description": "图片URL/Base64/素材ID"
              },
              "role": {
                "type": "string",
                "enum": ["first_frame", "last_frame", "reference_image"],
                "description": "角色：首帧/尾帧/参考图"
              }
            }
          }
        },
        "reference_videos": {
          "type": "array",
          "description": "参考视频列表（0-3个，总时长≤15s）",
          "maxItems": 3,
          "items": {
            "type": "object",
            "properties": {
              "url": {
                "type": "string",
                "description": "视频URL/素材ID"
              },
              "role": {
                "type": "string",
                "enum": ["reference_video"],
                "default": "reference_video"
              }
            }
          }
        },
        "reference_audios": {
          "type": "array",
          "description": "参考音频列表（0-3段，总时长≤15s）",
          "maxItems": 3,
          "items": {
            "type": "object",
            "properties": {
              "url": {
                "type": "string",
                "description": "音频URL/Base64/素材ID"
              },
              "role": {
                "type": "string",
                "enum": ["reference_audio"],
                "default": "reference_audio"
              }
            }
          }
        }
      }
    },
    "generate_audio": {
      "type": "boolean",
      "default": true
    },
    "reference_strength": {
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "default": 0.5,
      "description": "参考强度（控制参考视频特征的影响程度）"
    },
    "seed": {"type": "integer"},
    "watermark": {"type": "boolean", "default": false},
    "callback_url": {"type": "string", "format": "uri"}
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_video_with_reference_output",
  "properties": {
    "success": {"type": "boolean"},
    "task_id": {"type": "string"},
    "status": {"type": "string", "enum": ["pending", "processing", "completed", "failed"]},
    "metadata": {
      "type": "object",
      "properties": {
        "provider": {"type": "string"},
        "model": {"type": "string"},
        "resolution": {"type": "string"},
        "ratio": {"type": "string"},
        "duration": {"type": "integer"},
        "reference_strength": {"type": "number"},
        "created_at": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

**任务完成后响应**:

```json
{
  "success": true,
  "task_id": "cgt-20260414130000-*****",
  "status": "completed",
  "result": {
    "video_url": "https://ark.doubao.com/api/v3/files/xxxxx/video.mp4",
    "duration": 5,
    "width": 720,
    "height": 1280,
    "format": "MP4"
  },
  "metadata": {
    "provider": "volcengine",
    "model": "doubao-seedance-2-0-260128",
    "reference_strength": 0.6,
    "created_at": "2025-05-11T11:00:00+08:00"
  }
}
```

**示例请求（多模态参考，Seedance 2.0）**:

```json
{
  "prompt": "生成与参考视频风格一致的人物行走场景，人物外观参考角色图片，动作参考参考视频1，节奏参考参考视频2，二次元插画风格",
  "provider": "volcengine",
  "model": "doubao-seedance-2-0-260128",
  "resolution": "720p",
  "ratio": "9:16",
  "duration": 5,
  "reference_content": {
    "reference_images": [
      {
        "url": "https://cdn.example.com/characters/char_001.png",
        "role": "reference_image"
      }
    ],
    "reference_videos": [
      {
        "url": "https://cdn.example.com/reference/walking_style.mp4",
        "role": "reference_video"
      },
      {
        "url": "https://cdn.example.com/reference/rhythm_ref.mp4",
        "role": "reference_video"
      }
    ]
  },
  "reference_strength": 0.6,
  "generate_audio": true
}
```

**示例请求（万相 r2v）**:

```json
{
  "prompt": "保持参考视频中人物的动作风格和场景氛围，生成新的人物形象",
  "provider": "alibaba",
  "model": "wan2.6-r2v",
  "resolution": "720P",
  "duration": 5,
  "reference_content": {
    "reference_images": [
      {
        "url": "https://cdn.example.com/characters/new_char.png",
        "role": "first_frame"
      }
    ],
    "reference_videos": [
      {
        "url": "https://cdn.example.com/reference/style_video.mp4",
        "role": "reference_video"
      }
    ]
  },
  "reference_strength": 0.5
}
```

---

#### 6.4.3 generate_video_text2video（文生视频）

**功能描述**: 仅基于文本提示词生成视频，无需图片输入。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_video_text2video_input",
  "required": ["prompt"],
  "properties": {
    "prompt": {
      "type": "string",
      "description": "视频生成提示词"
    },
    "provider": {
      "type": "string",
      "enum": ["volcengine", "alibaba"],
      "default": "volcengine"
    },
    "model": {
      "type": "string",
      "examples": ["doubao-seedance-2-0-260128", "wan2.6-t2v"]
    },
    "resolution": {
      "type": "string",
      "enum": ["480p", "720p", "1080p"],
      "default": "720p"
    },
    "ratio": {
      "type": "string",
      "enum": ["16:9", "4:3", "1:1", "3:4", "9:16", "21:9", "adaptive"],
      "default": "16:9"
    },
    "duration": {
      "type": "integer",
      "description": "视频时长（秒）"
    },
    "generate_audio": {
      "type": "boolean",
      "default": true
    },
    "seed": {"type": "integer"},
    "watermark": {"type": "boolean", "default": false}
  }
}
```

**示例请求**:

```json
{
  "prompt": "一个年轻女孩站在樱花树下，微风拂过，花瓣飘落，阳光透过树叶洒下斑驳光影，二次元动漫风格，画面唯美梦幻",
  "provider": "volcengine",
  "model": "doubao-seedance-2-0-260128",
  "resolution": "720p",
  "ratio": "16:9",
  "duration": 5,
  "generate_audio": false
}
```

---

#### 6.4.4 generate_lip_sync_video（口型驱动）

**功能描述**: 基于人物图片和音频驱动口型动作，生成说话/唱歌/表演视频。对齐 wan2.2-s2v 数字人口型驱动能力。

**功能说明**: 这是PRD中"嘴巴状态标注"的核心技术实现。通过输入角色图片和配音音频，自动生成与音频同步的口型动画。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_lip_sync_video_input",
  "required": ["image_url", "audio_url"],
  "properties": {
    "image_url": {
      "type": "string",
      "description": "人物图片URL（建议正面照，效果最佳）"
    },
    "audio_url": {
      "type": "string",
      "description": "配音音频URL"
    },
    "prompt": {
      "type": "string",
      "description": "可选的提示词，用于补充动作描述"
    },
    "provider": {
      "type": "string",
      "enum": ["alibaba"],
      "default": "alibaba"
    },
    "model": {
      "type": "string",
      "enum": ["wan2.2-s2v"],
      "default": "wan2.2-s2v"
    },
    "resolution": {
      "type": "string",
      "enum": ["480P", "720P"],
      "default": "720P"
    },
    "watermark": {
      "type": "boolean",
      "default": false
    }
  }
}
```

**音频要求（wan2.2-s2v）**:

| 参数 | 要求 |
|------|------|
| 格式 | wav/mp3 |
| 时长 | <20s |
| 大小 | <15MB |

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_lip_sync_video_output",
  "properties": {
    "success": {"type": "boolean"},
    "task_id": {"type": "string"},
    "status": {"type": "string", "enum": ["pending", "processing", "completed", "failed"]},
    "metadata": {
      "type": "object",
      "properties": {
        "provider": {"type": "string"},
        "model": {"type": "string"},
        "audio_duration": {"type": "number"},
        "created_at": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

**示例请求**:

```json
{
  "image_url": "https://cdn.example.com/characters/char_001.png",
  "audio_url": "https://cdn.example.com/audio/dialogue_001.wav",
  "prompt": "角色表情自然，口型与音频同步，微微点头配合说话",
  "provider": "alibaba",
  "model": "wan2.2-s2v",
  "resolution": "720P",
  "watermark": false
}
```

**示例响应**:

```json
{
  "success": true,
  "task_id": "task_20260511001",
  "status": "completed",
  "result": {
    "video_url": "https://dashscope.aliyuncs.com/api/v1/videos/task_20260511001.mp4",
    "duration": 8.5,
    "width": 720,
    "height": 1280,
    "format": "MP4"
  },
  "metadata": {
    "provider": "alibaba",
    "model": "wan2.2-s2v",
    "audio_duration": 8.5,
    "created_at": "2025-05-11T11:05:00+08:00"
  }
}
```

---

#### 6.4.5 generate_multi_shot_video（多镜头叙事）

**功能描述**: 在单个视频片段中自动切换多个镜头/场景。对齐 wan2.6-i2v 的 shot_type=multi 能力。

**功能说明**: 适用于需要多角度叙事或快速场景切换的分镜，可大幅减少分镜数量。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "generate_multi_shot_video_input",
  "required": ["image_url", "prompt"],
  "properties": {
    "image_url": {
      "type": "string",
      "description": "首帧图片URL"
    },
    "prompt": {
      "type": "string",
      "description": "包含多镜头描述的提示词"
    },
    "shot_descriptions": {
      "type": "array",
      "description": "各镜头描述列表",
      "items": {
        "type": "object",
        "properties": {
          "shot_type": {
            "type": "string",
            "enum": ["EWS", "WS", "MS", "CU", "ECU"],
            "description": "景别：极远景/全景/中景/近景/特写"
          },
          "camera": {
            "type": "string",
            "enum": ["static", "pan_left", "pan_right", "dolly_in", "dolly_out"],
            "description": "镜头运动"
          },
          "description": {
            "type": "string",
            "description": "该镜头描述"
          }
        }
      }
    },
    "provider": {
      "type": "string",
      "enum": ["alibaba"],
      "default": "alibaba"
    },
    "model": {
      "type": "string",
      "enum": ["wan2.7-i2v"],
      "default": "wan2.7-i2v"
    },
    "resolution": {
      "type": "string",
      "enum": ["720P", "1080P"],
      "default": "720P"
    },
    "duration": {
      "type": "integer",
      "description": "视频时长（秒），2-15s"
    },
    "prompt_extend": {
      "type": "boolean",
      "default": true,
      "description": "是否启用prompt智能改写"
    }
  }
}
```

**示例请求**:

```json
{
  "image_url": "https://cdn.example.com/scene/office.png",
  "prompt": "办公室场景，展现员工工作的多角度画面，镜头流畅切换",
  "shot_descriptions": [
    {
      "shot_type": "WS",
      "camera": "dolly_in",
      "description": "全景展示办公室环境"
    },
    {
      "shot_type": "MS",
      "camera": "pan_left",
      "description": "中景展示员工A工作"
    },
    {
      "shot_type": "CU",
      "camera": "static",
      "description": "特写展示电脑屏幕"
    },
    {
      "shot_type": "MS",
      "camera": "pan_right",
      "description": "中景展示员工B交流"
    }
  ],
  "provider": "alibaba",
  "model": "wan2.7-i2v",
  "resolution": "720P",
  "duration": 10,
  "prompt_extend": true
}
```

---

#### 6.4.6 query_video_task（查询任务状态）

**功能描述**: 查询视频生成任务的状态。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "query_video_task_input",
  "required": ["task_id"],
  "properties": {
    "task_id": {
      "type": "string",
      "description": "任务ID"
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "query_video_task_output",
  "properties": {
    "success": {"type": "boolean"},
    "task_id": {"type": "string"},
    "status": {
      "type": "string",
      "enum": ["pending", "queued", "processing", "running", "completed", "succeeded", "failed", "cancelled", "expired"],
      "description": "任务状态（统一映射后的状态）"
    },
    "progress": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "进度百分比"
    },
    "progress_message": {
      "type": "string",
      "description": "进度描述"
    },
    "result": {
      "type": "object",
      "description": "任务结果（仅当status为completed/succeeded时）",
      "properties": {
        "video_url": {"type": "string"},
        "last_frame_url": {"type": "string"},
        "duration": {"type": "number"},
        "width": {"type": "integer"},
        "height": {"type": "integer"},
        "file_size": {"type": "integer"},
        "format": {"type": "string"}
      }
    },
    "error": {
      "type": "object",
      "description": "错误信息（仅当status为failed时）",
      "properties": {
        "code": {"type": "string"},
        "message": {"type": "string"},
        "retryable": {"type": "boolean"}
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "provider": {"type": "string"},
        "provider_task_id": {"type": "string", "description": "平台原始任务ID"},
        "provider_status": {"type": "string", "description": "平台原始状态"},
        "model": {"type": "string"},
        "created_at": {"type": "string", "format": "date-time"},
        "updated_at": {"type": "string", "format": "date-time"},
        "processing_time_ms": {"type": "integer"}
      }
    }
  }
}
```

**状态映射表**:

| MCP统一状态 | 火山引擎状态 | 阿里云状态 |
|-------------|--------------|------------|
| pending | queued | PENDING |
| processing | running | RUNNING |
| completed | succeeded | SUCCEEDED |
| failed | failed | FAILED |
| cancelled | cancelled | - |
| expired | expired | - |

**示例请求**:

```json
{
  "task_id": "cgt-20260414114820-*****"
}
```

**示例响应（生成中）**:

```json
{
  "success": true,
  "task_id": "cgt-20260414114820-*****",
  "status": "processing",
  "progress": 45,
  "progress_message": "正在生成视频帧...",
  "metadata": {
    "provider": "volcengine",
    "provider_task_id": "cgt-20260414114820-*****",
    "provider_status": "running",
    "model": "doubao-seedance-2-0-260128",
    "created_at": "2025-05-11T10:55:00+08:00",
    "updated_at": "2025-05-11T10:57:30+08:00"
  }
}
```

**示例响应（已完成）**:

```json
{
  "success": true,
  "task_id": "cgt-20260414114820-*****",
  "status": "completed",
  "progress": 100,
  "result": {
    "video_url": "https://ark.doubao.com/api/v3/files/xxxxx/video.mp4",
    "last_frame_url": null,
    "duration": 5,
    "width": 720,
    "height": 1280,
    "file_size": 5242880,
    "format": "MP4"
  },
  "metadata": {
    "provider": "volcengine",
    "provider_task_id": "cgt-20260414114820-*****",
    "provider_status": "succeeded",
    "model": "doubao-seedance-2-0-260128",
    "created_at": "2025-05-11T10:55:00+08:00",
    "updated_at": "2025-05-11T11:00:00+08:00",
    "processing_time_ms": 45000
  }
}
```

**⚠️ 重要提示**:

> **视频URL有效期**: 火山引擎视频URL有效期为24小时，阿里云可能更短。请在生成完成后及时转存到自有CDN或云存储。

---

#### 6.4.7 download_video（下载视频）

**功能描述**: 获取视频的直接下载链接或下载视频文件。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "download_video_input",
  "required": ["task_id"],
  "properties": {
    "task_id": {
      "type": "string",
      "description": "已完成任务的ID"
    },
    "format": {
      "type": "string",
      "enum": ["mp4", "webm", "mov"],
      "default": "mp4",
      "description": "下载格式"
    },
    "quality": {
      "type": "string",
      "enum": ["high", "medium", "low"],
      "default": "high",
      "description": "视频质量"
    }
  }
}
```

**输出schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "download_video_output",
  "properties": {
    "success": {"type": "boolean"},
    "download_url": {
      "type": "string",
      "description": "直接下载链接（有效期通常为1小时，部分平台仅24小时）"
    },
    "expires_at": {
      "type": "string",
      "format": "date-time",
      "description": "链接过期时间"
    },
    "file_info": {
      "type": "object",
      "properties": {
        "format": {"type": "string"},
        "duration": {"type": "number"},
        "file_size": {"type": "integer"},
        "bitrate": {"type": "integer"}
      }
    }
  }
}
```

**⚠️ 重要提示**:

> **下载链接有效期**: 火山引擎视频URL有效期仅为24小时。请在生成完成后立即转存，或使用 callback_url 接收完成通知后立即下载。

---

#### 6.4.8 retry_generation（重试生成）

**功能描述**: 保留原始参数，重新调用视频生成接口。

**输入参数schema**:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "retry_generation_input",
  "required": ["original_task_id"],
  "properties": {
    "original_task_id": {
      "type": "string",
      "description": "原始任务ID"
    },
    "modify_params": {
      "type": "object",
      "description": "需要修改的参数（可选）",
      "properties": {
        "prompt": {"type": "string"},
        "duration": {"type": "integer"},
        "seed": {"type": "integer"},
        "reference_strength": {"type": "number"}
      }
    }
  }
}
```

**输出schema**: 与对应的生成工具输出格式相同

**说明**: 
- 系统会保留原始任务的所有参数
- 如果提供了 `modify_params`，会用新参数覆盖原参数
- 返回新的task_id
- 原任务记录会被标记为"已重试"

---

### 6.5 完整API对应关系表

| MCP Tool | 火山引擎 Seedance | 阿里云 百炼万相 | 说明 |
|----------|-------------------|-----------------|------|
| generate_video_from_images | 图生视频-首帧/首尾帧/多模态参考 | wan2.7-i2v / wan2.6-i2v | 多参图片生视频（1-9张图） |
| generate_video_with_reference | 多模态参考生视频 | wan2.6-r2v | 参考视频+图片（企业级场景） |
| generate_video_text2video | 文生视频 | wan2.6-t2v | 纯文本生成 |
| generate_lip_sync_video | - | wan2.2-s2v | 口型驱动 |
| generate_multi_shot_video | - | wan2.7-i2v (shot_type=multi) | 多镜头叙事 |
| query_video_task | GET /tasks/{id} | GET /tasks/{id} | 状态查询 |
| download_video | 视频URL下载 | 视频URL下载 | 下载视频 |
| retry_generation | 重建任务 | 重建任务 | 重试生成 |

**功能重叠说明**：`generate_video_with_reference`（参考生视频）侧重于"参考视频+角色/场景/道具资源选择"的企业级场景，而 `generate_video_from_images`（多参图片生视频）侧重于API原生模式的多图+音视频素材。两者定位不同，按需选用。

---

## 7. 通用规范

### 7.1 认证方式

所有MCP Server必须实现以下认证机制：

```json
{
  "authentication": {
    "methods": [
      {
        "type": "bearer_token",
        "header": "Authorization",
        "scheme": "Bearer",
        "description": "Bearer Token认证"
      },
      {
        "type": "api_key",
        "header": "X-API-Key",
        "description": "API Key认证"
      }
    ],
    "required": true,
    "token_expiry": 3600
  }
}
```

### 7.2 错误处理规范

#### 7.2.1 统一错误响应格式

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述",
    "details": {},
    "retryable": true,
    "trace_id": "uuid"
  }
}
```

#### 7.2.2 视频生成专属错误码

| 错误码 | 描述 | 来源 | 处理建议 |
|--------|------|------|----------|
| VID101 | 请求体超过64MB | Seedance | 压缩图片 |
| VID102 | 单图片超过30MB | Seedance | 压缩图片 |
| VID103 | 单视频超过50MB | Seedance | 压缩视频 |
| VID104 | 单音频超过15MB | Seedance | 压缩音频 |
| VID105 | 参考视频总时长超过15s | Seedance | 缩短参考视频 |
| VID106 | 图片宽高比超出(0.4,2.5) | Seedance | 调整图片尺寸 |
| VID107 | 音频时长超过20s | 万相s2v | 截断音频 |
| VID108 | 音频时长超过15s | Seedance | 截断音频 |
| VID201 | 任务超时 | 平台 | 重试 |
| VID202 | 账户余额不足 | 平台 | 充值 |
| VID203 | QPS超限 | 平台 | 限流或升级 |

### 7.3 超时策略

| 操作类型 | 默认超时 | 最大超时 | 说明 |
|----------|----------|----------|------|
| 同步工具调用 | 30s | 60s | 如校验、分析类 |
| 图片生成 | 60s | 120s | 包含模型推理时间 |
| 视频生成 | 300s | 600s | 视频生成时间较长 |
| 资源查询 | 10s | 30s | 读取资源定义 |

### 7.4 并发限制

```json
{
  "rate_limits": {
    "per_user": {
      "requests_per_minute": 60,
      "requests_per_hour": 1000,
      "concurrent_tasks": 5
    },
    "video_generation": {
      "max_pending_tasks": 10,
      "max_concurrent": 3,
      "per_model": {
        "doubao-seedance-2-0-260128": {"max_concurrent": 2},
        "wan2.7-i2v": {"max_concurrent": 2}
      }
    }
  }
}
```

---

## 8. 部署建议

### 8.1 部署架构选项

#### 8.1.1 本地部署方案

**适用场景**：
- 数据安全性要求高
- 网络延迟敏感
- 成本可控

**架构图**：

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端应用层                                 │
│                    (漫剧工作流系统)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MCP Gateway                                 │
│                  (统一接入层/负载均衡)                             │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Script        │   │ Image         │   │ Video         │
│ Cleaning      │   │ Generation    │   │ Generation    │
│ Server        │   │ Server        │   │ Server        │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ 本地LLM服务    │   │ 图片生成模型   │   │ API网关      │
│ (如vLLM)      │   │ (GPU服务器)    │   │ (代理三方API)  │
└───────────────┘   └───────────────┘   └───────────────┘
```

**资源配置建议**：

| Server | CPU | 内存 | GPU | 说明 |
|--------|-----|------|-----|------|
| Script Cleaning | 8核 | 16GB | - | LLM推理 |
| Script Analysis | 8核 | 16GB | - | LLM推理 |
| Image Generation | 4核 | 8GB | RTX 3090×1 | 图片模型 |
| Video Generation | 8核 | 16GB | - | 仅做API代理 |

#### 8.1.2 远程部署方案（推荐）

**适用场景**：
- 成本优化
- 弹性扩展
- 快速上线

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端应用层                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MCP Gateway                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ 远程LLM API   │   │ 图片生成服务   │   │ 视频生成API   │
│ (OpenAI兼容)  │   │ (自有GPU)     │   │ (三方API代理)  │
└───────────────┘   └───────────────┘   └───────────────┘
                                              │
                            ┌─────────────────┼─────────────────┐
                            ▼                 ▼                 ▼
                    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
                    │ 火山引擎    │   │ 阿里云百炼   │   │ 生数Vidu    │
                    │ Seedance    │   │ 万相        │   │ (可选)      │
                    └─────────────┘   └─────────────┘   └─────────────┘
```

### 8.2 视频生成API代理设计

```python
# video_api_proxy.py 示例

class VideoAPIGateway:
    """视频生成API统一网关"""
    
    PROVIDERS = {
        "volcengine": {
            "base_url": "https://ark.cn-beijing.volces.com/api/v3",
            "auth_header": {"Authorization": f"Bearer {API_KEY}"}
        },
        "alibaba": {
            "base_url": "https://dashscope.aliyuncs.com/api/v1",
            "auth_header": {
                "Authorization": f"Bearer {DASHSCOPE_API_KEY}",
                "X-DashScope-Async": "enable"
            }
        }
    }
    
    def create_task(self, provider: str, model: str, params: dict) -> str:
        """创建视频生成任务"""
        config = self.PROVIDERS[provider]
        
        if provider == "volcengine":
            return self._create_volcengine_task(model, params, config)
        elif provider == "alibaba":
            return self._create_alibaba_task(model, params, config)
    
    def _create_volcengine_task(self, model: str, params: dict, config: dict) -> str:
        """火山引擎任务创建"""
        # 对齐content数组结构
        content = self._build_volcengine_content(params)
        
        payload = {
            "model": model,
            "content": content,
            "resolution": params.get("resolution", "720p"),
            "ratio": params.get("ratio", "9:16"),
            "duration": params.get("duration", -1),
            "generate_audio": params.get("generate_audio", True),
            "seed": params.get("seed", -1),
            "watermark": params.get("watermark", False)
        }
        
        response = requests.post(
            f"{config['base_url']}/contents/generations/tasks",
            headers=config["auth_header"],
            json=payload
        )
        
        return response.json()["id"]
    
    def _create_alibaba_task(self, model: str, params: dict, config: dict) -> str:
        """阿里云百炼任务创建"""
        # 对齐万相的input/parameters结构
        payload = {
            "model": model,
            "input": {
                "prompt": params["prompt"],
                "img_url": params.get("image_url")
            },
            "parameters": {
                "resolution": params.get("resolution", "720P"),
                "duration": params.get("duration", 5)
            }
        }
        
        response = requests.post(
            f"{config['base_url']}/services/aigc/video-generation/video-synthesis",
            headers=config["auth_header"],
            json=payload
        )
        
        return response.json()["output"]["task_id"]
    
    def query_task(self, provider: str, task_id: str) -> dict:
        """查询任务状态"""
        config = self.PROVIDERS[provider]
        
        if provider == "volcengine":
            url = f"{config['base_url']}/contents/generations/tasks/{task_id}"
        elif provider == "alibaba":
            url = f"{config['base_url']}/tasks/{task_id}"
        
        response = requests.get(url, headers=config["auth_header"])
        
        return self._normalize_status(response.json())
    
    def _normalize_status(self, raw_response: dict) -> dict:
        """统一状态映射"""
        # 将各平台状态映射为MCP统一状态
        volcengine_map = {
            "queued": "pending",
            "running": "processing",
            "succeeded": "completed",
            "failed": "failed",
            "cancelled": "cancelled",
            "expired": "expired"
        }
        
        alibaba_map = {
            "PENDING": "pending",
            "RUNNING": "processing",
            "SUCCEEDED": "completed",
            "FAILED": "failed"
        }
        
        # ... 状态转换逻辑
```

---

## 9. 前端集成方案

### 9.1 集成架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端应用                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  剧本列表    │  │  剧本解析    │  │  分镜列表    │  │  融图管理    │        │
│  │  组件        │  │  组件        │  │  组件        │  │  组件        │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│          │                │                │                │              │
│          └────────────────┴────────────────┴────────────────┘              │
│                                      │                                        │
│                              ┌───────┴───────┐                                │
│                              │  MCP Client   │                                │
│                              │    SDK        │                                │
│                              └───────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            MCP Server Layer                                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐ │
│  │ Script         │  │ Script        │  │ Image         │  │ Video         │ │
│  │ Cleaning       │  │ Analysis      │  │ Generation    │  │ Generation    │ │
│  └───────────────┘  └───────────────┘  └───────────────┘  └───────────────┘ │
│                                                                              │
│                            Video Generation Server                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  Provider: volcengine | alibaba                                          ││
│  │  Models: Seedance 2.0 | wan2.7-i2v | wan2.2-s2v | ...                    ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 MCP Client SDK使用示例

#### 9.2.1 初始化SDK

```javascript
// mcp-client.js
import { MCPSDK } from '@comic-workflow/mcp-sdk';

const mcpClient = new MCPSDK({
  servers: {
    scriptCleaning: {
      endpoint: process.env.SCRIPT_CLEANING_ENDPOINT,
      auth: { type: 'bearer_token', token: process.env.MCP_TOKEN }
    },
    scriptAnalysis: {
      endpoint: process.env.SCRIPT_ANALYSIS_ENDPOINT,
      auth: { type: 'bearer_token', token: process.env.MCP_TOKEN }
    },
    imageGeneration: {
      endpoint: process.env.IMAGE_GENERATION_ENDPOINT,
      auth: { type: 'bearer_token', token: process.env.MCP_TOKEN }
    },
    videoGeneration: {
      endpoint: process.env.VIDEO_GENERATION_ENDPOINT,
      auth: { type: 'bearer_token', token: process.env.MCP_TOKEN }
    }
  },
  retry: { maxRetries: 3, initialDelay: 1000, maxDelay: 10000 }
});

export default mcpClient;
```

#### 9.2.2 视频生成流程

```javascript
// videoGeneration.js

/**
 * 视频生成工厂，根据场景选择最佳模型和工具
 */
class VideoGenerationFactory {
  
  // 获取最佳图生视频配置（支持首帧、首尾帧、多参参考三种模式）
  static getImageToVideoConfig(scene, options = {}) {
    const { images = [] } = options;
    return {
      tool: 'generate_video_from_images',
      provider: 'volcengine', // 默认用Seedance
      model: 'doubao-seedance-2-0-260128',
      params: {
        resolution: '720p',
        ratio: scene.aspect_ratio || '9:16',
        duration: this.mapDuration(scene.estimated_duration),
        generate_audio: true,
        mouth_state: scene.mouth_state,
        ...options
      }
    };
  }
  
  // 获取首尾帧视频配置
  static getFirstLastFrameConfig(firstFrameUrl, lastFrameUrl, prompt, options = {}) {
    return {
      tool: 'generate_video_from_images',
      provider: 'volcengine',
      model: 'doubao-seedance-2-0-260128',
      params: {
        images: [
          { url: firstFrameUrl, role: 'first_frame' },
          { url: lastFrameUrl, role: 'last_frame' }
        ],
        prompt,
        resolution: '720p',
        ratio: '9:16',
        duration: 8,
        generate_audio: false,
        ...options
      }
    };
  }
  
  // 获取口型驱动配置
  static getLipSyncConfig(characterImage, audioUrl) {
    return {
      tool: 'generate_lip_sync_video',
      provider: 'alibaba',
      model: 'wan2.2-s2v',
      params: {
        image_url: characterImage,
        audio_url: audioUrl,
        resolution: '720P'
      }
    };
  }
  
  // 获取参考生视频配置
  static getReferenceVideoConfig(scene, referenceVideo, referenceImage) {
    return {
      tool: 'generate_video_with_reference',
      provider: 'volcengine',
      model: 'doubao-seedance-2-0-260128',
      params: {
        resolution: '720p',
        ratio: scene.aspect_ratio || '9:16',
        duration: 5,
        reference_content: {
          reference_images: referenceImage ? [{
            url: referenceImage,
            role: 'reference_image'
          }] : [],
          reference_videos: [{
            url: referenceVideo,
            role: 'reference_video'
          }]
        },
        reference_strength: 0.6
      }
    };
  }
  
  static mapDuration(seconds) {
    if (seconds <= 4) return 4;
    if (seconds <= 5) return 5;
    if (seconds <= 6) return 6;
    if (seconds <= 8) return 8;
    return 10;
  }
}

// 视频生成服务
class VideoGenerationService {
  
  async generateVideo(task) {
    const { type, data, config } = task;
    const factory = VideoGenerationFactory;
    
    let result;
    
    switch (type) {
      case 'image_to_video':
        result = await this.generateFromImage(data, config);
        break;
      case 'first_last_frame':
        result = await this.generateFirstLastFrame(data, config);
        break;
      case 'with_reference':
        result = await this.generateWithReference(data, config);
        break;
      case 'lip_sync':
        result = await this.generateLipSync(data, config);
        break;
      case 'text_to_video':
        result = await this.generateText2Video(data, config);
        break;
      default:
        throw new Error(`Unknown video generation type: ${type}`);
    }
    
    return result;
  }
  
  async generateFromImage(data, config) {
    const { compositeImageUrl, prompt, mouthState } = data;
    const { provider, model, params } = config;
    
    // 自动添加口型控制
    let finalPrompt = prompt;
    if (mouthState === 'no_mouth' || mouthState === 'closed') {
      finalPrompt = `${prompt}。全程人物无对白口型，嘴巴不动`;
    }
    
    // 使用新的多参图片生视频接口
    const result = await mcpClient.videoGeneration.generate_video_from_images({
      images: [{ url: compositeImageUrl, role: 'first_frame' }],
      prompt: finalPrompt,
      provider,
      model,
      ...params
    });
    
    // 轮询直到完成
    return await this.pollUntilComplete(result.task_id);
  }
  
  async generateFirstLastFrame(data, config) {
    const { firstFrameUrl, lastFrameUrl, prompt } = data;
    const { provider, model, params } = config;
    
    const result = await mcpClient.videoGeneration.generate_video_from_images({
      images: [
        { url: firstFrameUrl, role: 'first_frame' },
        { url: lastFrameUrl, role: 'last_frame' }
      ],
      prompt,
      provider,
      model,
      ...params
    });
    
    return await this.pollUntilComplete(result.task_id);
  }
  
  async generateLipSync(data, config) {
    const { characterImage, audioUrl, prompt } = data;
    
    const result = await mcpClient.videoGeneration.generate_lip_sync_video({
      image_url: characterImage,
      audio_url: audioUrl,
      prompt,
      ...config.params
    });
    
    return await this.pollUntilComplete(result.task_id);
  }
  
  async pollUntilComplete(taskId, maxAttempts = 120) {
    for (let i = 0; i < maxAttempts; i++) {
      const status = await mcpClient.videoGeneration.query_video_task({
        task_id: taskId
      });
      
      if (status.status === 'completed') {
        // ⚠️ 重要：视频URL有效期24小时，及时转存
        const videoUrl = await this.transferToOwnCDN(status.result.video_url);
        return { ...status.result, video_url: videoUrl };
      }
      
      if (status.status === 'failed') {
        throw new Error(`Video generation failed: ${status.error?.message}`);
      }
      
      // 指数退避
      await new Promise(resolve => setTimeout(resolve, 2000 + i * 100));
    }
    
    throw new Error('Video generation timeout');
  }
  
  async transferToOwnCDN(url) {
    // 实现自己的CDN转存逻辑
    const response = await fetch(url);
    const blob = await response.blob();
    const uploadedUrl = await yourCDN.upload(blob);
    return uploadedUrl;
  }
}

export { VideoGenerationFactory, VideoGenerationService };
```

#### 9.2.3 完整工作流示例

```javascript
// workflow.js

async function generateComicEpisode(scriptId) {
  const workflow = new VideoGenerationService();
  
  // 1. 获取剧本数据
  const script = await getScript(scriptId);
  
  // 2. 清洗剧本
  const cleanedScript = await mcpClient.scriptCleaning.clean_script({
    raw_script: script.raw_text,
    script_type: script.type
  });
  
  // 3. 分析剧本
  const analysis = await mcpClient.scriptAnalysis.generate_storyboard_prompts({
    script_data: cleanedScript.data,
    characters: characters,
    scenes: scenes,
    options: {
      default_style: '二次元插画风格',
      aspect_ratio: '9:16',
      include_mouth_control: true
    }
  });
  
  // 4. 生成融图
  const compositeImages = await generateCompositeImages(analysis.storyboards);
  
  // 5. 生成视频
  const videos = [];
  for (const sb of analysis.storyboards) {
    const composite = compositeImages.find(c => c.storyboard_id === sb.storyboard_id);
    
    // 根据场景选择最佳视频生成方式
    let task;
    
    if (sb.voice_type === 'dialogue') {
      // 对话场景：先生成融图视频，再口型驱动
      const videoTask = await workflow.generateVideo({
        type: 'image_to_video',
        data: {
          compositeImageUrl: composite.image_url,
          prompt: sb.video_prompt,
          mouthState: 'speaking'
        },
        config: VideoGenerationFactory.getImageToVideoConfig(sb)
      });
      
      // 口型驱动
      const lipSyncTask = await workflow.generateVideo({
        type: 'lip_sync',
        data: {
          characterImage: composite.character_image_url,
          audioUrl: sb.dialogue_audio_url,
          prompt: `角色${sb.emotion}地说：${sb.dialogue}`
        },
        config: VideoGenerationFactory.getLipSyncConfig()
      });
      
      videos.push(lipSyncTask);
    } else if (sb.has_last_frame && sb.last_frame_url) {
      // 需要首尾帧控制的场景
      const videoTask = await workflow.generateVideo({
        type: 'first_last_frame',
        data: {
          firstFrameUrl: composite.image_url,
          lastFrameUrl: sb.last_frame_url,
          prompt: sb.video_prompt
        },
        config: VideoGenerationFactory.getFirstLastFrameConfig(
          composite.image_url,
          sb.last_frame_url,
          sb.video_prompt
        )
      });
      
      videos.push(videoTask);
    } else {
      // 非对话场景：直接图生视频
      const videoTask = await workflow.generateVideo({
        type: 'image_to_video',
        data: {
          compositeImageUrl: composite.image_url,
          prompt: sb.video_prompt,
          mouthState: sb.mouth_state
        },
        config: VideoGenerationFactory.getImageToVideoConfig(sb)
      });
      
      videos.push(videoTask);
    }
  }
  
  // 6. 返回生成的视频列表
  return videos;
}
```

---

## 10. 附录

### 10.1 模型参数对照表

#### 火山引擎 Seedance 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| model | string | - | 模型ID，必填 |
| content | array | - | 内容数组，见下方 |
| resolution | string | 720p | 480p/720p/1080p |
| ratio | string | 9:16 | 16:9/4:3/1:1/3:4/9:16/21:9/adaptive |
| duration | integer | -1 | 4-15s或-1智能 |
| generate_audio | boolean | true | 是否有声（2.0/1.5 pro） |
| seed | integer | -1 | 随机种子 |
| camera_fixed | boolean | false | 固定镜头 |
| watermark | boolean | false | 水印 |
| callback_url | string | - | 回调URL |
| return_last_frame | boolean | false | 返回尾帧 |
| service_tier | string | default | default/flex |

**content数组结构**:

```json
[
  {"type": "text", "text": "提示词"},
  {"type": "image_url", "image_url": {"url": "...", "role": "first_frame/last_frame/reference_image"}},
  {"type": "video_url", "video_url": {"url": "...", "role": "reference_video"}},
  {"type": "audio_url", "audio_url": {"url": "...", "role": "reference_audio"}}
]
```

#### 阿里云百炼 万相参数

| 模型 | 参数结构 | 分辨率 | 时长 |
|------|----------|--------|------|
| wan2.7-i2v | media array + prompt | 720P/1080P | 2-15s |
| wan2.6-i2v | input.prompt + input.img_url | 480P/720P/1080P | 5/10/15s |
| wan2.6-r2v | input + parameters | 720P/1080P | 5/10s |
| wan2.2-s2v | input.image_url + input.audio_url | 480P/720P | - |

### 10.2 图片/视频/音频规格

| 类型 | 格式 | 尺寸 | 大小 | 其他 |
|------|------|------|------|------|
| 图片 | jpeg/png/webp/bmp/tiff/gif/heic/heif | 300-6000px, 宽高比(0.4,2.5) | 单张<30MB, 请求<64MB | - |
| 视频 | mp4/mov (H.264/H.265) | 480p/720p/1080p | 单视频<50MB | 帧率[24,60] |
| 音频 | wav/mp3 | - | 单音频<15MB | 时长[2,15]s |

### 10.3 嘴巴状态枚举

| 值 | 描述 | 技术实现 |
|----|------|----------|
| no_mouth | 无嘴/旁白 | Seedance prompt控制 / 万相s2v |
| closed | 闭嘴 | Seedance prompt控制 |
| speaking | 说话中 | 万相s2v口型驱动 |
| mumbling | 含糊 | 万相s2v |
| laughing | 大笑 | 万相s2v |
| shouting | 喊叫 | 万相s2v |

### 10.4 景别枚举

| 值 | 英文 | 描述 |
|----|------|------|
| EWS | Extreme Wide Shot | 极远景 |
| WS | Wide Shot | 全景 |
| MS | Medium Shot | 中景 |
| MCU | Medium Close-Up | 中近景 |
| CU | Close-Up | 近景 |
| ECU | Extreme Close-Up | 特写 |
| Big CU | Big Close-Up | 大特写 |

### 10.5 参考资料

- 火山引擎方舟控制台: https://console.volcengine.com/ark
- 阿里云百炼: https://bailian.console.aliyun.com/
- 生数科技Vidu: https://www.shengshu-ai.com/
- MCP Protocol: https://modelcontextprotocol.io/

---

**文档结束**

*本文档由漫剧工作流系统技术团队维护，如有问题请联系相关负责人。*

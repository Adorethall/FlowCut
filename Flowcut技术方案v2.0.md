# FlowCut 技术方案
**文档版本**：v2.0  
**编写日期**：2025-01-15  
**更新说明**：重构混剪策略，解决「素材时长 ≈ 输出时长」场景下的剪辑质量问题  
**基于文档**：FlowCut PRD v2.0  
**目标读者**：后端开发、前端开发、DevOps、测试开发  
---
## 一、系统架构
### 1.1 整体架构
采用前后端分离 + 异步任务队列的经典架构，核心流程为：
```
浏览器（Vue 3） → FastAPI 后端 → PostgreSQL（元数据） → MinIO（视频文件） → Redis（缓存 + 消息队列） → Celery Worker（视频处理） → FFmpeg（剪辑引擎） → TTS 服务（配音生成）
```
### 1.2 架构图
```
┌─────────────────────────────────────────────────────────────┐
│                         Nginx 反向代理                        │
│                    （静态资源 + API 转发）                     │
├────────────────────────┬────────────────────────────────────┤
│     Vue 3 前端         │         FastAPI 后端                │
│     (Element Plus)     │    ┌─────────────────────┐         │
│                        │    │   Auth 中间件        │         │
│  ┌──────────┐          │    ├─────────────────────┤         │
│  │素材库模块 │          │    │ /api/materials/*   │         │
│  ├──────────┤          │    │ /api/tasks/*       │         │
│  │工作流模块 │          │    │ /api/collect/*     │         │
│  ├──────────┤          │    │ /api/preview/*     │         │
│  │任务管理   │          │    └────────┬────────────┘         │
│  ├──────────┤          │             │                      │
│  │素材采集   │          │    ┌────────▼────────────┐         │
│  └──────────┘          │    │   Celery Beat       │         │
│                        │    │   (定时/周期任务)     │         │
└────────────────────────┴────┼─────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────────┐
        │PostgreSQL│  │  MinIO   │  │    Redis     │
        │ 元数据    │  │ 视频文件  │  │ 缓存/队列    │
        └──────────┘  └──────────┘  └──────┬───────┘
                                          │
                              ┌───────────▼───────────┐
                              │    Celery Workers     │
                              │  ┌───────┐ ┌───────┐  │
                              │  │Worker1│ │Worker2│  │
                              │  └───┬───┘ └───┬───┘  │
                              │      │         │      │
                              │  ┌───▼─────────▼───┐  │
                              │  │   FFmpeg CLI    │  │
                              │  └────────────────┘  │
                              │  ┌────────────────┐  │
                              │  │  TTS Service   │  │
                              │  └────────────────┘  │
                              │  ┌────────────────┐  │
                              │  │ 运动检测引擎    │  │
                              │  │ (OpenCV)       │  │
                              │  └────────────────┘  │
                              └──────────────────────┘
```
### 1.3 技术栈明细
| 层级 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 前端框架 | Vue 3 | 3.4+ | SPA 应用框架 |
| UI 组件库 | Element Plus | 2.5+ | 后台管理型组件 |
| 状态管理 | Pinia | 2.1+ | 全局状态 |
| HTTP 客户端 | Axios | 1.6+ | 接口请求 |
| 后端框架 | FastAPI | 0.110+ | API 服务 |
| ASGI 服务器 | Uvicorn | 0.27+ | 生产部署 |
| ORM | SQLAlchemy | 2.0+ | 数据库操作 |
| 数据库 | PostgreSQL 15+ (含 pgvector) | 15+ | 结构化数据持久化 + 向量存储 |
| 对象存储 | MinIO | 最新版 | 视频文件存储 |
| 缓存/队列 | Redis | 7.0+ | 缓存 + Celery Broker |
| 任务队列 | Celery | 5.3+ | 异步任务调度 |
| 视频处理 | FFmpeg | 6.0+ | 核心剪辑引擎 |
| 画面分析 | OpenCV | 4.8+ | 素材运动检测、高光标注 |
| 采集脚本 | yt-dlp + Playwright | 最新版 | 视频下载 |
| 语音合成 | 阿里云智能语音 | - | TTS 配音生成 |
| 反向代理 | Nginx | 1.24+ | 静态资源 + API 转发 |
| 容器化 | Docker + Docker Compose | - | 部署编排 |
---
## 二、数据库设计
### 2.1 ER 关系
```
Material ──┐
Material ──┼── TaskMaterial ── Task ── TaskOutput
Material ──┘                         │
                                     │
CollectTask ── CollectResult ── Material
```
### 2.2 表结构
#### material（素材表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, 自增 | 主键 |
| title | VARCHAR(200) | NOT NULL | 素材标题 |
| file_key | VARCHAR(500) | NOT NULL, UNIQUE | MinIO 中的对象键 |
| file_size | INTEGER | NOT NULL | 文件大小（字节） |
| duration | FLOAT | NOT NULL | 视频时长（秒） |
| width | INTEGER | NOT NULL | 视频宽度 |
| height | INTEGER | NOT NULL | 视频高度 |
| fps | FLOAT | | 帧率 |
| format | VARCHAR(20) | | 容器格式（mp4/webm等） |
| codec | VARCHAR(50) | | 视频编码（h264/h265等） |
| source | VARCHAR(20) | NOT NULL | 来源平台枚举 |
| source_url | VARCHAR(1000) | | 原始链接 |
| md5_hash | VARCHAR(32) | NOT NULL, INDEX | 用于去重 |
| cover_key | VARCHAR(500) | | 封面图 MinIO 键 |
| type_tag | VARCHAR(30) | | 素材类型标签（用于混剪策略决策） |
| highlight_segments | JSONB | | 高光时段列表（运动检测产出） |
| motion_score | FLOAT | | 整体运动强度评分（0-100） |
| analysis_status | VARCHAR(20) | DEFAULT 'pending' | pending/analyzing/done/failed |
| uploaded_by | VARCHAR(50) | | 上传者 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 入库时间 |
**type_tag 枚举值说明：**
| 值 | 含义 | 混剪策略倾向 |
|----|------|-------------|
| `food_action` | 食物制作/动作类（倒奶茶、切肉） | 可高速变速（3-5x），适合快节奏 |
| `people_talking` | 人物说话/互动类 | 禁止变速，必须原速截取 |
| `scenery` | 风景/静态展示类（店面、街景） | 可中等变速（2-3x），适合做过渡 |
| `product_show` | 产品展示类（摆拍、特写） | 低速变速或不变速，保留细节 |
| `ambience` | 氛围类（热气、灯光、人群） | 可变速，适合做开头/结尾过渡 |
| `other` | 未分类 | 默认策略 |
**highlight_segments 字段 JSONB 结构：**
```json
[
  { "start": 3.2, "end": 8.5, "score": 45.3 },
  { "start": 11.0, "end": 14.2, "score": 32.1 }
]
```
- `start` / `end`：高光区间起止秒数
- `score`：该区间平均运动强度（帧差分均值，0-100）
#### material_tag（素材标签关联表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, 自增 | 主键 |
| material_id | BIGINT | FK → material.id, ON DELETE CASCADE | 素材ID |
| tag | VARCHAR(30) | NOT NULL | 标签文字 |
| UNIQUE(material_id, tag) | | | 防止重复标签 |
#### task（剪辑任务表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | VARCHAR(36) | PK | UUID 主键 |
| name | VARCHAR(200) | NOT NULL | 任务名称 |
| theme | VARCHAR(50) | NOT NULL | 广告主题 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'queue' | 状态枚举 |
| progress | INTEGER | DEFAULT 0 | 进度百分比 0-100 |
| log_text | VARCHAR(200) | | 当前处理阶段描述 |
| config | JSONB | NOT NULL | 剪辑配置（完整JSON） |
| subtitle_config | JSONB | | 字幕配置（预览用） |
| output_count | INTEGER | NOT NULL | 预期输出数量 |
| output_duration | INTEGER | NOT NULL | 单条时长（秒） |
| output_ratio | VARCHAR(10) | NOT NULL | 画面比例 |
| output_resolution | VARCHAR(10) | NOT NULL | 分辨率 |
| created_by | VARCHAR(50) | | 创建者 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| started_at | TIMESTAMPTZ | | 开始处理时间 |
| finished_at | TIMESTAMPTZ | | 完成时间 |
| error_msg | TEXT | | 失败原因 |
config 字段 JSONB 结构：
```json
{
  "voice": { "enabled": true, "style": "活力女声", "speed": "1.0", "volume": 80 },
  "subtitle": { "enabled": true, "position": "底部居中", "font": "思源黑体", "size": "中", "color": "白色", "stroke": true },
  "music": { "enabled": true, "style": "轻快", "volume": 30 },
  "transition": { "type": "溶解", "duration": "0.5s" },
  "output": { "count": 5, "duration": "15s", "ratio": "9:16", "resolution": "1080p" },
  "mix_strategy": {
    "style": "fast",
    "allow_speedup": true,
    "max_speed": "3x",
    "prefer_highlights": true
  }
}
```
#### task_material（任务-素材关联表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, 自增 | 主键 |
| task_id | VARCHAR(36) | FK → task.id, ON DELETE CASCADE | 任务ID |
| material_id | BIGINT | FK → material.id | 素材ID |
| sort_order | INTEGER | NOT NULL | 排序序号 |
| use_start | FLOAT | DEFAULT 0 | 使用起始秒 |
| use_end | FLOAT | | 使用结束秒（NULL表示完整使用） |
| UNIQUE(task_id, material_id) | | | 防止重复关联 |
#### task_output（任务产出表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, 自增 | 主键 |
| task_id | VARCHAR(36) | FK → task.id, ON DELETE CASCADE | 任务ID |
| index | INTEGER | NOT NULL | 序号（从1开始） |
| file_key | VARCHAR(500) | NOT NULL | 视频文件 MinIO 键 |
| file_size | INTEGER | NOT NULL | 文件大小（字节） |
| duration | FLOAT | NOT NULL | 实际时长（秒） |
| cover_key | VARCHAR(500) | | 成片封面 MinIO 键 |
| copy_text | VARCHAR(100) | | 使用的广告文案 |
| subtitle_segments | JSONB | | 字幕时间线数据 |
| edit_plan | JSONB | | 本条视频的剪辑方案记录 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 生成时间 |
subtitle_segments 字段 JSONB 结构：
```json
[
  { "text": "这杯奶茶", "start": 0.0, "end": 2.5 },
  { "text": "绝了", "start": 2.5, "end": 5.0 }
]
```
edit_plan 字段 JSONB 结构（记录每段素材的实际处理方式，用于追溯和调试）：
```json
[
  {
    "material_id": 1,
    "strategy": "speedup",
    "source_start": 0.0,
    "source_end": 15.0,
    "output_duration": 3.0,
    "speed": 5.0
  },
  {
    "material_id": 6,
    "strategy": "cut_highlight",
    "source_start": 2.1,
    "source_end": 7.5,
    "output_duration": 5.4,
    "speed": 1.0
  }
]
```
#### collect_task（采集任务表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | VARCHAR(36) | PK | UUID 主键 |
| keywords | JSONB | NOT NULL | 关键词列表 |
| platforms | JSONB | NOT NULL | 平台列表 |
| count_per_keyword | INTEGER | NOT NULL | 每关键词数量 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | pending/running/done/failed |
| progress | INTEGER | DEFAULT 0 | 进度 0-100 |
| total_found | INTEGER | DEFAULT 0 | 发现数量 |
| total_downloaded | INTEGER | DEFAULT 0 | 实际下载量 |
| total_deduplicated | INTEGER | DEFAULT 0 | 去重跳过量 |
| created_by | VARCHAR(50) | | 创建者 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
#### collect_result（采集结果明细表）
| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, 自增 | 主键 |
| collect_task_id | VARCHAR(36) | FK → collect_task.id | 采集任务ID |
| keyword | VARCHAR(100) | NOT NULL | 使用的关键词 |
| platform | VARCHAR(20) | NOT NULL | 来源平台 |
| source_url | VARCHAR(1000) | NOT NULL | 原始链接 |
| status | VARCHAR(20) | NOT NULL | success/skip_duplicate/skip_error |
| material_id | BIGINT | FK → material.id, NULLABLE | 关联到的素材（成功时） |
| error_msg | VARCHAR(500) | | 失败原因 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 采集时间 |
### 2.3 索引设计
```sql
-- 素材高频查询索引
CREATE INDEX idx_material_md5 ON material(md5_hash);           -- 去重查询
CREATE INDEX idx_material_source ON material(source);           -- 按来源筛选
CREATE INDEX idx_material_created ON material(created_at DESC); -- 按时间排序
CREATE INDEX idx_material_duration ON material(duration);       -- 按时长筛选
CREATE INDEX idx_material_type_tag ON material(type_tag);       -- 按素材类型筛选（混剪决策用）
CREATE INDEX idx_material_analysis ON material(analysis_status); -- 按分析状态筛选
-- 标签查询
CREATE INDEX idx_tag_material ON material_tag(material_id);
CREATE INDEX idx_tag_name ON material_tag(tag);
-- 任务状态索引
CREATE INDEX idx_task_status ON task(status, created_at DESC);  -- 任务列表筛选
-- 采集去重
CREATE INDEX idx_collect_result_url ON collect_result(source_url);
CREATE INDEX idx_collect_result_task ON collect_result(collect_task_id);
```
---
## 三、MinIO 存储设计
### 3.1 Bucket 规划
| Bucket | 用途 | 权限 |
|--------|------|------|
| `flowcut-materials` | 原始素材视频 | 私有 |
| `flowcut-covers` | 素材封面图 | 私有 |
| `flowcut-outputs` | 剪辑产出视频 | 私有 |
| `flowcut-output-covers` | 产出封面图 | 私有 |
| `flowcut-fonts` | 字体文件 | 私有 |
| `flowcut-music` | 背景音乐库 | 私有 |
| `flowcut-tmp` | 视频处理临时文件 | 私有 |
### 3.2 对象键命名规则
```
素材视频:    materials/{source}/{year}/{month}/{day}/{md5_hash}.mp4
素材封面:    covers/materials/{md5_hash}.jpg
产出视频:    outputs/{task_id}/{index:03d}_{theme}_广告_{ratio}_{resolution}.mp4
产出封面:    covers/outputs/{task_id}/{index:03d}.jpg
字体文件:    fonts/{font_name}.{ext}
背景音乐:    music/{style}/{filename}.mp3
临时文件:    tmp/{task_id}/{step}_{index}.mp4
```
### 3.3 预签名 URL 策略
前端不直接访问 MinIO，所有文件下载/预览通过后端生成预签名 URL：
```python
# 预签名 URL 有效期
COVER_URL_EXPIRE = 3600      # 封面图 1 小时
MATERIAL_URL_EXPIRE = 7200   # 素材预览 2 小时
OUTPUT_URL_EXPIRE = 86400    # 产出下载 24 小时
TMP_URL_EXPIRE = 3600        # 临时文件 1 小时
```
---
## 四、API 设计
### 4.1 素材库接口
#### `GET /api/materials` — 素材列表
**Query 参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| keyword | string | 否 | 搜索关键词（标题/标签模糊匹配） |
| sources | string | 否 | 来源平台，逗号分隔，如 `抖音,快手` |
| tags | string | 否 | 标签，逗号分隔 |
| type_tag | string | 否 | 素材类型，逗号分隔，如 `food_action,ambience` |
| duration_min | float | 否 | 最小时长（秒） |
| duration_max | float | 否 | 最大时长（秒） |
| resolution | string | 否 | 分辨率过滤，如 `1080p` |
| analysis_status | string | 否 | 分析状态过滤（`done` 表示已完成高光标注） |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页数量，默认 20，最大 50 |
| sort_by | string | 否 | 排序字段：`created_at` / `duration` / `file_size` / `motion_score` |
| sort_order | string | 否 | `desc` / `asc`，默认 `desc` |
**响应：**
```json
{
  "total": 156,
  "page": 1,
  "page_size": 20,
  "items": [
    {
      "id": 1,
      "title": "奶茶摇制过程特写",
      "cover_url": "https://minio.example.com/...?X-Amz-Signature=...",
      "duration": 8.0,
      "width": 1080,
      "height": 1920,
      "file_size": 13107200,
      "source": "抖音",
      "tags": ["奶茶", "制作"],
      "type_tag": "food_action",
      "motion_score": 52.3,
      "highlight_segments": [
        { "start": 2.1, "end": 6.8, "score": 45.2 },
        { "start": 7.0, "end": 8.0, "score": 38.1 }
      ],
      "analysis_status": "done",
      "created_at": "2025-01-15T14:30:00+08:00"
    }
  ]
}
```
#### `POST /api/materials/upload` — 手动上传素材
**Request:** `multipart/form-data`
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | File | 是 | 视频文件 |
| title | string | 否 | 标题（不填则用文件名） |
| tags | string | 否 | 标签，逗号分隔 |
后端自动执行：ffprobe 提取元信息 → 计算 MD5 去重 → 生成封面 → 入库 → **异步触发运动检测**。
> 注：运动检测不阻塞上传响应，通过 `analysis_status` 字段标识状态。未完成分析的素材在混剪时降级为"中间截取"策略。
#### `GET /api/materials/{id}/preview-url` — 获取预览地址
**响应：**
```json
{
  "url": "https://minio.example.com/materials/...?X-Amz-Signature=...",
  "expires_in": 7200
}
```
#### `POST /api/materials/batch-download` — 批量下载
**Request:**
```json
{ "material_ids": [1, 2, 3, 5] }
```
**响应：** 返回打包下载的预签名 URL（后端异步打包为 ZIP）。
```json
{
  "task_id": "zip_xxx",
  "status": "processing",
  "poll_url": "/api/materials/batch-download/zip_xxx/status"
}
```
#### `GET /api/materials/tags` — 获取所有标签
**响应：**
```json
{
  "tags": ["奶茶", "制作", "特写", "门店", "火锅", "食材", ...],
  "total": 32
}
```
#### `GET /api/materials/type-tags` — 获取素材类型标签列表
**响应：**
```json
{
  "type_tags": [
    { "value": "food_action", "label": "食物制作", "count": 8 },
    { "value": "people_talking", "label": "人物互动", "count": 5 },
    { "value": "scenery", "label": "风景静态", "count": 4 },
    { "value": "product_show", "label": "产品展示", "count": 6 },
    { "value": "ambience", "label": "氛围场景", "count": 3 },
    { "value": "other", "label": "其他", "count": 2 }
  ]
}
```
---
### 4.2 剪辑工作流接口
#### `POST /api/tasks` — 创建剪辑任务
**Request:**
```json
{
  "theme": "奶茶",
  "material_ids": [1, 2, 4, 5, 24],
  "material_order": [1, 2, 24, 4, 5],
  "config": {
    "voice": { "enabled": true, "style": "活力女声", "speed": "1.0", "volume": 80 },
    "subtitle": { "enabled": true, "position": "底部居中", "font": "思源黑体", "size": "中", "color": "白色", "stroke": true },
    "music": { "enabled": true, "style": "轻快", "volume": 30 },
    "transition": { "type": "溶解", "duration": "0.5s" },
    "output": { "count": 5, "duration": "15s", "ratio": "9:16", "resolution": "1080p" },
    "mix_strategy": {
      "style": "fast",
      "allow_speedup": true,
      "max_speed": "3x",
      "prefer_highlights": true
    }
  }
}
```
**mix_strategy 字段说明：**
| 参数 | 类型 | 可选值 | 默认值 | 说明 |
|------|------|--------|--------|------|
| style | string | `fast` / `normal` / `slow` | `normal` | 混剪风格，影响整体策略倾向 |
| allow_speedup | bool | true / false | true | 是否允许变速处理 |
| max_speed | string | `2x` / `3x` / `5x` | `3x` | 最大变速倍率 |
| prefer_highlights | bool | true / false | true | 优先使用高光片段（需素材已完成分析） |
**style 对策略的具体影响：**
| style | 变速倾向 | 单段目标时长 | 转场时长 | 适用场景 |
|-------|---------|-------------|---------|---------|
| `fast` | 激进（优先高速压缩） | 2-4秒 | 0.3s | 抖音信息流 |
| `normal` | 适中 | 3-6秒 | 0.5s | 通用 |
| `slow` | 保守（少变速，长段截取） | 5-10秒 | 0.8s | 小红书/B站 |
**响应：**
```json
{
  "task_id": "task_abc123",
  "name": "奶茶_混剪_1430",
  "status": "queue",
  "estimated_time": 120,
  "warnings": [
    "3 条素材尚未完成运动分析，将降级使用中间截取策略"
  ]
}
```
后端校验逻辑：
- 素材数量 ≥ 3
- **素材总时长 ≥ 输出时长 × 30%**（低于此值即使用变速也很难混剪，需明确警告）
- 输出数量 1-50
- 分辨率与比例兼容性校验
- 若存在 `type_tag = people_talking` 的素材且 `allow_speedup = true`，自动将该素材标记为不可变速
#### `GET /api/tasks` — 任务列表
**Query 参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| status | string | 否 | 状态过滤 |
| page | int | 否 | 页码 |
| page_size | int | 否 | 每页数量 |
#### `GET /api/tasks/{id}` — 任务详情
**响应：**
```json
{
  "id": "task_abc123",
  "name": "奶茶_混剪_1430",
  "theme": "奶茶",
  "status": "done",
  "progress": 100,
  "log_text": "全部完成",
  "config": { ... },
  "subtitle_config": { ... },
  "materials": [
    {
      "id": 1,
      "title": "奶茶摇制过程特写",
      "cover_url": "...",
      "duration": 8.0,
      "type_tag": "food_action",
      "sort_order": 1
    },
    ...
  ],
  "outputs": [
    {
      "index": 1,
      "cover_url": "...",
      "duration": 15.0,
      "file_size": 8900000,
      "copy_text": "这杯奶茶绝了",
      "subtitle_segments": [ ... ],
      "edit_plan": [ ... ],
      "download_url": "..."
    },
    ...
  ],
  "created_at": "2025-01-15T14:30:00+08:00",
  "finished_at": "2025-01-15T14:32:15+08:00"
}
```
#### `POST /api/tasks/{id}/retry` — 重试失败任务
#### `DELETE /api/tasks/{id}` — 删除任务（级联删除产出文件）
#### `GET /api/tasks/{id}/outputs/batch-download` — 批量下载产出
返回打包 ZIP 的预签名 URL。
---
### 4.3 产出预览接口
#### `GET /api/tasks/{id}/outputs/{index}/preview` — 成片预览
**响应：**
```json
{
  "index": 1,
  "video_url": "https://minio.example.com/outputs/...?X-Amz-Signature=...",
  "cover_url": "...",
  "duration": 15.0,
  "copy_text": "这杯奶茶绝了",
  "subtitle_segments": [
    { "text": "这杯奶茶", "start": 0.0, "end": 3.0 },
    { "text": "绝了", "start": 3.0, "end": 6.0 }
  ],
  "subtitle_config": {
    "enabled": true,
    "position": "底部居中",
    "font": "思源黑体",
    "color": "白色",
    "stroke": true
  },
  "edit_plan": [
    { "material_id": 1, "strategy": "speedup", "source_start": 0.0, "source_end": 15.0, "output_duration": 3.0, "speed": 5.0 },
    { "material_id": 6, "strategy": "cut_highlight", "source_start": 2.1, "source_end": 7.5, "output_duration": 5.4, "speed": 1.0 }
  ]
}
```
前端根据 `subtitle_segments` 在播放器上叠加字幕层，`edit_plan` 可用于调试追溯。
#### `GET /api/tasks/{id}/outputs/{index}/download` — 单条下载
---
### 4.4 素材采集接口
#### `POST /api/collect` — 创建采集任务
**Request:**
```json
{
  "keywords": ["奶茶", "珍珠奶茶", "手打柠檬茶"],
  "platforms": ["抖音", "快手"],
  "count_per_keyword": 10
}
```
#### `GET /api/collect` — 采集任务列表
#### `GET /api/collect/{id}` — 采集任务详情（含进度）
**响应：**
```json
{
  "id": "collect_xyz",
  "status": "running",
  "progress": 65,
  "total_found": 42,
  "total_downloaded": 25,
  "total_deduplicated": 8,
  "results": [
    { "keyword": "奶茶", "platform": "抖音", "source_url": "...", "status": "success", "material_id": 25 },
    { "keyword": "奶茶", "platform": "抖音", "source_url": "...", "status": "skip_duplicate", "material_id": null },
    ...
  ]
}
```
#### `POST /api/collect/{id}/cancel` — 取消采集
---
### 4.5 辅助接口
#### `GET /api/themes` — 获取主题列表
#### `GET /api/stats/dashboard` — 工作台统计数据
#### `POST /api/fonts/upload` — 上传自定义字体
#### `GET /api/music/library` — 获取背景音乐库列表
#### `POST /api/materials/{id}/reanalyze` — 手动触发重新分析
对单个素材重新执行运动检测（用于分析失败或素材被替换的场景）。
---
## 五、视频处理核心流程
### 5.1 FFmpeg 剪辑流水线
单个产出视频的处理步骤：
```
┌─────────────────────────────────────────────────────┐
│ Step 0: 混剪决策（新增）                              │
│ MixDecisionEngine 根据素材属性 + 策略配置             │
│ 决定每段素材的处理方式（变速/截取高光/原速截取）       │
│ → 输出: edit_plan[]                                  │
├─────────────────────────────────────────────────────┤
│ Step 1: 素材策略化处理（升级）                         │
│ 根据 edit_plan 中每段的 strategy 字段执行：             │
│   - speedup: setpts + atempo 变速                    │
│   - cut_highlight: -ss 截取高光时段                   │
│   - cut_mid: -ss 截取中间段（降级方案）                │
│   - skip_cut: select 抽帧跳剪                        │
│ → 输出: segment_0.mp4, segment_1.mp4, ...           │
├─────────────────────────────────────────────────────┤
│ Step 2: 转场效果合成                                   │
│ xfade 滤镜实现溶解/滑动/缩放/模糊                     │
│ → 输出: merged_raw.mp4                               │
├─────────────────────────────────────────────────────┤
│ Step 3: 画面比例与分辨率调整                            │
│ scale + pad 滤镜 → 9:16/16:9/1:1                     │
│ → 输出: merged_scaled.mp4                            │
├─────────────────────────────────────────────────────┤
│ Step 4: TTS 配音生成（如启用）                         │
│ 阿里云语音合成 API → voiceover.mp3                   │
├─────────────────────────────────────────────────────┤
│ Step 5: 字幕压制（如启用）                             │
│ ASS 字幕文件 + subtitles 滤镜                         │
│ → 输出: with_subtitle.mp4                            │
├─────────────────────────────────────────────────────┤
│ Step 6: 音频混合                                      │
│ 配音 + 背景音乐（ducking 处理）                        │
│ amerge + sidechaincompress 滤镜                      │
│ → 输出: with_audio.mp4                               │
├─────────────────────────────────────────────────────┤
│ Step 7: 最终编码输出                                   │
│ libx264 + aac, CRF 23, preset medium                 │
│ → 输出: {theme}_广告_01_9x16_1080p.mp4               │
├─────────────────────────────────────────────────────┤
│ Step 8: 封面提取 + 元信息写入                           │
│ ffmpeg -ss 1 -vframes 1 → cover.jpg                 │
│ 更新 task_output 记录（含 edit_plan）                  │
└─────────────────────────────────────────────────────┘
```
### 5.2 混剪决策引擎（核心新增）
```python
from dataclasses import dataclass
from typing import List, Optional
import random
@dataclass
class ClipStrategy:
    type: str          # "speedup" / "cut_highlight" / "cut_mid" / "skip_cut"
    start: float       # 源素材截取起点
    end: float         # 源素材截取终点
    output_duration: float  # 输出时长
    speed: float = 1.0      # 变速倍率
class MixDecisionEngine:
    """根据素材情况和输出要求，智能决策每段素材的处理方式"""
    # 不同风格下的单段目标时长范围（秒）
    STYLE_CLIP_RANGE = {
        "fast":  (2.0, 4.0),
        "normal": (3.0, 6.0),
        "slow":  (5.0, 10.0),
    }
    # 禁止变速的素材类型
    NO_SPEEDUP_TYPES = {"people_talking"}
    # 变速安全系数：food_action 类可以大胆加速，其他需保守
    SPEEDUP_AGGRESSION = {
        "food_action": 1.0,    # 全速
        "ambience": 0.8,       # 较高速
        "scenery": 0.6,        # 中速
        "product_show": 0.3,   # 低速
        "other": 0.5,          # 默认
    }
    def plan_clips(
        self,
        materials: list,
        output_duration: float,
        mix_config: dict,
    ) -> List[ClipStrategy]:
        """
        输入：素材列表 + 输出时长 + 混剪策略配置
        输出：剪辑方案列表
        """
        style = mix_config.get("style", "normal")
        allow_speedup = mix_config.get("allow_speedup", True)
        max_speed = float(mix_config.get("max_speed", "3x").replace("x", ""))
        prefer_highlights = mix_config.get("prefer_highlights", True)
        clip_min, clip_max = self.STYLE_CLIP_RANGE[style]
        remaining = output_duration
        plan = []
        used_ids = set()
        attempts = 0
        max_attempts = len(materials) * 2  # 防止无限循环
        while remaining > 0.5 and attempts < max_attempts:
            attempts += 1
            # 1. 选素材：优先选未使用的、运动强度高的
            candidates = [m for m in materials if m["id"] not in used_ids]
            if not candidates:
                # 所有素材用完了，从头轮换
                used_ids.clear()
                candidates = materials
            # 按运动强度排序，高运动的排前面
            candidates.sort(
                key=lambda m: m.get("motion_score", 0),
                reverse=True
            )
            # 加一点随机性，避免每次结果完全一样
            top_n = max(1, len(candidates) // 2)
            mat = random.choice(candidates[:top_n])
            # 2. 决定这段目标时长
            if remaining <= clip_max:
                target_dur = remaining
            else:
                target_dur = random.uniform(clip_min, clip_max)
            # 3. 决定处理策略
            strategy = self._decide_strategy(
                mat=mat,
                target_duration=target_dur,
                allow_speedup=allow_speedup,
                max_speed=max_speed,
                prefer_highlights=prefer_highlights,
            )
            plan.append(strategy)
            remaining -= strategy.output_duration
            remaining = round(remaining, 2)
            used_ids.add(mat["id"])
        return plan
    def _decide_strategy(
        self,
        mat: dict,
        target_duration: float,
        allow_speedup: bool,
        max_speed: float,
        prefer_highlights: bool,
    ) -> ClipStrategy:
        """为单个素材决定处理策略"""
        mat_dur = mat["duration"]
        type_tag = mat.get("type_tag", "other")
        highlights = mat.get("highlight_segments", [])
        ratio = target_duration / mat_dur  # 目标/原始 = 需要压缩的比例
        # --- 策略 A：变速压缩 ---
        can_speedup = (
            allow_speedup
            and type_tag not in self.NO_SPEEDUP_TYPES
            and ratio < 1.0  # 需要压缩时才考虑变速
        )
        if can_speedup:
            aggression = self.SPEEDUP_AGGRESSION.get(type_tag, 0.5)
            needed_speed = 1.0 / ratio
            # 实际速度 = 需要速度 × 侵略系数（越激进越接近满速）
            actual_speed = 1.0 + (min(needed_speed, max_speed) - 1.0) * aggression
            actual_speed = max(1.0, min(actual_speed, max_speed))
            # 变速后实际能得到的时长
            actual_output = mat_dur / actual_speed
            if abs(actual_output - target_duration) < 1.0:
                # 变速后时长接近目标，直接用变速
                return ClipStrategy(
                    type="speedup",
                    start=0.0,
                    end=mat_dur,
                    output_duration=round(actual_output, 2),
                    speed=round(actual_speed, 2),
                )
        # --- 策略 B：截取高光片段 ---
        if prefer_highlights and highlights and ratio < 0.8:
            # 需要的远小于素材全长，从高光中截取
            best_seg = self._find_best_segment(highlights, target_duration)
            if best_seg:
                return ClipStrategy(
                    type="cut_highlight",
                    start=best_seg["start"],
                    end=best_seg["start"] + target_duration,
                    output_duration=target_duration,
                    speed=1.0,
                )
        # --- 策略 C：普通截取 ---
        if ratio <= 1.0:
            # 需要的 ≤ 素材全长，截取一段
            if prefer_highlights and highlights:
                # 优先从高光中截
                best_seg = self._find_best_segment(highlights, target_duration)
                if best_seg:
                    return ClipStrategy(
                        type="cut_highlight",
                        start=best_seg["start"],
                        end=best_seg["start"] + target_duration,
                        output_duration=target_duration,
                        speed=1.0,
                    )
            # 没有高光信息，从中间截取
            start = max(0, (mat_dur - target_duration) / 2)
            return ClipStrategy(
                type="cut_mid",
                start=round(start, 2),
                end=round(start + target_duration, 2),
                output_duration=target_duration,
                speed=1.0,
            )
        # --- 策略 D：需要比素材更长（极少见），原速全段使用 ---
        return ClipStrategy(
            type="cut_mid",
            start=0.0,
            end=mat_dur,
            output_duration=mat_dur,
            speed=1.0,
        )
    def _find_best_segment(
        self,
        highlights: list,
        target_duration: float,
    ) -> Optional[dict]:
        """从高光时段中找到最适合截取的段"""
        feasible = [
            h for h in highlights
            if (h["end"] - h["start"]) >= target_duration * 0.8
        ]
        if not feasible:
            return None
        # 选运动强度最高的段
        feasible.sort(key=lambda h: h["score"], reverse=True)
        best = feasible[0]
        # 在该段内居中截取
        seg_dur = best["end"] - best["start"]
        offset = (seg_dur - target_duration) / 2
        return {
            "start": round(best["start"] + max(0, offset), 2),
            "end": round(best["start"] + max(0, offset) + target_duration, 2),
        }
```
### 5.3 策略化素材处理（FFmpeg 实现）
```python
def apply_strategy(segment: ClipStrategy, material_path: str, output_path: str):
    """根据策略处理单段素材"""
    strategy = segment.type
    start = segment.start
    end = segment.end
    if strategy == "speedup":
        speed = segment.speed
        # 视频变速
        vf = f"setpts=PTS/{speed}"
        # 音频变速（atempo 范围 0.5-100，超过 2 倍需链式调用）
        af = _build_atempo_chain(speed)
        cmd = [
            "ffmpeg", "-y",
            "-ss", str(start), "-to", str(end),
            "-i", material_path,
            "-vf", vf, "-af", af,
            "-c:v", "libx264", "-preset", "ultrafast",
            "-c:a", "aac",
            output_path
        ]
    elif strategy in ("cut_highlight", "cut_mid"):
        # 原速截取
        cmd = [
            "ffmpeg", "-y",
            "-ss", str(start), "-t", str(segment.output_duration),
            "-i", material_path,
            "-c:v", "libx264", "-preset", "ultrafast",
            "-c:a", "aac",
            output_path
        ]
    elif strategy == "skip_cut":
        # 抽帧跳剪（每 N 帧取 1 帧）
        skip = max(2, int(segment.speed))
        vf = f"select='not(mod(n\\,{skip}))',setpts=N/FRAME_RATE/TB"
        cmd = [
            "ffmpeg", "-y",
            "-ss", str(start), "-to", str(end),
            "-i", material_path,
            "-vf", vf,
            "-c:v", "libx264", "-preset", "ultrafast",
            "-an",  # 跳剪通常去音频
            output_path
        ]
    else:
        raise ValueError(f"未知策略: {strategy}")
    subprocess.run(cmd, check=True, capture_output=True)
def _build_atempo_chain(speed: float) -> str:
    """
    构建 atempo 链式滤镜。
    atempo 单次范围 [0.5, 100.0]，超过 2 倍需链式：2 × 2.5 = 5x
    """
    if speed <= 2.0:
        return f"atempo={speed}"
    filters = []
    remaining = speed
    while remaining > 1.0:
        factor = min(remaining, 2.0)
        filters.append(f"atempo={factor}")
        remaining /= factor
    return ",".join(filters)
```
### 5.4 转场效果 FFmpeg 实现
```python
def dissolve_transition(seg1_path, seg2_path, offset, duration, output_path):
    cmd = [
        "ffmpeg", "-y",
        "-i", seg1_path, "-i", seg2_path,
        "-filter_complex",
        f"[0:v][1:v]xfade=transition=fade:duration={duration}:offset={offset}[v]",
        "-map", "[v]", output_path
    ]
    subprocess.run(cmd, check=True)
TRANSITION_MAP = {
    "溶解": "fade",
    "滑动": "slideleft",
    "缩放": "zoomin",
    "模糊": "blur",
}
```
### 5.5 字幕压制实现
采用 ASS 字幕文件方式（比 drawtext 更灵活，支持样式控制）：
```python
def generate_ass_file(segments, config, duration, output_path):
    """生成 ASS 字幕文件"""
    font_map = {"思源黑体": "Source Han Sans SC", "思源宋体": "Source Han Serif SC", "站酷快乐体": "ZCOOL KuaiLe"}
    font_name = font_map.get(config["font"], config["font"])
    size_map = {"小": 18, "中": 22, "大": 28}
    font_size = size_map.get(config["size"], 22)
    pos_map = {"底部居中": (8, 85), "底部左对齐": (3, 85), "顶部居中": (8, 10)}
    align_map = {"底部居中": 2, "底部左对齐": 1, "顶部居中": 8}
    pos = pos_map.get(config["position"], (8, 85))
    align = align_map.get(config["position"], 2)
    color_map = {"白色": "&H00FFFFFF", "黄色": "&H00FFE135", "青色": "&H0000F0FF"}
    primary_color = color_map.get(config["color"], "&H00FFFFFF")
    outline = 2 if config.get("stroke") else 0
    ass_content = f"""[Script Info]
Title: FlowCut Subtitle
ScriptType: v4.00+
PlayResX: 1080
PlayResY: 1920
WrapStyle: 0
[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour, BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing, Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Default,{font_name},{font_size},{primary_color},&H000000FF,&H00000000,&H80000000,0,0,0,0,100,100,0,0,1,{outline},1,{align},10,10,{pos[1]},1
[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
"""
    for seg in segments:
        start = seconds_to_ass_time(seg["start"])
        end = seconds_to_ass_time(seg["end"])
        ass_content += f"Dialogue: 0,{start},{end},Default,,0,0,0,,{seg['text']}\n"
    with open(output_path, "w", encoding="utf-8") as f:
        f.write(ass_content)
def seconds_to_ass_time(seconds):
    h = int(seconds // 3600)
    m = int((seconds % 3600) // 60)
    s = int(seconds % 60)
    cs = int((seconds % 1) * 100)
    return f"{h}:{m:02d}:{s:02d}.{cs:02d}"
```
### 5.6 音频混合（Ducking 处理）
当同时有配音和背景音乐时，需要在配音出现时自动压低背景音乐音量：
```python
def mix_audio_with_ducking(video_path, voice_path, music_path, voice_vol, music_vol, output_path):
    voice_db = (voice_vol - 100) / 2
    music_db = (music_vol - 100) / 2
    filter_complex = (
        f"[1:a]volume={voice_db}dB[voice];"
        f"[2:a]volume={music_db}dB,asplit=2[music_raw][music_sc];"
        f"[music_sc][voice]sidechaincompress=threshold=-20dB:ratio=4:attack=5:release=200[music_ducked];"
        f"[music_raw][voice]amix=inputs=2:duration=shortest[mixed]"
    )
    cmd = [
        "ffmpeg", "-y",
        "-i", video_path,
        "-i", voice_path,
        "-i", music_path,
        "-filter_complex", filter_complex,
        "-map", "0:v", "-map", "[mixed]",
        "-c:v", "copy", "-c:a", "aac",
        "-shortest", output_path
    ]
    subprocess.run(cmd, check=True)
```
### 5.7 TTS 语音合成
```python
import requests
ALIYUN_TTS_URL = "https://nls-gateway-cn-shanghai.aliyuncs.com/stream/v1/tts"
VOICE_STYLE_MAP = {
    "活力女声": "zhixiaoxia",
    "沉稳男声": "zhiyan_emo",
    "可爱女声": "zhiyu_emo",
    "磁性男声": "zhimi_emo",
}
SPEED_MAP = {"0.8x": 0.8, "1.0x": 1.0, "1.2x": 1.2, "1.5x": 1.5}
def generate_tts(text, style, speed, output_path):
    voice = VOICE_STYLE_MAP.get(style, "zhixiaoxia")
    rate = SPEED_MAP.get(speed, 1.0)
    payload = {
        "text": text,
        "voice": voice,
        "speech_rate": rate,
        "format": "mp3",
        "sample_rate": 16000,
    }
    resp = requests.post(ALIYUN_TTS_URL, json=payload, headers={
        "Authorization": f"Bearer {get_access_token()}"
    }, timeout=30)
    with open(output_path, "wb") as f:
        f.write(resp.content)
```
---
## 六、素材分析管线（核心新增）
### 6.1 运动检测算法
使用 OpenCV 帧差分法检测视频中的高光时段，纯 CPU 计算，无需 GPU 或 AI 模型：
```python
import cv2
import numpy as np
def detect_highlights(video_path: str, threshold: float = 25.0, min_segment: float = 1.5) -> dict:
    """
    检测视频中的高光时段（画面变化大的区间）
    返回:
    {
        "highlight_segments": [{"start": 3.2, "end": 8.5, "score": 45.3}, ...],
        "motion_score": 42.1,  # 整体运动强度（0-100）
        "type_tag": "food_action"  # 推断的素材类型
    }
    """
    cap = cv2.VideoCapture(video_path)
    if not cap.isOpened():
        raise ValueError(f"无法打开视频: {video_path}")
    fps = cap.get(cv2.CAP_PROP_FPS)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    duration = total_frames / fps
    # 缩小分辨率加速计算（160x90 足够检测运动）
    process_width = 160
    process_height = 90
    prev_gray = None
    frame_scores = []
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        gray = cv2.resize(gray, (process_width, process_height))
        if prev_gray is not None:
            diff = np.abs(gray.astype(np.float32) - prev_gray.astype(np.float32))
            score = float(np.mean(diff))
            frame_scores.append(score)
        prev_gray = gray
    cap.release()
    if not frame_scores:
        return {
            "highlight_segments": [],
            "motion_score": 0.0,
            "type_tag": "other",
        }
    frame_scores = np.array(frame_scores)
    # 1. 平滑处理：1秒窗口移动平均
    kernel_size = max(3, int(fps))
    if kernel_size % 2 == 0:
        kernel_size += 1
    smoothed = np.convolve(frame_scores, np.ones(kernel_size) / kernel_size, mode='same')
    # 2. 计算整体运动强度
    overall_score = float(np.percentile(smoothed, 75))  # 取75分位数，排除静止段
    # 3. 动态阈值：不用固定值，根据整体运动强度自适应
    adaptive_threshold = max(threshold, overall_score * 0.4)
    # 4. 找到高于阈值的连续区间
    segments = []
    in_segment = False
    start_idx = 0
    for i, score in enumerate(smoothed):
        t = i / fps
        if score > adaptive_threshold and not in_segment:
            in_segment = True
            start_idx = i
        elif score <= adaptive_threshold and in_segment:
            in_segment = False
            seg_start = start_idx / fps
            seg_end = t
            if seg_end - seg_start >= min_segment:
                avg_score = float(np.mean(smoothed[start_idx:i]))
                segments.append({
                    "start": round(seg_start, 2),
                    "end": round(seg_end, 2),
                    "score": round(avg_score, 1),
                })
    if in_segment:
        seg_start = start_idx / fps
        seg_end = len(smoothed) / fps
        if seg_end - seg_start >= min_segment:
            avg_score = float(np.mean(smoothed[start_idx:]))
            segments.append({
                "start": round(seg_start, 2),
                "end": round(seg_end, 2),
                "score": round(avg_score, 1),
            })
    # 5. 如果没有检测到高光段，返回整个视频作为一个段
    if not segments:
        segments.append({
            "start": 0.0,
            "end": round(duration, 2),
            "score": round(overall_score, 1),
        })
    # 6. 按运动强度排序
    segments.sort(key=lambda s: s["score"], reverse=True)
    # 7. 推断素材类型
    type_tag = infer_type_tag(overall_score, segments, duration)
    return {
        "highlight_segments": segments,
        "motion_score": round(overall_score, 1),
        "type_tag": type_tag,
    }
def infer_type_tag(overall_score: float, segments: list, duration: float) -> str:
    """根据运动特征推断素材类型（启发式规则，后续可替换为 VLM）"""
    # 高运动强度 + 多个短高光段 = 动作类
    if overall_score > 30 and len(segments) >= 2:
        avg_seg_dur = sum(s["end"] - s["start"] for s in segments) / len(segments)
        if avg_seg_dur < duration * 0.5:
            return "food_action"
    # 低运动强度 + 少量长段 = 静态/展示类
    if overall_score < 15:
        return "scenery"
    # 中等运动 + 长段 = 可能是人物互动
    if overall_score >= 15 and len(segments) <= 2:
        avg_seg_dur = sum(s["end"] - s["start"] for s in segments) / len(segments)
        if avg_seg_dur > duration * 0.5:
            return "people_talking"
    # 中等运动 + 短段 = 产品展示
    if 15 <= overall_score <= 35:
        return "product_show"
    # 默认
    return "other"
```
### 6.2 异步分析任务
运动检测放在 Celery 异步任务中，不阻塞上传响应：
```python
@celery.task(bind=True, queue="material_process", max_retries=2)
def analyze_material_task(self, material_id: int):
    """异步分析素材：运动检测 + 类型推断 + 高光标注"""
    from app.database import get_sync_db
    from app.utils.minio import download_to_tmp
    db = get_sync_db()
    material = db.query("SELECT * FROM material WHERE id = %s", (material_id,))[0]
    try:
        # 1. 更新状态为"分析中"
        db.execute(
            "UPDATE material SET analysis_status = 'analyzing' WHERE id = %s",
            (material_id,)
        )
        # 2. 下载到临时文件
        tmp_path = download_to_tmp(material["file_key"])
        # 3. 执行运动检测
        result = detect_highlights(tmp_path)
        # 4. 写入数据库
        db.execute(
            """UPDATE material
               SET highlight_segments = %s,
                   motion_score = %s,
                   type_tag = %s,
                   analysis_status = 'done'
               WHERE id = %s""",
            (
                json.dumps(result["highlight_segments"]),
                result["motion_score"],
                result["type_tag"],
                material_id,
            )
        )
    except Exception as e:
        logger.error(f"素材分析失败 {material_id}: {e}")
        db.execute(
            "UPDATE material SET analysis_status = 'failed' WHERE id = %s",
            (material_id,)
        )
        raise self.retry(exc=e, countdown=60)
    finally:
        # 清理临时文件
        if os.path.exists(tmp_path):
            os.remove(tmp_path)
```
### 6.3 入库管线更新
```
原管线：上传 → ffprobe → MD5去重 → 生成封面 → 入库 → 结束
新管线：上传 → ffprobe → MD5去重 → 生成封面 → 入库(analysis_status=pending)
                                                              ↓
                                                    异步触发 analyze_material_task
                                                              ↓
                                                    运动检测 → 更新 highlight_segments / type_tag / motion_score
                                                              ↓
                                                    analysis_status = 'done'
```
### 6.4 降级策略
当素材尚未完成分析（`analysis_status != 'done'`）时，混剪引擎自动降级：
```python
def get_material_for_decision(material: dict) -> dict:
    """获取用于混剪决策的素材数据，未分析时提供降级默认值"""
    if material.get("analysis_status") == "done":
        return material
    # 降级：使用默认值
    dur = material["duration"]
    return {
        **material,
        "type_tag": "other",
        "motion_score": 20.0,  # 中等默认值
        "highlight_segments": [
            {"start": 0.0, "end": dur, "score": 20.0}  # 整段作为"高光"
        ],
    }
```
---
## 七、Celery 任务设计
### 7.1 任务队列配置
```python
# celeryconfig.py
broker_url = "redis://redis:6379/0"
result_backend = "redis://redis:6379/1"
task_queues = {
    "material_process": {
        "exchange": "material_process",
        "routing_key": "material_process",
    },
    "collect": {
        "exchange": "collect",
        "routing_key": "collect",
    },
    "video_render": {
        "exchange": "video_render",
        "routing_key": "video_render",
    },
}
# Worker 并发控制
# material_process 队列: 并发 4（OpenCV 不太吃CPU，但吃内存）
# collect 队列: 并发 2（避免被平台封）
# video_render 队列: 并发 = CPU 核心数（FFmpeg CPU 密集）
```
### 7.2 剪辑主任务
```python
@celery.task(bind=True, queue="video_render", max_retries=1)
def render_task(self, task_id):
    """
    剪辑主任务，负责调度多个产出视频的生成
    """
    task = get_task_from_db(task_id)
    task.status = "processing"
    task.started_at = datetime.now()
    save_task(task)
    materials = get_task_materials(task_id)
    # 为决策引擎准备素材数据（含降级处理）
    materials_for_decision = [get_material_for_decision(m) for m in materials]
    target_duration = task.output_duration
    output_count = task.output_count
    config = task.config
    mix_config = config.get("mix_strategy", {})
    # 使用混剪决策引擎生成剪辑方案
    engine = MixDecisionEngine()
    copy_texts = get_copy_texts(task.theme, output_count)
    total = output_count
    for i in range(total):
        try:
            update_task_progress(task_id, 10 + int(80 * i / total), f"正在规划第 {i+1}/{total} 条视频...")
            # 每条视频独立决策（随机性保证批量产出不重复）
            edit_plan = engine.plan_clips(
                materials=materials_for_decision,
                output_duration=target_duration,
                mix_config=mix_config,
            )
            update_task_progress(task_id, 10 + int(80 * i / total) + 2, f"正在渲染第 {i+1}/{total} 条视频...")
            output = render_single_video(task_id, i + 1, edit_plan, copy_texts[i % len(copy_texts)], materials, config)
            save_task_output(task_id, i + 1, output)
        except Exception as e:
            logger.error(f"渲染第 {i+1} 条失败: {e}", exc_info=True)
    task.status = "done"
    task.progress = 100
    task.log_text = "全部完成"
    task.finished_at = datetime.now()
    save_task(task)
```
### 7.3 采集任务
```python
@celery.task(bind=True, queue="collect")
def collect_task(self, collect_id):
    """采集任务，按关键词 × 平台逐个下载"""
    collect = get_collect_task(collect_id)
    collect.status = "running"
    save_collect(collect)
    keywords = collect.keywords
    platforms = collect.platforms
    count = collect.count_per_keyword
    total_jobs = len(keywords) * len(platforms)
    completed = 0
    for keyword in keywords:
        for platform in platforms:
            try:
                results = download_by_keyword(keyword, platform, count)
                for r in results:
                    save_collect_result(collect_id, keyword, platform, r)
                    collect.total_downloaded += (1 if r["status"] == "success" else 0)
                    collect.total_deduplicated += (1 if r["status"] == "skip_duplicate" else 0)
            except Exception as e:
                logger.error(f"采集失败 {keyword}@{platform}: {e}")
            completed += 1
            collect.progress = int(100 * completed / total_jobs)
            save_collect(collect)
    collect.status = "done"
    save_collect(collect)
```
### 7.4 进度上报机制
```python
def update_task_progress(task_id, progress, log_text):
    """更新任务进度，同时推送到前端（通过 Redis Pub/Sub）"""
    db.execute(
        "UPDATE task SET progress = %s, log_text = %s WHERE id = %s",
        (progress, log_text, task_id)
    )
    redis_client.publish(f"task_progress:{task_id}", json.dumps({
        "task_id": task_id,
        "progress": progress,
        "log_text": log_text,
        "timestamp": datetime.now().isoformat(),
    }))
```
### 7.5 前端实时进度（SSE）
```python
@app.get("/api/tasks/{task_id}/progress-stream")
async def task_progress_stream(task_id: str):
    async def event_generator():
        pubsub = redis_client.pubsub()
        await pubsub.subscribe(f"task_progress:{task_id}")
        try:
            async for message in pubsub.listen():
                if message["type"] == "message":
                    yield f"data: {message['data']}\n\n"
        finally:
            await pubsub.unsubscribe(f"task_progress:{task_id}")
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"}
    )
```
---
## 八、采集脚本设计
### 8.1 技术方案
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  yt-dlp      │────▶│  下载视频文件  │────▶│  上传 MinIO  │
│  视频下载     │     │  到临时目录    │     │  + 提取元信息 │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                   │
                                            ┌──────▼───────┐
                                            │  MD5 去重检查  │
                                            │  写入 DB      │
                                            │  (status=     │
                                            │   pending)    │
                                            └──────┬───────┘
                                                   │
                                            ┌──────▼───────┐
                                            │  异步触发     │
                                            │  运动检测     │
                                            └──────────────┘
```
### 8.2 各平台适配
| 平台 | 下载方式 | 关键参数 |
|------|----------|----------|
| 抖音 | yt-dlp + `--extractor-args` | 需设置 cookie 避免登录限制 |
| 快手 | yt-dlp | 直接支持 |
| B站 | yt-dlp | 直接支持，可选画质 |
| 小红书 | Playwright 模拟 | 需要浏览器环境提取真实视频 URL |
### 8.3 去重流程
```python
def download_and_dedup(url, keyword, platform):
    """下载视频并去重"""
    tmp_path = download_with_ytdlp(url)
    md5 = calculate_md5_fast(tmp_path)
    existing = db.query("SELECT id FROM material WHERE md5_hash = %s", (md5,))
    if existing:
        os.remove(tmp_path)
        return {"status": "skip_duplicate", "material_id": existing[0]["id"]}
    probe = ffprobe(tmp_path)
    file_key = f"materials/{platform}/{datetime.now():%Y/%m/%d}/{md5}.mp4"
    minio_client.fput_object("flowcut-materials", file_key, tmp_path)
    cover_path = extract_cover(tmp_path)
    cover_key = f"covers/materials/{md5}.jpg"
    minio_client.fput_object("flowcut-covers", cover_key, cover_path)
    material_id = insert_material(
        title=f"{keyword}_{platform}_{md5[:8]}",
        file_key=file_key, md5_hash=md5,
        duration=probe["duration"], width=probe["width"], height=probe["height"],
        fps=probe["fps"], source=platform, source_url=url,
        cover_key=cover_key, tags=[keyword],
        analysis_status="pending",  # 新增：标记为待分析
    )
    os.remove(tmp_path)
    os.remove(cover_path)
    # 异步触发运动检测
    analyze_material_task.delay(material_id)
    return {"status": "success", "material_id": material_id}
```
---
## 九、前端技术方案
### 9.1 项目结构
```
src/
├── api/                    # 接口层
│   ├── materials.ts        # 素材库接口
│   ├── tasks.ts            # 任务接口
│   ├── collect.ts          # 采集接口
│   └── request.ts          # Axios 封装
├── components/             # 通用组件
│   ├── MaterialCard.vue    # 素材卡片（含类型标签、高光条指示）
│   ├── TaskCard.vue        # 任务卡片
│   ├── VideoPlayer.vue     # 视频播放器（含字幕叠加）
│   ├── StepWizard.vue      # 步骤引导组件
│   ├── ConfigPanel.vue     # 配置面板
│   ├── MixStrategyPanel.vue # 混剪策略配置面板（新增）
│   ├── SubtitleOverlay.vue # 字幕叠加层
│   └── HighlightBar.vue    # 高光时段可视化条（新增）
├── views/                  # 页面
│   ├── Dashboard.vue       # 工作台
│   ├── Materials.vue       # 素材库
│   ├── Workflow.vue        # 剪辑工作流
│   ├── Tasks.vue           # 任务管理
│   └── Collect.vue         # 素材采集
├── stores/                 # Pinia 状态
│   ├── materials.ts
│   ├── workflow.ts
│   └── tasks.ts
├── types/                  # TypeScript 类型
│   └── index.ts
├── utils/                  # 工具函数
│   ├── format.ts           # 格式化
│   └── subtitle.ts         # 字幕时间计算
├── App.vue
└── main.ts
```
### 9.2 混剪策略配置面板
工作流 Step3 中新增的配置区域：
```vue
<!-- MixStrategyPanel.vue -->
<template>
  <div class="config-group space-y-3">
    <h3 class="text-sm font-medium text-txt flex items-center gap-2">
      <i class="fas fa-scissors text-primary text-xs"></i>混剪策略
    </h3>
    <div class="space-y-2">
      <label class="text-xs text-txt-dim">混剪风格</label>
      <div class="grid grid-cols-3 gap-2">
        <button v-for="s in styles" :key="s.value"
          :class="['px-3 py-2 rounded-lg border text-sm text-center transition-colors',
            modelValue === s.value ? 'border-primary bg-primary-soft text-primary' : 'border-border text-txt-dim']"
          @click="$emit('update:modelValue', s.value)">
          <div class="font-medium">{{ s.label }}</div>
          <div class="text-[10px] mt-0.5 opacity-70">{{ s.desc }}</div>
        </button>
      </div>
    </div>
    <div class="flex items-center justify-between">
      <div>
        <span class="text-sm text-txt">允许变速</span>
        <span class="text-xs text-txt-muted ml-1">（吃东西/动作类加速，说话类保持原速）</span>
      </div>
      <div class="switch" :class="{ on: allowSpeedup }" @click="$emit('update:allowSpeedup', !allowSpeedup)"></div>
    </div>
    <div v-if="allowSpeedup" class="space-y-2">
      <label class="text-xs text-txt-dim">最大变速倍率</label>
      <div class="grid grid-cols-3 gap-2">
        <button v-for="s in ['2x', '3x', '5x']" :key="s"
          :class="['px-3 py-1.5 rounded-lg border text-xs text-center transition-colors',
            maxSpeed === s ? 'border-primary bg-primary-soft text-primary' : 'border-border text-txt-dim']"
          @click="$emit('update:maxSpeed', s)">
          {{ s }}
          <div class="text-[10px] opacity-60">{{ { '2x': '轻微加速', '3x': '适中', '5x': '快闪' }[s] }}</div>
        </button>
      </div>
    </div>
    <div class="flex items-center justify-between">
      <span class="text-sm text-txt">优先使用高光片段</span>
      <div class="switch" :class="{ on: preferHighlights }" @click="$emit('update:preferHighlights', !preferHighlights)"></div>
    </div>
  </div>
</template>
<script setup>
const styles = [
  { value: 'fast', label: '快节奏', desc: '2-4秒/段，适合抖音' },
  { value: 'normal', label: '常规', desc: '3-6秒/段，通用' },
  { value: 'slow', label: '慢节奏', desc: '5-10秒/段，适合小红书' },
];
defineProps({
  modelValue: { type: String, default: 'normal' },
  allowSpeedup: { type: Boolean, default: true },
  maxSpeed: { type: String, default: '3x' },
  preferHighlights: { type: Boolean, default: true },
});
defineEmits(['update:modelValue', 'update:allowSpeedup', 'update:maxSpeed', 'update:preferHighlights']);
</script>
```
### 9.3 高光时段可视化
素材卡片和预览弹窗中可展示高光条（可选，不阻塞主流程）：
```vue
<!-- HighlightBar.vue -->
<template>
  <div class="relative h-1 bg-surface rounded-full overflow-hidden">
    <!-- 灰色底条表示完整时长 -->
    <!-- 高亮段用主色填充 -->
    <div v-for="(seg, i) in segments" :key="i"
      class="absolute top-0 h-full bg-primary/60 rounded-full"
      :style="{
        left: (seg.start / totalDuration * 100) + '%',
        width: ((seg.end - seg.start) / totalDuration * 100) + '%',
      }">
    </div>
  </div>
</template>
```
### 9.4 工作流状态管理更新
```typescript
// stores/workflow.ts
export const useWorkflowStore = defineStore('workflow', () => {
  // ... 其他状态同原方案
  const config = reactive({
    voice: { enabled: true, style: '活力女声', speed: '1.0', volume: 80 },
    subtitle: { enabled: true, position: '底部居中', font: '思源黑体', size: '中', color: '白色', stroke: true },
    music: { enabled: true, style: '轻快', volume: 30 },
    transition: { type: '溶解', duration: '0.5s' },
    output: { count: 5, duration: '15s', ratio: '9:16', resolution: '1080p' },
    // 新增：混剪策略
    mix_strategy: {
      style: 'normal',
      allow_speedup: true,
      max_speed: '3x',
      prefer_highlights: true,
    },
  });
  // ... 其他方法同原方案
});
```
### 9.5 任务实时进度
同原方案，见 v1.0 第八章 8.4 节。
---
## 十、部署方案
### 10.1 Docker Compose 编排
```yaml
version: '3.8'
services:
  nginx:
    image: nginx:1.24-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./frontend/dist:/usr/share/nginx/html
    depends_on: [backend]
  backend:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
    environment:
      - DATABASE_URL=postgresql+asyncpg://flowcut:xxx@postgres:5432/flowcut
      - REDIS_URL=redis://redis:6379/0
      - MINIO_ENDPOINT=minio:9000
      - MINIO_ACCESS_KEY=minioadmin
      - MINIO_SECRET_KEY=minioadmin
      - ALIYUN_TTS_ACCESS_KEY=${ALIYUN_TTS_AK}
      - ALIYUN_TTS_ACCESS_SECRET=${ALIYUN_TTS_SK}
    depends_on: [postgres, redis, minio]
  celery-worker-video:
    build: ./backend
    command: celery -A app.celery_app worker -Q video_render -c 4 --loglevel=info
    environment:
      <<: *backend-env
    volumes:
      - /tmp/flowcut-tmp:/tmp/flowcut-tmp
    depends_on: [backend]
  celery-worker-collect:
    build: ./backend
    command: celery -A app.celery_app worker -Q collect -c 2 --loglevel=info
    environment:
      <<: *backend-env
    depends_on: [backend]
  celery-worker-analysis:
    build: ./backend
    command: celery -A app.celery_app worker -Q material_process -c 4 --loglevel=info
    environment:
      <<: *backend-env
    depends_on: [backend]
  postgres:
    image: pgvector/pgvector:pg15
    environment:
      POSTGRES_DB: flowcut
      POSTGRES_USER: flowcut
      POSTGRES_PASSWORD: xxx
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9001:9001"]
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - miniodata:/data
volumes:
  pgdata:
  redisdata:
  miniodata:
```
> 注意：PostgreSQL 镜像改为 `pgvector/pgvector:pg15`，以支持向量存储（为后续 Agent 化预留，当前版本不强制使用向量功能）。
### 10.2 服务器配置建议
| 角色 | 配置 | 数量 | 说明 |
|------|------|------|------|
| Web + API | 4C 8G | 1 | Nginx + FastAPI |
| 视频渲染 Worker | 8C 16G | 1-2 | FFmpeg CPU 密集 |
| 素材分析 Worker | 4C 8G | 1 | OpenCV 运动检测，可与采集 Worker 合并 |
| 采集 Worker | 2C 4G | 1 | IO 密集 |
| PostgreSQL | 4C 8G SSD | 1 | 可与 Web 合部署 |
| Redis | 2C 4G | 1 | 可与 Web 合部署 |
| MinIO | 4C 8G + 2TB 磁盘 | 1 | 纯存储节点 |
**最低起步配置**：单台 16C 32G 服务器，所有服务合部署，可支撑日均 100 条视频产出。
### 10.3 FFmpeg 性能优化
```bash
# 编译 FFmpeg 时启用硬件加速（如服务器有 NVIDIA GPU）
./configure --enable-gpl --enable-libx264 --enable-libx265 \
  --enable-nvenc --enable-cuda --enable-cuvid
# Celery Worker 启动时设置线程亲和性
taskset -c 0-3 celery -A app.celery_app worker -Q video_render -c 4
```
### 10.4 Nginx 配置要点
```nginx
server {
    listen 80;
    server_name flowcut.example.com;
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # SSE 长连接支持
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600s;
    }
    client_max_body_size 500M;
}
```
---
## 十一、安全方案
| 风险点 | 措施 |
|--------|------|
| 文件上传恶意内容 | 限制后缀白名单+ FFprobe 校验是否为合法视频 |
| MinIO 文件泄露 | 全部私有 Bucket，通过预签名 URL 访问，URL 有效期限制 |
| 接口未授权访问 | JWT Token 认证，前端 localStorage 存储 |
| 采集脚本被封 | 单平台并发限制为 2、请求间隔随机延迟 2-5s、Cookie 轮换 |
| FFmpeg 命令注入 | 所有参数通过 subprocess 的列表参数传递，不拼接 shell 字符串 |
| 大文件上传 OOM | 流式上传到 MinIO，不经过后端内存；FFmpeg 处理使用临时文件而非内存 |
| 运动检测恶意输入 | OpenCV 处理前校验文件头，限制最大处理时长（10分钟），超时强制终止 |
---
## 十二、监控与告警
### 12.1 关键指标
| 指标 | 采集方式 | 告警阈值 |
|------|----------|----------|
| 任务队列积压 | Redis `LLEN` | > 20 |
| 单条视频渲染耗时 | Celery 任务耗时 | > 120s |
| 任务失败率 | DB 统计 | > 10%（1小时内） |
| MinIO 存储用量 | MinIO API | > 1.8TB |
| 采集成功率 | DB 统计 | < 60% |
| API 平均响应时间 | Nginx 日志 | > 1s |
| Worker 进程存活 | Celery 心跳 | 进程退出 |
| 素材分析积压 | DB `analysis_status='pending'` 计数 | > 100 |
| 素材分析失败率 | DB `analysis_status='failed'` 计数 | > 5% |
### 12.2 日志规范
```python
import structlog
logger = structlog.get_logger()
# 任务关键节点日志
logger.info("task.video.render.start", task_id=task_id, output_index=i, segment_count=len(edit_plan))
logger.info("task.video.render.strategy", task_id=task_id, output_index=i, segment_index=j, strategy=seg.type, speed=seg.speed)
logger.info("task.video.render.done", task_id=task_id, output_index=i, file_size=file_size, total_time=elapsed)
logger.error("task.video.render.failed", task_id=task_id, output_index=i, error=str(e), exc_info=True)
# 素材分析日志
logger.info("material.analysis.start", material_id=material_id, duration=duration)
logger.info("material.analysis.done", material_id=material_id, type_tag=type_tag, highlight_count=len(segments), motion_score=score, elapsed=elapsed)
logger.error("material.analysis.failed", material_id=material_id, error=str(e), exc_info=True)
```
---
## 十三、性能预估
### 13.1 单条视频渲染耗时估算
| 步骤 | 耗时（1080p, 15s） | 耗时（720p, 15s） | 备注 |
|------|---------------------|---------------------|------|
| 混剪决策 | < 0.01s | < 0.01s | 纯计算，可忽略 |
| 策略化片段处理（含变速） | 2-4s | 1-2s | 比原方案略增（变速需重编码） |
| 转场合成 | 2-3s | 1-2s | 不变 |
| 比例调整 | 1-2s | 0.5-1s | 不变 |
| TTS 配音 | 1-2s | 1-2s | 不变 |
| 字幕压制 | 2-3s | 1-2s | 不变 |
| 音频混合 | 1-2s | 0.5-1s | 不变 |
| 最终编码 | 5-8s | 2-4s | 不变 |
| 封面提取 | 0.5s | 0.5s | 不变 |
| **合计** | **15-25s** | **8-14s** | 比原方案增加约 1-3s |
### 13.2 素材分析耗时估算
| 素材时长 | 分析耗时 | 备注 |
|---------|---------|------|
| 10 秒 | 1-2s | 帧数少 |
| 30 秒 | 2-4s | 最常见 |
| 60 秒 | 3-6s | 帧数多但算法是 O(n) |
| 300 秒 | 10-15s | 长视频，缩放到 160x90 后仍可控 |
### 13.3 吞吐量预估
| 配置 | 日产出能力 |
|------|-----------|
| 1 Worker（4C），15s/条 | ~2800 条/天 |
| 2 Worker（8C），15s/条 | ~5600 条/天 |
| 1 Worker（4C），10s/条 | ~4200 条/天 |
| 素材分析（4C） | ~8000 条/天 |
满足 PRD 中日均 50-100 条的需求，有 28-56 倍余量。
---
## 十四、开发排期
| 阶段 | 内容 | 前端 | 后端 | 联调 | 状态 |
|------|------|------|------|------|------|
| P0 | 素材库（上传/浏览/搜索/筛选/下载/预览） | 3天 | 3天 | 1天 | 🔲 |
| P0.5 | **素材分析管线（运动检测/高光标注/类型推断）** | — | **2天** | — | 🔲 **新增** |
| P1 | 素材采集（脚本+入库+进度，入库后自动触发分析） | 2天 | 3天 | 1天 | 🔲 |
| P2 | 工作流主题选择+素材筛选 | 2天 | — | — | 🔲 |
| P3 | 工作流参数配置（**含混剪策略面板**） | **2.5天** | — | — | 🔲 |
| P4 | FFmpeg 剪辑核心（**策略化处理+变速+转场**） | — | **5天** | 1天 | 🔲 |
| P5 | TTS 配音 + 字幕压制(ASS) + 音频混合 | — | 2天 | 1天 | 🔲 |
| P6 | **混剪决策引擎 + 批量产出 + 任务管理** | 2天 | **3天** | 1天 | 🔲 |
| P7 | SSE 实时进度推送 | — | 1天 | 1天 | 🔲 |
| P8 | 成片预览播放器（**含剪辑方案追溯展示**） | 2天 | 1天 | 1天 | 🔲 |
| P9 | 工作台（数据概览+快捷入口） | 1天 | 1天 | — | 🔲 |
| P10 | 部署 + 集成测试 + 修复 | 1天 | 2天 | — | 🔲 |
| | **合计** | **15.5天** | **23天** | **7天** | |
> **总工期约 4 周**（前后端并行开发，联调穿插进行）。相比 v1.0 方案，后端增加约 5 天（运动检测 2天 + 策略化处理 1天 + 决策引擎 1天 + 降级逻辑 1天），前端增加 0.5 天（混剪策略面板）。
---
## 十五、v1.0 → v2.0 变更摘要
| 章节 | 变更内容 |
|------|----------|
| 架构图 | Worker 层新增「运动检测引擎」和独立 `material_process` 队列 |
| 技术栈 | 新增 OpenCV 4.8+；PostgreSQL 改为 pgvector 镜像（预留向量能力） |
| 数据库 | material 表新增 `type_tag` / `highlight_segments` / `motion_score` / `analysis_status` 四个字段；task_output 表新增 `edit_plan` 字段；新增 `idx_material_type_tag` 等索引 |
| API | 素材列表新增 `type_tag` / `motion_score` / `highlight_segments` / `analysis_status` 返回字段及筛选参数；创建任务接口 config 中 `random_strategy` 替换为 `mix_strategy`；新增 `GET /api/materials/type-tags` 和 `POST /api/materials/{id}/reanalyze` 接口 |
| 视频处理 | 新增 Step 0「混剪决策」；Step 1 从简单截取升级为「策略化处理」（speedup/cut_highlight/cut_mid/skip_cut 四种策略） |
| **素材分析** | **全新章节**：运动检测算法（OpenCV 帧差分）、自适应阈值、类型推断启发式规则、异步分析任务、降级策略 |
| Celery | 新增 `material_process` 队列及 `analyze_material_task`；渲染主任务中 `generate_random_combinations` 替换为 `MixDecisionEngine.plan_clips` |
| 采集脚本 | 入库后自动触发异步分析 |
| 前端 | 新增 `MixStrategyPanel.vue`、`HighlightBar.vue` 组件；工作流 store 中 config 新增 `mix_strategy`；产出预览增加 `edit_plan` 展示 |
| 部署 | Docker Compose 新增 `celery-worker-analysis` 服务；PostgreSQL 镜像切换为 pgvector |
| 监控 | 新增素材分析积压和失败率指标 |
| 性能 | 单条渲染耗时增加 1-3s（变速重编码开销）；新增素材分析耗时估算 |
| 排期 | 后端 +5 天，前端 +0.5 天，总工期从 3.5 周增至 4 周 |

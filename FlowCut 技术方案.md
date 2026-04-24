# FlowCut 技术方案
**文档版本**：v1.0  
**编写日期**：2025-01-15  
**基于文档**：FlowCut PRD v1.0  
**目标读者**：后端开发、前端开发、DevOps、测试开发  
---
## 一、系统架构
### 1.1 整体架构
采用前后端分离 + 异步任务队列的经典架构，核心流程为：
```
浏览器（Vue 3） → FastAPI 后端 → PostgreSQL（元数据）
                              → MinIO（视频文件）
                              → Redis（缓存 + 消息队列）
                                    → Celery Worker（视频处理）
                                         → FFmpeg（剪辑引擎）
                                         → TTS 服务（配音生成）
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
└────────────────────────┴────┼─────────────────────┘         │
                             │                               │
              ┌──────────────┼──────────────┐                │
              ▼              ▼              ▼                │
        ┌──────────┐  ┌──────────┐  ┌──────────────┐        │
        │PostgreSQL│  │  MinIO   │  │    Redis     │        │
        │ 元数据    │  │ 视频文件  │  │ 缓存/队列    │        │
        └──────────┘  └──────────┘  └──────┬───────┘        │
                                          │                 │
                              ┌───────────▼───────────┐      │
                              │    Celery Workers     │      │
                              │  ┌───────┐ ┌───────┐  │      │
                              │  │Worker1│ │Worker2│  │      │
                              │  └───┬───┘ └───┬───┘  │      │
                              │      │         │      │      │
                              │  ┌───▼─────────▼───┐  │      │
                              │  │   FFmpeg CLI    │  │      │
                              │  └────────────────┘  │      │
                              │  ┌────────────────┐  │      │
                              │  │  TTS Service   │  │      │
                              │  └────────────────┘  │      │
                              └──────────────────────┘      │
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
| 数据库 | PostgreSQL | 15+ | 结构化数据持久化 |
| 对象存储 | MinIO | 最新版 | 视频文件存储 |
| 缓存/队列 | Redis | 7.0+ | 缓存 + Celery Broker |
| 任务队列 | Celery | 5.3+ | 异步任务调度 |
| 视频处理 | FFmpeg | 6.0+ | 核心剪辑引擎 |
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
| uploaded_by | VARCHAR(50) | | 上传者 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 入库时间 |
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
  "random_strategy": "random_segments"
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
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 生成时间 |
subtitle_segments 字段 JSONB 结构：
```json
[
  { "text": "这杯奶茶", "start": 0.0, "end": 2.5 },
  { "text": "绝了", "start": 2.5, "end": 5.0 }
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
### 3.2 对象键命名规则
```
素材视频:    materials/{source}/{year}/{month}/{day}/{md5_hash}.mp4
素材封面:    covers/materials/{md5_hash}.jpg
产出视频:    outputs/{task_id}/{index:03d}_{theme}_广告_{ratio}_{resolution}.mp4
产出封面:    covers/outputs/{task_id}/{index:03d}.jpg
字体文件:    fonts/{font_name}.{ext}
背景音乐:    music/{style}/{filename}.mp3
```
### 3.3 预签名 URL 策略
前端不直接访问 MinIO，所有文件下载/预览通过后端生成预签名 URL：
```python
# 预签名 URL 有效期
COVER_URL_EXPIRE = 3600      # 封面图 1 小时
MATERIAL_URL_EXPIRE = 7200   # 素材预览 2 小时
OUTPUT_URL_EXPIRE = 86400    # 产出下载 24 小时
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
| duration_min | float | 否 | 最小时长（秒） |
| duration_max | float | 否 | 最大时长（秒） |
| resolution | string | 否 | 分辨率过滤，如 `1080p` |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页数量，默认 20，最大 50 |
| sort_by | string | 否 | 排序字段：`created_at` / `duration` / `file_size` |
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
后端自动执行：ffprobe 提取元信息 → 计算 MD5 去重 → 生成封面 → 入库。
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
    "random_strategy": "random_segments"
  }
}
```
**响应：**
```json
{
  "task_id": "task_abc123",
  "name": "奶茶_混剪_1430",
  "status": "queue",
  "estimated_time": 120
}
```
后端校验逻辑：
- 素材数量 ≥ 3
- 素材总时长 ≥ 输出时长
- 输出数量 1-50
- 分辨率与比例兼容性校验
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
    { "id": 1, "title": "奶茶摇制过程特写", "cover_url": "...", "duration": 8.0, "sort_order": 1 },
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
  }
}
```
前端根据 `subtitle_segments` 在播放器上叠加字幕层，不需要后端额外处理。
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
---
## 五、视频处理核心流程
### 5.1 FFmpeg 剪辑流水线
单个产出视频的处理步骤：
```
┌─────────────────────────────────────────────────┐
│ Step 1: 素材片段抽取与裁剪                        │
│   ffmpeg -ss {start} -t {dur} -i {input} ...    │
│   → 输出: segment_0.mp4, segment_1.mp4, ...     │
├─────────────────────────────────────────────────┤
│ Step 2: 转场效果合成                               │
│   xfade 滤镜实现溶解/滑动/缩放/模糊               │
│   → 输出: merged_raw.mp4                         │
├─────────────────────────────────────────────────┤
│ Step 3: 画面比例与分辨率调整                        │
│   scale + pad 滤镜 → 9:16/16:9/1:1              │
│   → 输出: merged_scaled.mp4                      │
├─────────────────────────────────────────────────┤
│ Step 4: TTS 配音生成（如启用）                      │
│   阿里云语音合成 API → voiceover.mp3             │
├─────────────────────────────────────────────────┤
│ Step 5: 字幕压制（如启用）                          │
│   drawtext 滤镜或 ASS 字幕文件 + subtitles 滤镜    │
│   → 输出: with_subtitle.mp4                      │
├─────────────────────────────────────────────────┤
│ Step 6: 音频混合                                  │
│   配音 + 背景音乐（ducking 处理）                  │
│   amerge + sidechaincompress 滤镜                │
│   → 输出: with_audio.mp4                         │
├─────────────────────────────────────────────────┤
│ Step 7: 最终编码输出                                │
│   libx264 + aac, CRF 23, preset medium          │
│   → 输出: {theme}_广告_01_9x16_1080p.mp4        │
├─────────────────────────────────────────────────┤
│ Step 8: 封面提取 + 元信息写入                       │
│   ffmpeg -ss 1 -vframes 1 → cover.jpg           │
│   更新 task_output 记录                           │
└─────────────────────────────────────────────────┘
```
### 5.2 转场效果 FFmpeg 实现
```python
# 溶解转场（xfade）
def dissolve_transition(seg1_path, seg2_path, offset, duration, output_path):
    cmd = [
        "ffmpeg", "-y",
        "-i", seg1_path, "-i", seg2_path,
        "-filter_complex",
        f"[0:v][1:v]xfade=transition=fade:duration={duration}:offset={offset}[v]",
        "-map", "[v]", output_path
    ]
    subprocess.run(cmd, check=True)
# 支持的转场类型映射
TRANSITION_MAP = {
    "溶解": "fade",
    "滑动": "slideleft",
    "缩放": "zoomin",
    "模糊": "blur",
}
```
### 5.3 字幕压制实现
采用 ASS 字幕文件方式（比 drawtext 更灵活，支持样式控制）：
```python
def generate_ass_file(segments, config, duration, output_path):
    """生成 ASS 字幕文件"""
    font_map = {"思源黑体": "Source Han Sans SC", "思源宋体": "Source Han Serif SC", "站酷快乐体": "ZCOOL KuaiLe"}
    font_name = font_map.get(config["font"], config["font"])
    
    size_map = {"小": 18, "中": 22, "大": 28}
    font_size = size_map.get(config["size"], 22)
    
    # ASS 位置映射
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
### 5.4 音频混合（Ducking 处理）
当同时有配音和背景音乐时，需要在配音出现时自动压低背景音乐音量：
```python
def mix_audio_with_ducking(video_path, voice_path, music_path, voice_vol, music_vol, output_path):
    """
    使用 sidechaincompress 实现 ducking：
    配音音量高于阈值时，背景音乐自动降低音量
    """
    # 将音量从 0-100 映射到 dB
    voice_db = (voice_vol - 100) / 2  # 例: 80 → -10dB
    music_db = (music_vol - 100) / 2  # 例: 30 → -35dB
    
    filter_complex = (
        f"[1:a]volume={voice_db}dB[voice];"
        f"[2:a]volume={music_db}dB,asplit=2[music_raw][music_sc];"
        f"[music_sc][voice]sidechaincompress=threshold=-20dB:ratio=4:attack=5:release=200[music_ducked];"
        f"[music_raw][voice]amix=inputs=2:duration=shortest[mixed]"
    )
    
    cmd = [
        "ffmpeg", "-y",
        "-i", video_path,        # 视频轨（可能已有原声）
        "-i", voice_path,        # 配音
        "-i", music_path,        # 背景音乐
        "-filter_complex", filter_complex,
        "-map", "0:v", "-map", "[mixed]",
        "-c:v", "copy", "-c:a", "aac",
        "-shortest", output_path
    ]
    subprocess.run(cmd, check=True)
```
### 5.5 随机片段组合策略
```python
import random
def generate_random_combinations(materials, output_count, target_duration):
    """
    从素材池中随机抽取片段，组合成 target_duration 长度的视频
    返回: List[List[Segment]]
    """
    combinations = []
    total_available = sum(m["duration"] for m in materials)
    
    for _ in range(output_count):
        segments = []
        remaining = target_duration
        shuffled = random.sample(materials, len(materials))
        
        while remaining > 0.5 and shuffled:
            mat = shuffled[random.randint(0, len(shuffled) - 1)]
            use_dur = min(mat["duration"], remaining)
            start = random.uniform(0, max(0, mat["duration"] - use_dur))
            segments.append({
                "material_id": mat["id"],
                "file_path": mat["file_path"],
                "start": round(start, 2),
                "duration": round(use_dur, 2),
            })
            remaining -= use_dur
        
        combinations.append(segments)
    
    return combinations
```
### 5.6 TTS 语音合成
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
## 六、Celery 任务设计
### 6.1 任务队列配置
```python
# celeryconfig.py
broker_url = "redis://redis:6379/0"
result_backend = "redis://redis:6379/1"
task_queues = {
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
# collect 队列: 并发 2（避免被平台封）
# video_render 队列: 并发 = CPU 核心数（FFmpeg CPU 密集）
```
### 6.2 剪辑主任务
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
    target_duration = task.output_duration
    output_count = task.output_count
    config = task.config
    
    # 生成随机组合
    combinations = generate_random_combinations(
        materials, output_count, target_duration
    )
    
    # 生成广告文案
    copy_texts = get_copy_texts(task.theme, output_count)
    
    total = len(combinations)
    for i, (segments, copy_text) in enumerate(zip(combinations, copy_texts)):
        try:
            update_task_progress(task_id, 10 + int(80 * i / total), f"正在生成第 {i+1}/{total} 条视频...")
            output = render_single_video(task_id, i + 1, segments, copy_text, config)
            save_task_output(task_id, i + 1, output)
        except Exception as e:
            logger.error(f"渲染第 {i+1} 条失败: {e}")
            # 单条失败不影响整体，记录但继续
    
    task.status = "done"
    task.progress = 100
    task.log_text = "全部完成"
    task.finished_at = datetime.now()
    save_task(task)
```
### 6.3 采集任务
```python
@celery.task(bind=True, queue="collect")
def collect_task(self, collect_id):
    """
    采集任务，按关键词 × 平台逐个下载
    """
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
### 6.4 进度上报机制
```python
def update_task_progress(task_id, progress, log_text):
    """更新任务进度，同时推送到前端（通过 Redis Pub/Sub）"""
    # 1. 更新数据库
    db.execute(
        "UPDATE task SET progress = %s, log_text = %s WHERE id = %s",
        (progress, log_text, task_id)
    )
    
    # 2. 发布到 Redis 频道（前端 SSE 订阅）
    redis_client.publish(f"task_progress:{task_id}", json.dumps({
        "task_id": task_id,
        "progress": progress,
        "log_text": log_text,
        "timestamp": datetime.now().isoformat(),
    }))
```
### 6.5 前端实时进度（SSE）
```python
# FastAPI SSE 端点
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
前端使用 `EventSource` 订阅：
```javascript
const es = new EventSource(`/api/tasks/${taskId}/progress-stream`);
es.onmessage = (e) => {
  const data = JSON.parse(e.data);
  updateProgressBar(data.progress);
  updateLogText(data.log_text);
};
```
---
## 七、采集脚本设计
### 7.1 技术方案
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  yt-dlp      │────▶│  下载视频文件  │────▶│  上传 MinIO  │
│  视频下载     │     │  到临时目录    │     │  + 提取元信息 │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                   │
                                            ┌──────▼───────┐
                                            │  MD5 去重检查  │
                                            │  写入 DB      │
                                            └──────────────┘
```
### 7.2 各平台适配
| 平台 | 下载方式 | 关键参数 |
|------|----------|----------|
| 抖音 | yt-dlp + `--extractor-args` | 需设置 cookie 避免登录限制 |
| 快手 | yt-dlp | 直接支持 |
| B站 | yt-dlp | 直接支持，可选画质 |
| 小红书 | Playwright 模拟 | 需要浏览器环境提取真实视频 URL |
### 7.3 去重流程
```python
def download_and_dedup(url, keyword, platform):
    """下载视频并去重"""
    # 1. yt-dlp 下载到临时文件
    tmp_path = download_with_ytdlp(url)
    
    # 2. 计算 MD5（取前 10MB + 尾部 1MB 加速大文件计算）
    md5 = calculate_md5_fast(tmp_path)
    
    # 3. 查重
    existing = db.query("SELECT id FROM material WHERE md5_hash = %s", (md5,))
    if existing:
        os.remove(tmp_path)
        return {"status": "skip_duplicate", "material_id": existing[0]["id"]}
    
    # 4. ffprobe 提取元信息
    probe = ffprobe(tmp_path)
    
    # 5. 上传 MinIO
    file_key = f"materials/{platform}/{datetime.now():%Y/%m/%d}/{md5}.mp4"
    minio_client.fput_object("flowcut-materials", file_key, tmp_path)
    
    # 6. 生成封面
    cover_path = extract_cover(tmp_path)
    cover_key = f"covers/materials/{md5}.jpg"
    minio_client.fput_object("flowcut-covers", cover_key, cover_path)
    
    # 7. 入库
    material_id = insert_material(
        title=f"{keyword}_{platform}_{md5[:8]}",
        file_key=file_key, md5_hash=md5,
        duration=probe["duration"], width=probe["width"], height=probe["height"],
        fps=probe["fps"], source=platform, source_url=url,
        cover_key=cover_key, tags=[keyword]
    )
    
    # 8. 清理临时文件
    os.remove(tmp_path)
    os.remove(cover_path)
    
    return {"status": "success", "material_id": material_id}
```
---
## 八、前端技术方案
### 8.1 项目结构
```
src/
├── api/                    # 接口层
│   ├── materials.ts        # 素材库接口
│   ├── tasks.ts            # 任务接口
│   ├── collect.ts          # 采集接口
│   └── request.ts          # Axios 封装
├── components/             # 通用组件
│   ├── MaterialCard.vue    # 素材卡片
│   ├── TaskCard.vue        # 任务卡片
│   ├── VideoPlayer.vue     # 视频播放器（含字幕叠加）
│   ├── StepWizard.vue      # 步骤引导组件
│   ├── ConfigPanel.vue     # 配置面板
│   └── SubtitleOverlay.vue # 字幕叠加层
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
### 8.2 视频播放器组件设计
`VideoPlayer.vue` 核心功能：
```
┌───────────────────────────────────┐
│  视频画面层（<video> 或 <img>）     │
│  ┌─────────────────────────────┐  │
│  │                             │  │
│  │   字幕叠加层                 │  │
│  │   (absolute 定位，根据       │  │
│  │    subtitle_segments 计算     │  │
│  │    当前应显示哪段文案)         │  │
│  │                             │  │
│  └─────────────────────────────┘  │
│  大播放按钮（居中，点击播放/暂停）  │
│  ┌─────────────────────────────┐  │
│  │ ▶ ━━━━━●━━━━━━━━ 01:23/15  │  │ ← 底部控制条（hover 显现）
│  │   ⏮  ⏭  文件名             │  │
│  └─────────────────────────────┘  │
└───────────────────────────────────┘
```
字幕叠加逻辑：
```typescript
// subtitle.ts
export function getCurrentSubtitle(
  segments: SubtitleSegment[],
  currentTime: number
): SubtitleSegment | null {
  return segments.find(
    s => currentTime >= s.start && currentTime < s.end
  ) || null;
}
// VideoPlayer.vue 中使用
const subtitle = computed(() => 
  getCurrentSubtitle(props.segments, currentTime.value)
);
```
### 8.3 工作流状态管理
```typescript
// stores/workflow.ts
export const useWorkflowStore = defineStore('workflow', () => {
  const step = ref(1);
  const theme = ref<string | null>(null);
  const selectedMaterials = ref<Material[]>([]);
  const config = reactive({
    voice: { enabled: true, style: '活力女声', speed: '1.0', volume: 80 },
    subtitle: { enabled: true, position: '底部居中', font: '思源黑体', size: '中', color: '白色', stroke: true },
    music: { enabled: true, style: '轻快', volume: 30 },
    transition: { type: '溶解', duration: '0.5s' },
    output: { count: 5, duration: '15s', ratio: '9:16', resolution: '1080p' },
  });
  function reset() {
    step.value = 1;
    theme.value = null;
    selectedMaterials.value = [];
    // config 重置为默认值...
  }
  async function submit() {
    const resp = await api.createTask({
      theme: theme.value,
      material_ids: selectedMaterials.value.map(m => m.id),
      material_order: selectedMaterials.value.map(m => m.id),
      config: toRaw(config),
    });
    reset();
    return resp;
  }
  return { step, theme, selectedMaterials, config, reset, submit };
});
```
### 8.4 任务实时进度
```typescript
// composables/useTaskProgress.ts
export function useTaskProgress(taskId: Ref<string>) {
  const progress = ref(0);
  const logText = ref('');
  let eventSource: EventSource | null = null;
  function start() {
    eventSource = new EventSource(`/api/tasks/${taskId.value}/progress-stream`);
    eventSource.onmessage = (e) => {
      const data = JSON.parse(e.data);
      progress.value = data.progress;
      logText.value = data.log_text;
    };
    eventSource.onerror = () => stop();
  }
  function stop() {
    eventSource?.close();
    eventSource = null;
  }
  onUnmounted(stop);
  return { progress, logText, start, stop };
}
```
---
## 九、部署方案
### 9.1 Docker Compose 编排
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
      - /tmp/flowcut-tmp:/tmp/flowcut-tmp  # 视频处理临时目录
    depends_on: [backend]
  celery-worker-collect:
    build: ./backend
    command: celery -A app.celery_app worker -Q collect -c 2 --loglevel=info
    environment:
      <<: *backend-env
    depends_on: [backend]
  postgres:
    image: postgres:15-alpine
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
    ports: ["9001:9001"]  # MinIO 管理控制台
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
### 9.2 服务器配置建议
| 角色 | 配置 | 数量 | 说明 |
|------|------|------|------|
| Web + API | 4C 8G | 1 | Nginx + FastAPI |
| 视频处理 Worker | 8C 16G | 1-2 | FFmpeg CPU 密集，需高性能 CPU |
| 采集 Worker | 2C 4G | 1 | IO 密集，配置要求低 |
| PostgreSQL | 4C 8G SSD | 1 | 可与 Web 合部署 |
| Redis | 2C 4G | 1 | 可与 Web 合部署 |
| MinIO | 4C 8G + 2TB 磁盘 | 1 | 纯存储节点，磁盘按需扩展 |
**最低起步配置**：单台 16C 32G 服务器，所有服务合部署，可支撑日均 100 条视频产出。
### 9.3 FFmpeg 性能优化
```bash
# 编译 FFmpeg 时启用硬件加速（如服务器有 NVIDIA GPU）
./configure --enable-gpl --enable-libx264 --enable-libx265 \
  --enable-nvenc --enable-cuda --enable-cuvid
# Celery Worker 启动时设置线程亲和性
taskset -c 0-3 celery -A app.celery_app worker -Q video_render -c 4
```
### 9.4 Nginx 配置要点
```nginx
server {
    listen 80;
    server_name flowcut.example.com;
    # 前端静态资源
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }
    # API 反向代理
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # SSE 长连接支持
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600s;
    }
    
    # 上传大小限制（单素材最大 500MB）
    client_max_body_size 500M;
}
```
---
## 十、安全方案
| 风险点 | 措施 |
|--------|------|
| 文件上传恶意内容 | 限制后缀白名单（mp4/webm/mov/jpg/png）、FFprobe 校验是否为合法视频 |
| MinIO 文件泄露 | 全部私有 Bucket，通过预签名 URL 访问，URL 有效期限制 |
| 接口未授权访问 | JWT Token 认证，前端 localStorage 存储 |
| 采集脚本被封 | 单平台并发限制为 2、请求间隔随机延迟 2-5s、Cookie 轮换 |
| FFmpeg 命令注入 | 所有参数通过 subprocess 的列表参数传递，不拼接 shell 字符串 |
| 大文件上传 OOM | 流式上传到 MinIO，不经过后端内存；FFmpeg 处理使用临时文件而非内存 |
---
## 十一、监控与告警
### 11.1 关键指标
| 指标 | 采集方式 | 告警阈值 |
|------|----------|----------|
| 任务队列积压 | Redis `LLEN` | > 20 |
| 单条视频渲染耗时 | Celery 任务耗时 | > 120s |
| 任务失败率 | DB 统计 | > 10%（1小时内） |
| MinIO 存储用量 | MinIO API | > 1.8TB |
| 采集成功率 | DB 统计 | < 60% |
| API 平均响应时间 | Nginx 日志 | > 1s |
| Worker 进程存活 | Celery 心跳 | 进程退出 |
### 11.2 日志规范
```python
import structlog
logger = structlog.get_logger()
# 任务关键节点日志
logger.info("task.video.render.start", task_id=task_id, output_index=i, segment_count=len(segments))
logger.info("task.video.render.segment_done", task_id=task_id, output_index=i, segment_index=j, duration=elapsed)
logger.info("task.video.render.done", task_id=task_id, output_index=i, file_size=file_size, total_time=elapsed)
logger.error("task.video.render.failed", task_id=task_id, output_index=i, error=str(e), exc_info=True)
```
---
## 十二、性能预估
### 12.1 单条视频渲染耗时估算
| 步骤 | 耗时（1080p, 15s） | 耗时（720p, 15s） |
|------|---------------------|---------------------|
| 素材片段抽取 | 1-2s | 0.5-1s |
| 转场合成 | 2-3s | 1-2s |
| 比例调整 | 1-2s | 0.5-1s |
| TTS 配音 | 1-2s（网络IO） | 1-2s |
| 字幕压制 | 2-3s | 1-2s |
| 音频混合 | 1-2s | 0.5-1s |
| 最终编码 | 5-8s | 2-4s |
| 封面提取 | 0.5s | 0.5s |
| **合计** | **14-22s** | **7-13s** |
### 12.2 吞吐量预估
| 配置 | 日产出能力 |
|------|-----------|
| 1 Worker（4C），15s/条 | ~3000 条/天 |
| 2 Worker（8C），15s/条 | ~6000 条/天 |
| 1 Worker（4C），10s/条 | ~4500 条/天 |
满足 PRD 中日均 50-100 条的需求，有 30-60 倍余量。
---
## 十三、开发排期（修订版）
| 阶段 | 内容 | 前端 | 后端 | 联调 |
|------|------|------|------|------|
| P0 | 素材库（上传/浏览/搜索/下载/预览） | 3天 | 3天 | 1天 |
| P1 | 素材采集（脚本+入库+进度） | 2天 | 3天 | 1天 |
| P2 | 工作流主题选择+素材筛选 | 2天 | - | - |
| P3 | 工作流参数配置 | 2天 | - | - |
| P4 | FFmpeg 剪辑核心（拼接+转场+字幕） | - | 4天 | 1天 |
| P5 | TTS 配音 + 音频混合 | - | 2天 | 1天 |
| P6 | 批量产出 + 随机组合 | - | 2天 | - |
| P7 | 任务管理 + SSE 进度 | 2天 | 2天 | 1天 |
| P8 | 成片预览播放器 | 2天 | 1天 | 1天 |
| P9 | 部署 + 测试 + 修复 | 1天 | 2天 | - |
| **合计** | | **14天** | **18天** | **7天** |
**总工期约 3.5 周**（前后端并行开发，联调穿插进行）。

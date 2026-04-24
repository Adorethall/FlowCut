# FlowCut 素材采集与自动化信息流广告视频剪辑工作流平台

## 简介

FlowCut 是为广告设计团队打造的一站式工具，将「找素材 → 选素材 → 剪视频 → 看效果」的链路从人工串联变为系统驱动。核心解决三个问题：

- **素材获取慢** — 脚本自动从抖音/快手/B站/小红书批量采集，MD5 去重后入库，团队共享
- **素材利用盲目** — 入库后自动执行 OpenCV 运动检测，提取高光时段、推断素材类型（动作类/说话类/展示类等），混剪时精准截取精彩画面
- **剪辑重复劳动** — 四步配置（选主题 → 选素材 → 设参数 → 提交），后端基于混剪决策引擎智能选择变速/截取高光/原速截取策略，FFmpeg 自动混剪、压字幕、配音、配乐，批量产出
- **审核效率低** — 内置成片预览播放器，字幕按配置还原、列表快速切换、自动连播、剪辑方案追溯，不用逐个下载到本地

## 功能概览

### 素材采集

- 配置关键词 × 平台 × 数量，一键启动采集
- yt-dlp + Playwright 多平台适配
- MD5 快速去重，ffprobe 自动提取元信息
- 实时进度条展示采集状态
- **入库后自动触发异步高光标注**（OpenCV 运动检测 + 素材类型推断），不阻塞采集流程

### 素材库

- 关键词模糊搜索（标题/标签，300ms 防抖）
- 来源平台多选 + 标签快捷筛选
- **素材类型筛选**（食物制作/人物互动/风景静态/产品展示/氛围场景）
- 网格卡片浏览（竖版封面、时长、来源标识）
- **素材类型标签**：卡片左下角显示智能分类结果，待分析素材显示「待分析」
- **高光指示条**：卡片封面底部用主色标记检测到的高光时段，直观展示素材精彩部分
- **运动评分**：已分析素材显示整体运动强度评分
- 在线预览（含**高光时段可视化条**，悬浮可查看时间与评分）、单条/批量下载
- 勾选素材直接跳转工作流
- 分析失败素材支持手动触发重新分析

### 剪辑工作流

四步引导式配置：

1. **选择主题** — 8 个预设主题 + 自定义，显示素材总量与已分析数量
2. **筛选编排** — 左右分栏，按主题自动匹配，列表展示素材类型标签，手动调整排序
3. **参数配置**
   - 配音（风格/语速/音量）
   - 字幕（位置/字体/颜色/描边）
   - 配乐（风格/音量）
   - 转场（类型/时长）
   - **混剪策略（新增）** — 混剪风格（快节奏/常规/慢节奏）、允许变速开关、最大变速倍率（2x/3x/5x）、优先使用高光片段开关。系统自动识别「人物说话类」素材并强制原速，未分析素材自动降级为普通截取
   - 输出（数量/时长/比例/分辨率）
4. **确认提交** — 配置摘要（含混剪策略）+ 素材预览，一键提交

### 任务管理

- 状态标签页筛选（全部/排队中/处理中/已完成/失败）
- SSE 实时进度推送，阶段文字同步更新（含「正在智能混剪视频片段...」等）
- 任务卡片显示混剪风格标签
- 失败任务一键重试
- 批量打包下载

### 成片预览

- 模拟播放器：进度条可跳转、播放/暂停、时间显示
- 字幕叠加层：根据任务配置还原位置/字体/颜色/描边，按时间轴淡入淡出
- 产出列表侧栏：缩略卡片、点击切换、当前项高亮、自动滚动
- 上一条/下一条快捷切换 + 自动连播
- **剪辑方案追溯（新增）**：侧栏底部展示当前视频的剪辑方案时间轴，标注每段素材的处理策略（5x变速 / 截取高光 / 原速截取），不同策略用颜色区分，切换视频时动态更新

## 技术栈

| 层级 | 技术 |
|------|------|
| 前端 | Vue 3 + Element Plus + Pinia + Axios |
| 后端 | Python FastAPI + SQLAlchemy 2.0 |
| 数据库 | PostgreSQL 15 (pgvector) |
| 文件存储 | MinIO（私有 Bucket + 预签名 URL） |
| 缓存/队列 | Redis 7 |
| 异步任务 | Celery 5（采集队列 + 渲染队列 + **素材分析队列**） |
| 视频处理 | FFmpeg 6（xfade 转场 / ASS 字幕 / sidechaincompress 音频混合 / **setpts+atempo 变速**） |
| **画面分析** | **OpenCV 4.8+（帧差分运动检测 / 高光时段提取 / 素材类型启发式推断）** |
| **混剪决策** | **MixDecisionEngine（策略选择：speedup / cut_highlight / cut_mid / skip_cut）** |
| 语音合成 | 阿里云智能语音 TTS |
| 采集 | yt-dlp + Playwright |
| 实时通信 | SSE (Server-Sent Events) |
| 部署 | Docker Compose + Nginx |

## 项目结构
```
flowcut/
├── frontend/                  # Vue 3 前端
│   ├── src/
│   │   ├── api/               # Axios 接口封装
│   │   ├── components/        # 通用组件
│   │   │   ├── VideoPlayer.vue        # 视频播放器（含字幕叠加）
│   │   │   ├── MaterialCard.vue       # 素材卡片（含类型标签、高光条）
│   │   │   ├── StepWizard.vue         # 步骤引导组件
│   │   │   ├── ConfigPanel.vue        # 配置面板
│   │   │   ├── MixStrategyPanel.vue   # 混剪策略配置面板
│   │   │   ├── SubtitleOverlay.vue    # 字幕叠加层
│   │   │   └── HighlightBar.vue       # 高光时段可视化条
│   │   ├── views/             # 页面
│   │   │   ├── Dashboard.vue          # 工作台（含待分析素材统计）
│   │   │   ├── Materials.vue          # 素材库（含类型筛选、高光指示）
│   │   │   ├── Workflow.vue           # 剪辑工作流（含混剪策略 Step）
│   │   │   ├── Tasks.vue              # 任务管理
│   │   │   └── Collect.vue            # 素材采集
│   │   ├── stores/            # Pinia 状态管理
│   │   │   ├── materials.ts
│   │   │   ├── workflow.ts            # 含 mix_strategy 状态
│   │   │   └── tasks.ts
│   │   ├── types/             # TypeScript 类型定义
│   │   ├── utils/             # 工具函数
│   │   ├── App.vue
│   │   └── main.ts
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
├── backend/                   # FastAPI 后端
│   ├── app/
│   │   ├── main.py            # 应用入口
│   │   ├── config.py          # 配置管理
│   │   ├── database.py        # 数据库连接
│   │   ├── models/            # SQLAlchemy 模型
│   │   │   ├── material.py            # 含 type_tag / highlight_segments / motion_score / analysis_status
│   │   │   ├── task.py                # 含 config.mix_strategy
│   │   │   └── task_output.py         # 含 edit_plan
│   │   ├── schemas/           # Pydantic 请求/响应模型
│   │   ├── api/               # 路由
│   │   ├── services/          # 业务逻辑
│   │   │   ├── mix_engine.py          # 混剪决策引擎 MixDecisionEngine
│   │   │   └── highlight_detector.py  # OpenCV 运动检测 + 类型推断
│   │   ├── workers/           # Celery 任务
│   │   │   ├── render.py              # 渲染任务（含策略化处理）
│   │   │   ├── collect.py             # 采集任务
│   │   │   └── analyze.py             # 素材分析任务（新增）
│   │   └── utils/             # FFmpeg 封装, TTS 封装, MinIO 封装
│   ├── celery_app.py          # Celery 实例配置（含 material_process 队列）
│   ├── alembic/               # 数据库迁移
│   ├── requirements.txt
│   └── Dockerfile
├── nginx/
│   └── conf.d/
│       └── default.conf
├── docker-compose.yml         # 服务编排（含 celery-worker-analysis）
├── .env.example               # 环境变量模板
├── docs/
│   ├── PRD.md                 # 产品需求文档（v2.1）
│   └── TECHNICAL.md           # 技术方案（v2.0）
└── README.md
```
## 快速开始
### 环境要求
- Docker + Docker Compose
- Node.js 18+（前端开发）
- Python 3.11+（后端开发）
- FFmpeg 6+（视频处理 Worker 机器需安装）
- **OpenCV 4.8+**（素材分析 Worker 机器需安装，`pip install opencv-python-headless`）
### 一、配置环境变量
```bash
cp .env.example .env
```
编辑 `.env`，填入以下关键配置：
```env
# 数据库
POSTGRES_PASSWORD=your_strong_password
# MinIO
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=your_minio_password
# 阿里云 TTS（配音功能需要）
ALIYUN_TTS_ACCESS_KEY=your_ak
ALIYUN_TTS_ACCESS_SECRET=your_sk
# JWT 密钥
JWT_SECRET_KEY=your_random_secret
```
### 二、启动所有服务
```bash
docker compose up -d
```
首次启动会自动：
- 创建 PostgreSQL 数据库并执行迁移
- 创建 MinIO Bucket（`flowcut-materials`, `flowcut-covers`, `flowcut-outputs` 等）
- 上传预设字体文件和背景音乐到对应 Bucket
- 启动 **素材分析 Worker**（`celery-worker-analysis`），自动处理新入库素材
### 三、初始化数据
```bash
# 创建超级管理员账号
docker compose exec backend python -m app.scripts.create_admin --username admin --password admin123
# 导入预设背景音乐（轻快/抒情/动感/古风/电子各若干首）
docker compose exec backend python -m app.scripts.import_music /data/music/
# 导入预设字体（思源黑体/思源宋体/站酷快乐体）
docker compose exec backend python -m app.scripts.import_fonts /data/fonts/
```
### 四、访问
| 服务 | 地址 |
|------|------|
| 前端应用 | http://localhost |
| MinIO 管理控制台 | http://localhost:9001 |
| API 文档 | http://localhost/api/docs |
默认账号：`admin` / `admin123`
### 本地开发模式
如果不需要 Docker，可以分别启动前后端：
```bash
# 后端
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
# 前端
cd frontend
npm install
npm run dev
# 视频渲染 Worker（新终端）
cd backend
celery -A app.celery_app worker -Q video_render -c 4 --loglevel=info
# 素材分析 Worker（新终端）
cd backend
celery -A app.celery_app worker -Q material_process -c 4 --loglevel=info
# 采集 Worker（新终端）
cd backend
celery -A app.celery_app worker -Q collect -c 2 --loglevel=info
# Redis（需要本地安装）
redis-server
```
## 部署
### 生产环境建议配置
| 角色 | 最低配置 | 说明 |
|------|----------|------|
| Web + API | 4C 8G | Nginx + FastAPI |
| 视频渲染 Worker | 8C 16G | FFmpeg CPU 密集，建议独立部署 |
| **素材分析 Worker** | **4C 8G** | **OpenCV 运动检测，可与采集 Worker 合并部署** |
| 采集 Worker | 2C 4G | IO 密集，配置要求低 |
| PostgreSQL | 4C 8G SSD | 可与 Web 合部署 |
| Redis | 2C 4G | 可与 Web 合部署 |
| MinIO | 4C 8G + 按需磁盘 | 纯存储节点 |
> **最低起步配置**：单台 16C 32G 服务器，所有服务合部署，可支撑日均 100 条视频产出。
### FFmpeg 性能优化
如 Worker 机器有 NVIDIA GPU，编译 FFmpeg 时启用硬件加速：
```bash
./configure --enable-gpl --enable-libx264 --enable-libx265 \
  --enable-nvenc --enable-cuda --enable-cuvid
```
Celery Worker 启动时绑定 CPU 核心：
```bash
taskset -c 0-3 celery -A app.celery_app worker -Q video_render -c 4 --loglevel=info
```
### Nginx 关键配置
```nginx
# SSE 长连接支持（任务进度推送）
location /api/tasks/ {
  proxy_pass http://backend:8000;
  proxy_buffering off;
  proxy_cache off;
  proxy_read_timeout 3600s;
}
# 上传大小限制
client_max_body_size 500M;
```
### 素材分析安全限制
- OpenCV 处理前校验文件头，防止恶意输入
- 限制最大处理时长为 10 分钟，超时强制终止进程
- 分析任务独立队列，不阻塞采集和渲染
## 性能参考
| 指标 | 数值 |
|------|------|
| 单条视频渲染（1080p, 15s） | 15-25 秒（含变速重编码开销） |
| 单条视频渲染（720p, 15s） | 8-14 秒 |
| 素材分析耗时（10s 视频） | 1-2 秒 |
| 素材分析耗时（30s 视频） | 2-4 秒 |
| 素材分析耗时（60s 视频） | 3-6 秒 |
| 日产出吞吐（单 Worker, 4C） | ~2800 条 |
| 素材分析吞吐（单 Worker, 4C） | ~8000 条/天 |
| 搜索响应 | < 500ms |
| 任务进度推送延迟 | < 500ms |
## 文档
| 文档 | 说明 |
|------|------|
| [PRD.md](docs/PRD.md) | 产品需求文档（v2.1，含智能混剪策略与高光标注） |
| [TECHNICAL.md](docs/TECHNICAL.md) | 技术方案 v2.0（架构设计/数据库/API/混剪决策引擎/运动检测/部署方案） |
## 界面预览
> 以下为原型截图，实际界面以最终实现为准
**工作台** — 数据概览（含待分析素材统计）+ 快速操作
**素材库** — 搜索筛选（含素材类型筛选）+ 网格浏览（含类型标签与高光指示条）
**剪辑工作流** — 四步引导式配置（含混剪策略配置面板）
**任务管理** — 实时进度 + 状态筛选（含混剪风格标签）
**成片预览** — 播放器 + 字幕叠加 + 列表切换 + 剪辑方案追溯
## 常见问题
**Q: 采集脚本会被平台封吗？**
A: 已做限流保护——单平台并发上限 2，请求间隔随机 2-5 秒，Cookie 轮换。建议每个关键词下载 10-20 条，避免大量高频采集。
**Q: 字幕样式在预览时能完全还原吗？**
A: 能。字幕配置（位置/字体/颜色/描边）在提交任务时保存到 `subtitle_config` 字段，预览时据此用 CSS 还原。最终产出使用 ASS 字幕文件 + FFmpeg subtitles 滤镜，效果一致。
**Q: 混剪策略中的「允许变速」会把人物说话变声吗？**
A: 不会。系统会自动识别 `people_talking`（人物互动）类型的素材，即使开启变速也会对该类素材强制原速处理，避免变声问题。
**Q: 素材还没分析完就提交了剪辑任务怎么办？**
A: 不影响任务执行。未完成分析的素材会自动降级为「中间截取」策略（取素材中间段），任务正常产出。分析完成后创建的新任务可享受智能策略。
**Q: 如何添加自定义背景音乐和字体？**
A: 上传文件到 MinIO 对应 Bucket（`flowcut-music/{风格}/` 和 `flowcut-fonts/`），后端会自动扫描注册。二期将支持管理界面上传。
**Q: 视频渲染失败怎么办？**
A: 单条失败不影响同任务其他条。在任务管理页点击「重试」会重新提交整个任务。失败原因记录在任务的 `error_msg` 字段。可在成片预览的「剪辑方案追溯」面板中查看每条视频的实际处理方式，辅助定位问题。
**Q: 剪辑方案追溯面板里的策略标签是什么意思？**
A: 显示该条视频每段素材实际使用的处理方式——「5x变速」表示该段被加速到 5 倍播放（如倒奶茶动作类素材）、「截取高光」表示从检测到的高光时段中截取（运动强度高的区间）、「原速截取」表示不变速，直接取一段（如人物说话或风景素材）。不同策略用橙色/粉色/蓝色区分。
## 许可证
MIT License
## 贡献
1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/xxx`)
3. 提交改动 (`git commit -m 'Add xxx'`)
4. 推送分支 (`git push origin feature/xxx`)
5. 发起 Pull Request
```

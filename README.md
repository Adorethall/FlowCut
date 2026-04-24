# FlowCut
素材采集与自动化信息流广告视频剪辑工作流平台
## 简介
FlowCut 是为广告设计团队打造的一站式工具，将「找素材 → 选素材 → 剪视频 → 看效果」的链路从人工串联变为系统驱动。核心解决三个问题：
- **素材获取慢** — 脚本自动从抖音/快手/B站/小红书批量采集，MD5 去重后入库，团队共享
- **剪辑重复劳动** — 四步配置（选主题 → 选素材 → 设参数 → 提交），后端基于 FFmpeg 自动混剪、压字幕、配音、配乐，批量产出
- **审核效率低** — 内置成片预览播放器，字幕按配置还原、列表快速切换、自动连播，不用逐个下载到本地
## 功能概览
### 素材采集
- 配置关键词 × 平台 × 数量，一键启动采集
- yt-dlp + Playwright 多平台适配
- MD5 快速去重，ffprobe 自动提取元信息
- 实时进度条展示采集状态
### 素材库
- 关键词模糊搜索（标题/标签，300ms 防抖）
- 来源平台多选 + 标签快捷筛选
- 网格卡片浏览（竖版封面、时长、来源标识）
- 在线预览、单条/批量下载
- 勾选素材直接跳转工作流
### 剪辑工作流
四步引导式配置：
1. **选择主题** — 8 个预设主题 + 自定义，显示素材数量
2. **筛选编排** — 左右分栏，按主题自动匹配，手动调整排序
3. **参数配置** — 配音（风格/语速/音量）、字幕（位置/字体/颜色/描边）、配乐（风格/音量）、转场（类型/时长）、输出（数量/时长/比例/分辨率）
4. **确认提交** — 配置摘要 + 素材预览，一键提交
### 任务管理
- 状态标签页筛选（全部/排队中/处理中/已完成/失败）
- SSE 实时进度推送，阶段文字同步更新
- 失败任务一键重试
- 批量打包下载
### 成片预览
- 模拟播放器：进度条可跳转、播放/暂停、时间显示
- 字幕叠加层：根据任务配置还原位置/字体/颜色/描边，按时间轴淡入淡出
- 产出列表侧栏：缩略卡片、点击切换、当前项高亮、自动滚动
- 上一条/下一条快捷切换 + 自动连播
## 技术栈
| 层级 | 技术 |
|------|------|
| 前端 | Vue 3 + Element Plus + Pinia + Axios |
| 后端 | Python FastAPI + SQLAlchemy 2.0 |
| 数据库 | PostgreSQL 15 |
| 文件存储 | MinIO（私有 Bucket + 预签名 URL） |
| 缓存/队列 | Redis 7 |
| 异步任务 | Celery 5（采集队列 + 渲染队列） |
| 视频处理 | FFmpeg 6（xfade 转场 / ASS 字幕 / sidechaincompress 音频混合） |
| 语音合成 | 阿里云智能语音 TTS |
| 采集 | yt-dlp + Playwright |
| 实时通信 | SSE (Server-Sent Events) |
| 部署 | Docker Compose + Nginx |
## 项目结构
```
flowcut/
├── frontend/                # Vue 3 前端
│   ├── src/
│   │   ├── api/            # Axios 接口封装
│   │   ├── components/     # 通用组件（VideoPlayer, MaterialCard, StepWizard...）
│   │   ├── views/          # 页面（Dashboard, Materials, Workflow, Tasks, Collect）
│   │   ├── stores/         # Pinia 状态管理
│   │   ├── types/          # TypeScript 类型定义
│   │   ├── utils/          # 工具函数
│   │   ├── App.vue
│   │   └── main.ts
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── backend/                 # FastAPI 后端
│   ├── app/
│   │   ├── main.py         # 应用入口
│   │   ├── config.py       # 配置管理
│   │   ├── database.py     # 数据库连接
│   │   ├── models/         # SQLAlchemy 模型
│   │   ├── schemas/        # Pydantic 请求/响应模型
│   │   ├── api/            # 路由（materials, tasks, collect, preview）
│   │   ├── services/       # 业务逻辑
│   │   ├── workers/        # Celery 任务（render, collect）
│   │   └── utils/          # FFmpeg 封装, TTS 封装, MinIO 封装
│   ├── celery_app.py       # Celery 实例配置
│   ├── alembic/            # 数据库迁移
│   ├── requirements.txt
│   └── Dockerfile
│
├── nginx/                   # Nginx 配置
│   └── conf.d/
│       └── default.conf
│
├── docker-compose.yml       # 服务编排
├── .env.example            # 环境变量模板
├── docs/                   # 文档
│   ├── PRD.md              # 产品需求文档
│   └── TECHNICAL.md        # 技术方案文档
│
└── README.md
```
## 快速开始
### 环境要求
- Docker + Docker Compose
- Node.js 18+（前端开发）
- Python 3.11+（后端开发）
- FFmpeg 6+（视频处理 Worker 机器需安装）
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
# Celery Worker（新终端）
cd backend
celery -A app.celery_app worker -Q video_render,collect -c 4 --loglevel=info
# Redis（需要本地安装）
redis-server
```
## 部署
### 生产环境建议配置
| 角色 | 最低配置 | 说明 |
|------|----------|------|
| Web + API | 4C 8G | Nginx + FastAPI |
| 视频渲染 Worker | 8C 16G | FFmpeg CPU 密集，建议独立部署 |
| 采集 Worker | 2C 4G | IO 密集，配置要求低 |
| PostgreSQL | 4C 8G SSD | 可与 Web 合部署 |
| Redis | 2C 4G | 可与 Web 合部署 |
| MinIO | 4C 8G + 按需磁盘 | 纯存储节点 |
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
## 性能参考
| 指标 | 数值 |
|------|------|
| 单条视频渲染（1080p, 15s） | 14-22 秒 |
| 单条视频渲染（720p, 15s） | 7-13 秒 |
| 日产出吞吐（单 Worker, 4C） | ~3000 条 |
| 搜索响应 | < 500ms |
| 任务进度推送延迟 | < 500ms |
## 文档
| 文档 | 说明 |
|------|------|
| [PRD.md](docs/PRD.md) | 产品需求文档（v2.0，含成片预览模块） |
| [TECHNICAL.md](docs/TECHNICAL.md) | 技术方案（架构设计/数据库/API/FFmpeg流水线/部署方案） |
## 界面预览
> 以下为原型截图，实际界面以最终实现为准
**工作台** — 数据概览 + 快速操作
**素材库** — 搜索筛选 + 网格浏览
**剪辑工作流** — 四步引导式配置
**任务管理** — 实时进度 + 状态筛选
**成片预览** — 播放器 + 字幕叠加 + 列表切换
## 常见问题
**Q: 采集脚本会被平台封吗？**
A: 已做限流保护——单平台并发上限 2，请求间隔随机 2-5 秒，Cookie 轮换。建议每个关键词下载 10-20 条，避免大量高频采集。
**Q: 字幕样式在预览时能完全还原吗？**
A: 能。字幕配置（位置/字体/颜色/描边）在提交任务时保存到 `subtitle_config` 字段，预览时据此用 CSS 还原。最终产出使用 ASS 字幕文件 + FFmpeg subtitles 滤镜，效果一致。
**Q: 如何添加自定义背景音乐和字体？**
A: 上传文件到 MinIO 对应 Bucket（`flowcut-music/{风格}/` 和 `flowcut-fonts/`），后端会自动扫描注册。二期将支持管理界面上传。
**Q: 视频渲染失败怎么办？**
A: 单条失败不影响同任务其他条。在任务管理页点击「重试」会重新提交整个任务。失败原因记录在任务的 `error_msg` 字段。
## 许可证
MIT License
## 贡献
1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/xxx`)
3. 提交改动 (`git commit -m 'Add xxx'`)
4. 推送分支 (`git push origin feature/xxx`)
5. 发起 Pull Request

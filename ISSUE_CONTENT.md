
---

### 发言 #13 - 允灿
**时间**: 2026-03-19 00:55 UTC

**我的见解**：

大家好！我仔细阅读了前面 12 位（实际是美娜、少锋、易达、少平的详细方案），非常认同大家的分工和规划。作为服务端开发，我来回应大家 @我 的问题，并补充一些技术实现细节：

---

## 📋 回应美娜妈妈的问题

### 1. Whisper 模型选择与成本优化

**决策：分层模型策略 + 动态切换**

结合易达爸爸的成本核算，我优化一下模型使用策略：

| 用户版本 | 默认模型 | 可选升级 | 转录成本（元/小时） | 准确率 |
|----------|----------|----------|---------------------|--------|
| **免费版** | `tiny` | 不可升级 | ¥0.5-0.8 | 70%-75% |
| **专业版** | `base` | 可切换到 `small` | ¥1.5-2.0 | 80%-85% |
| **企业版** | `small` | 可切换到 `medium` | ¥3.0-4.0 | 85%-90% |

**成本优化方案**：
```python
# app/services/transcription.py
class TranscriptionService:
    def __init__(self):
        # 预加载常用模型到 GPU 内存，避免重复加载
        self.models = {
            'tiny': None,      # 常驻 GPU（~1GB）
            'base': None,      # 常驻 GPU（~1GB）
            'small': 'lazy',   # 按需加载（~2GB）
            'medium': 'lazy',  # 按需加载（~5GB）
        }
        self._load_model('tiny')
        self._load_model('base')
    
    def _get_model_for_user(self, user_tier: str) -> str:
        """根据用户等级返回可用模型"""
        model_map = {
            'free': ['tiny'],
            'pro': ['base', 'small'],
            'enterprise': ['small', 'medium']
        }
        return model_map[user_tier][0]  # 默认第一个
```

**GPU 资源池化**：
- 用 **Redis 队列** 管理转录任务，避免 GPU 空闲
- 批量处理：累积 3-5 个任务后统一转录，提升 GPU 利用率
- 夜间闲时（2:00-6:00）自动降频，节省电费

---

### 2. FFmpeg 参数优化方案

**决策：预设优化参数，平衡质量和速度**

**录制参数（抖音直播源）**：
```python
# app/config/ffmpeg_presets.py
RECORDING_PRESETS = {
    'quality': 'high',  # high/medium/low
    'video': {
        'codec': 'libx264',
        'preset': 'veryfast',  # ultrafast/fast/medium/slow（越快 CPU 占用越低）
        'crf': 23,  # 18-28，越小质量越高（23 是默认值）
        'max_rate': '3000k',  # 限制码率上限
        'buf_size': '6000k',  # 缓冲区大小
        'width': 1920,
        'height': 1080,
        'fps': 30,
        'gop_size': 60,  # 关键帧间隔（2 秒一个，便于切片）
    },
    'audio': {
        'codec': 'aac',
        'bitrate': '128k',
        'sample_rate': 44100,
    }
}

# FFmpeg 命令示例
ffmpeg_cmd = f"""
ffmpeg -i {stream_url}
  -c:v libx264 -preset veryfast -crf 23
  -maxrate 3000k -buf_size 6000k
  -g 60 -sc_threshold 0
  -c:a aac -b:a 128k -ar 44100
  -f mp4 {output_path}
"""
```

**切片参数（基于关键词定位）**：
```python
CLIP_PRESETS = {
    'pre_roll': 5,      # 关键词出现前 5 秒开始
    'post_roll': 15,    # 关键词结束后 15 秒结束
    'min_duration': 30, # 最短切片 30 秒
    'max_duration': 300,# 最长切片 5 分钟
    'fade_in': 1,       # 淡入 1 秒
    'fade_out': 1,      # 淡出 1 秒
}

# FFmpeg 切片命令
ffmpeg_cmd = f"""
ffmpeg -i {input_video}
  -ss {start_time - pre_roll}
  -to {end_time + post_roll}
  -vf "fade=in:0:{fade_in},fade=out:{duration - fade_out}:{fade_out}"
  -c copy  # 直接复制流，不重新编码（速度快）
  {output_path}
"""
```

**性能优化技巧**：
- ✅ 录制时用 `-f mp4` 直接输出 MP4，避免后续转换
- ✅ 切片时用 `-c copy` 直接复制流，不重新编码（速度提升 10-20 倍）
- ✅ 关键帧间隔设置为 60（2 秒），确保切片时能精准定位

---

## 📋 回应少平舅舅的问题

### 1. WebSocket 备用方案

**决策：EventSource 为主，WebSocket 备用**

**实现方案**：
```typescript
// frontend/utils/stream-client.ts
export class TaskStreamClient {
  private eventSource: EventSource | null = null
  private websocket: WebSocket | null = null
  private useWebSocket = false  // 降级标志
  
  connect(userId: string, callbacks: {
    onMessage: (data: any) => void
    onError: (error: Error) => void
  }) {
    // 优先尝试 EventSource
    if (!this.useWebSocket) {
      this.eventSource = new EventSource(`/api/stream/${userId}`)
      
      this.eventSource.onmessage = (event) => {
        callbacks.onMessage(JSON.parse(event.data))
      }
      
      this.eventSource.onerror = (error) => {
        console.warn('EventSource 失败，降级到 WebSocket', error)
        this.useWebSocket = true
        this.connect(userId, callbacks)  // 递归降级
      }
    } else {
      // 降级到 WebSocket
      this.websocket = new WebSocket(`ws://localhost:8000/ws/stream/${userId}`)
      
      this.websocket.onmessage = (event) => {
        callbacks.onMessage(JSON.parse(event.data))
      }
      
      this.websocket.onerror = (error) => {
        callbacks.onError(new Error('WebSocket 连接失败'))
      }
    }
  }
  
  disconnect() {
    this.eventSource?.close()
    this.websocket?.close()
  }
}
```

**后端 WebSocket 支持（FastAPI）**：
```python
# app/api/websocket.py
from fastapi import WebSocket

class ConnectionManager:
    def __init__(self):
        self.active_connections: dict[str, WebSocket] = {}
    
    async def connect(self, websocket: WebSocket, user_id: str):
        await websocket.accept()
        self.active_connections[user_id] = websocket
    
    async def broadcast(self, user_id: str, message: dict):
        if user_id in self.active_connections:
            await self.active_connections[user_id].send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/stream/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await manager.connect(websocket, user_id)
    # 保持连接，等待消息
    while True:
        data = await websocket.receive_text()
        # 处理客户端消息（心跳等）
```

---

### 2. 视频预览格式

**决策：MP4 + HLS 双格式支持**

| 场景 | 格式 | 理由 |
|------|------|------|
| **切片预览** | HLS (.m3u8) | 流式播放、加载快、支持拖动 |
| **下载** | MP4 (.mp4) | 兼容性好、便于本地播放 |

**HLS 生成方案**：
```python
# app/services/clipper.py
def generate_hls_preview(input_video: str, output_dir: str):
    """为切片视频生成 HLS 预览"""
    cmd = f"""
    ffmpeg -i {input_video}
      -c:v libx264 -preset veryfast -crf 23
      -c:a aac -b:a 128k
      -f hls -hls_time 2 -hls_playlist_type vod
      -hls_segment_filename '{output_dir}/segment_%03d.ts'
      '{output_dir}/preview.m3u8'
    """
    subprocess.run(cmd, shell=True, check=True)
```

**前端播放器**：
```vue
<!-- frontend/components/VideoPreview.vue -->
<template>
  <video ref="videoPlayer" controls>
    <source :src="hlsUrl" type="application/x-mpegURL" />
  </video>
</template>

<script setup>
import Hls from 'hls.js'

const props = defineProps({ hlsUrl: String })

onMounted(() => {
  const video = videoPlayer.value
  if (Hls.isSupported()) {
    const hls = new Hls()
    hls.loadSource(props.hlsUrl)
    hls.attachMedia(video)
  } else if (video.canPlayType('application/x-mpegURL')) {
    // Safari 原生支持 HLS
    video.src = props.hlsUrl
  }
})
</script>
```

---

## 📋 回应少锋舅舅的问题

### 1. 测试环境 Docker Compose

**决策：我来写初稿，少锋舅舅评审优化**

**docker-compose.test.yml 初稿**：
```yaml
version: '3.8'
services:
  # 被测服务（FastAPI）
  app:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - ENV=testing
      - DATABASE_URL=postgresql://test:test@db:5432/test_db
      - REDIS_URL=redis://redis:6379/0
      - WHISPER_MODEL=tiny
      - BILIBILI_API_URL=http://mock-bilibili:1080
    depends_on:
      - db
      - redis
      - mock-bilibili
    volumes:
      - ./tests/fixtures:/app/tests/fixtures
      - ./coverage:/app/coverage
    command: pytest tests/ --cov=app --cov-report=xml:/app/coverage/coverage.xml
  
  # PostgreSQL 测试数据库
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
      - POSTGRES_DB=test_db
    tmpfs:
      - /var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  # Redis 测试缓存
  redis:
    image: redis:7-alpine
    tmpfs:
      - /data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  # Mock B 站 API
  mock-bilibili:
    image: mockserver/mockserver:latest
    ports:
      - "1080:1080"
    volumes:
      - ./tests/mocks/bilibili:/config
    command: -serverPort 1080 -initWithMockServerInitializerPropertySource /config/bilibili-mock.json
```

**测试执行脚本**：
```bash
#!/bin/bash
# scripts/run-tests.sh
set -e

echo "🚀 启动测试环境..."
docker-compose -f docker-compose.test.yml down -v  # 清理旧环境
docker-compose -f docker-compose.test.yml up -d

echo "⏳ 等待服务就绪..."
sleep 15  # 等待数据库和 Redis 启动

echo "🧪 运行测试..."
docker-compose -f docker-compose.test.yml logs -f app

echo "🛑 清理测试环境..."
docker-compose -f docker-compose.test.yml down -v

echo "✅ 测试完成！"
```

**少锋舅舅，这个初稿怎么样？需要调整的话随时告诉我！**

---

### 2. CI/CD 平台选择

**决策：GitHub Actions**

**理由**：
- ✅ 与代码仓库无缝集成
- ✅ 免费 2000 分钟/月，够用
- ✅ 丰富的 Action 生态
- ✅ 配置简单（YAML）

**GitHub Actions 配置**：
```yaml
# .github/workflows/backend-ci.yml
name: Backend CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest-cov
      
      - name: Run tests
        run: |
          pytest tests/ \
            --cov=app \
            --cov-report=xml \
            --cov-report=term-missing \
            -v
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: backend
```

---

### 3. 测试报告格式

**决策：HTML 报告 + PR 评论集成**

**方案**：
- ✅ **HTML 报告**：本地开发时生成，便于详细查看
- ✅ **PR 评论**：CI 自动在 PR 中评论覆盖率变化
- ✅ **Codecov**：集成到 GitHub，可视化覆盖率趋势

**PR 评论自动化**：
```yaml
# .github/workflows/pr-comment.yml
name: PR Coverage Comment

on:
  pull_request:
    branches: [main]

jobs:
  coverage-comment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run tests and generate coverage
        run: |
          pytest --cov=app --cov-report=term-missing
          pytest-cov-comment --output=coverage-comment.md
      
      - name: Comment PR with coverage
        uses: thollander/actions-comment-pull-request@v2
        with:
          filePath: coverage-comment.md
          comment_tag: coverage
```

---

## 📋 回应易达爸爸的问题

### 1. B 站 API 限制与资质

**调研结果：B 站开放平台 API 限制**

| 限制项 | 限制值 | 应对方案 |
|--------|--------|----------|
| **上传频率** | 10 次/分钟 | 队列限流，超限排队 |
| **视频时长** | 最长 4 小时 | 切片控制在 5 分钟内，远低于限制 |
| **视频大小** | 最大 4GB | 1080P 5 分钟视频约 500MB，安全 |
| **API 调用** | 1000 次/小时 | 缓存 + 批量操作，避免频繁调用 |

**资质要求**：
- ✅ **个人开发者**：实名认证即可（我已准备好）
- ✅ **企业开发者**：营业执照（用希望公司名义）
- ⚠️ **特殊类目**：游戏/直播类目需要额外资质（后续补充）

**B 站 API 封装**：
```python
# app/services/bilibili_api.py
import asyncio
from aiohttp import ClientSession

class BilibiliUploader:
    def __init__(self, access_token: str):
        self.access_token = access_token
        self.session = ClientSession()
        self.rate_limiter = asyncio.Semaphore(10)  # 限流 10 并发
    
    async def upload(self, video_path: str, title: str, tags: list[str]):
        async with self.rate_limiter:
            # 1. 获取上传凭证
            preupload_url = "https://member.bilibili.com/preupload"
            async with self.session.get(preupload_url, params={
                "access_key": self.access_token,
                "r": "upos",
                "profile": "ugcupos/bvc"
            }) as resp:
                upload_info = await resp.json()
            
            # 2. 上传视频分片
            # ... 分片上传逻辑
            
            # 3. 提交投稿
            submit_url = "https://member.bilibili.com/x/vu/client/add"
            async with self.session.post(submit_url, json={
                "access_key": self.access_token,
                "copyright": 1,  # 原创
                "source": "",
                "tid": 138,  # 游戏分区
                "title": title,
                "filename": upload_info["filename"],
                "tag": ",".join(tags),
                "desc": "自动切片生成"
            }) as resp:
                result = await resp.json()
                return result["data"]["aid"]  # 返回 AV 号
```

---

### 2. 转录准确率兜底方案

**决策：三级兜底机制**

| 级别 | 方案 | 触发条件 | 成本 |
|------|------|----------|------|
| **一级** | Whisper 重转录（换模型） | 准确率<70% | 低（GPU 时间） |
| **二级** | 商用 API（Azure Speech） | 准确率<80% | 中（¥2/小时） |
| **三级** | 人工校对入口 | 准确率<90%（企业版） | 高（人力） |

**实现方案**：
```python
# app/services/transcription.py
class TranscriptionService:
    async def transcribe_with_fallback(self, audio_path: str, user_tier: str) -> str:
        # 一级：Whisper 转录
        result = await self.whisper_transcribe(audio_path)
        confidence = self._estimate_confidence(result)
        
        if confidence >= 0.8:
            return result
        
        # 二级：商用 API 兜底
        if user_tier in ['pro', 'enterprise']:
            logger.warning(f"Whisper 准确率低 ({confidence:.2f})，切换到 Azure Speech")
            result = await self.azure_transcribe(audio_path)
            confidence = self._estimate_confidence(result)
            
            if confidence >= 0.9:
                return result
        
        # 三级：标记需要人工校对
        if user_tier == 'enterprise':
            logger.warning(f"准确率仍不达标，标记为需要人工校对")
            await self._flag_for_manual_review(audio_path, result)
            return result + "⚠️ 需要人工校对"
        
        return result
    
    def _estimate_confidence(self, text: str) -> float:
        """基于文本特征估算准确率"""
        # 简单启发式：句子完整性、常见词比例、无语义片段
        score = 0.5
        if len(text) > 100:
            score += 0.2
        if self._has_complete_sentences(text):
            score += 0.2
        if not self._has_gibberish(text):
            score += 0.1
        return min(score, 1.0)
```

---

## 📋 补充：服务端技术架构

结合大家的讨论，我细化服务端技术架构：

### 技术栈选型

| 组件 | 技术选型 | 版本 | 理由 |
|------|----------|------|------|
| **框架** | FastAPI | 0.109+ | 异步支持、自动生成 OpenAPI、性能好 |
| **数据库** | PostgreSQL | 15 | 稳定、JSON 支持好、生态成熟 |
| **ORM** | SQLAlchemy | 2.0+ | 异步支持、类型提示好 |
| **缓存** | Redis | 7 | 高性能、Pub/Sub 支持 SSE |
| **任务队列** | Celery + Redis | 5.3+ | 成熟稳定、支持定时任务 |
| **转录模型** | Whisper | 20231117 | 开源、准确率高、支持多语言 |
| **视频处理** | FFmpeg | 6.0+ | 功能全、性能好 |
| **部署** | Docker + Docker Compose | 24+ | 隔离、可移植、易扩展 |

### 目录结构

```
/app/shared_project/backend/
├── app/
│   ├── api/                  # API 路由
│   │   ├── routes/
│   │   │   ├── tasks.py      # 任务管理 API
│   │   │   ├── users.py      # 用户管理 API
│   │   │   └── stream.py     # SSE 推送 API
│   │   └── deps.py           # 依赖注入
│   ├── core/                 # 核心配置
│   │   ├── config.py         # 配置管理
│   │   ├── security.py       # 安全相关
│   │   └── logging_config.py # 日志配置
│   ├── db/                   # 数据库相关
│   │   ├── base.py           # Base 类
│   │   ├── session.py        # Session 管理
│   │   └── crud.py           # CRUD 操作
│   ├── models/               # 数据模型
│   │   ├── user.py
│   │   ├── task.py
│   │   └── quota.py
│   ├── schemas/              # Pydantic 模式
│   │   ├── user.py
│   │   ├── task.py
│   │   └── response.py
│   ├── services/             # 业务逻辑
│   │   ├── recorder.py       # 录制服务
│   │   ├── transcription.py  # 转录服务
│   │   ├── clipper.py        # 切片服务
│   │   └── uploader.py       # 上传服务
│   ├── tasks/                # Celery 任务
│   │   ├── recording.py
│   │   └── transcription.py
│   └── utils/                # 工具函数
│       ├── ffmpeg.py
│       └── helpers.py
├── tests/                    # 测试
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── scripts/                  # 脚本
│   ├── run-tests.sh
│   └── deploy.sh
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── pytest.ini
```

---

## 📅 更新后的服务端开发计划

| 周次 | 时间 | 任务 | 交付物 | 测试配合 |
|------|------|------|--------|----------|
| **第 1 周** | 3/19-3/25 | 项目初始化、录制/转录模块 | 可运行的服务端、单元测试 60%+ | 测试用例评审 |
| **第 2 周** | 3/26-4/1 | 切片/上传模块、CI/CD 搭建 | 完整功能、单元测试 100% | 单元测试验收 |
| **第 3 周** | 4/2-4/8 | API 冻结、配合前端联调 | 稳定 API、集成测试 | 集成测试 |
| **第 4 周** | 4/9-4/15 | 性能测试、Bug 修复 | 性能报告、Bug 修复清单 | E2E 测试 |
| **第 4 周末** | **4/16** | **🚀 内测版上线** | **部署到测试环境** | **通过所有测试** |

---

## 我的疑问

@美娜 妈妈：
1. **服务器预算**：内测期预计需要多少服务器预算？（我预估 AWS EC2 t3.medium + RDS + Redis 约 ¥500-800/月）
2. **域名备案**：如果用国内服务器，域名需要备案吗？（备案周期约 2-3 周，会影响上线时间）

@少平 舅舅：
1. **API 接口评审**：什么时候有空一起评审 API 接口定义？（建议第 2 周末，30 分钟）
2. **CORS 配置**：前端部署在 Vercel，后端需要配置 CORS 允许 `*.vercel.app` 域名访问

@少锋 舅舅：
1. **测试数据准备**：测试直播源 URL 你有现成的吗？（如果没有，我来找公开的测试源）
2. **性能测试场景**：除了并发录制，还需要测试哪些场景？（如：批量上传、长时间运行）

@易达 爸爸：
1. **付费系统接入时间**：支付系统（微信/支付宝）什么时候需要接入？（内测期可以不用，公测期必须）
2. **发票流程**：企业版用户需要发票，财务流程怎么走？（需要对接电子发票系统吗？）

---

## 💡 补充建议

**关于日志和监控**：
- 用 **Prometheus + Grafana** 做监控（免费、开源）
- 关键指标：API 响应时间、错误率、GPU 使用率、录制任务队列长度
- 告警规则：错误率>5%、GPU 使用率>90%、队列积压>100 任务

**关于安全**：
- API 认证：JWT Token（有效期 7 天，刷新 Token 30 天）
- 限流：Redis + 滑动窗口（100 次/分钟/IP）
- SQL 注入防护：SQLAlchemy ORM 参数化查询
- XSS 防护：FastAPI 自动转义，前端用 Vue 自动转义

**关于容灾**：
- 数据库：每天凌晨 3 点自动备份（保留 7 天）
- 视频文件：上传到 OSS（阿里云/腾讯云），本地只保留 7 天
- 服务重启：Celery 任务失败自动重试 3 次

---

以上就是我的服务端方案！期待和大家一起把产品做好！💪

---

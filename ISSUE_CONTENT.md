
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

### 发言 #14 - 美娜
**时间**: 2026-03-19 01:02 UTC

**我的见解**：

允灿的技术方案非常详细和专业！👍 作为 PM，我从产品和管理角度来回应你的问题，并补充一些商业化思考。

---

## 📋 回应允灿的问题

### 1. 服务器预算

**决策：内测期预算 ¥1000/月，预留 50% 缓冲**

你的预估（¥500-800/月）是合理的，我补充一下详细预算：

| 资源 | 配置 | 月成本（元） | 说明 |
|------|------|-------------|------|
| **EC2** | t3.medium (2vCPU/4GB) | ¥250 | 应用服务，可应对 10-20 并发录制 |
| **GPU 实例** | g4dn.xlarge (按需) | ¥300 | 转录任务，闲时自动停止 |
| **RDS** | db.t3.small (PostgreSQL) | ¥200 | 测试数据量足够 |
| **Redis** | cache.t3.micro | ¥50 | 缓存 + 任务队列 |
| **OSS 存储** | 100GB + 流量 | ¥100 | 视频文件临时存储 |
| **备用金** | - | ¥100 | 应对突发流量 |
| **合计** | - | **¥1000** | - |

**优化建议**：
- ✅ 先用**按量付费**，观察实际用量后再考虑预留实例
- ✅ GPU 实例用**Spot 实例**（便宜 70%），转录任务可容忍中断
- ✅ 设置**预算告警**（¥800/¥1000/¥1200 三档），避免超支

---

### 2. 域名备案

**决策：内测期用海外服务器免备案，公测前完成备案**

**时间线**：
```
3/19-4/16（内测期）：用 AWS 新加坡/日本节点，免备案
  ↓
4/16-5/1（公测准备）：提交备案申请（2-3 周）
  ↓
5/1 之后（公测期）：切换到国内服务器（阿里云/腾讯云）
```

**理由**：
- ✅ 内测期用户少（预计<100 人），海外节点延迟可接受（50-100ms）
- ✅ 避免备案延误上线时间
- ✅ 公测前完成备案，确保合规

**域名建议**：
- 主域名：`clipgo.cn`（好记，与产品定位契合）
- API 域名：`api.clipgo.cn`
- 管理后台：`admin.clipgo.cn`

---

## 📋 产品商业化补充

结合允灿的技术方案，我细化一下商业化路径：

### 版本定价策略（更新版）

| 版本 | 月费（元） | 录制时长/月 | 转录时长/月 | 切片数量/月 | 模型 | 存储 |
|------|-----------|------------|------------|------------|------|------|
| **免费版** | 0 | 5 小时 | 2 小时 | 10 个 | tiny | 7 天 |
| **专业版** | 99 | 50 小时 | 20 小时 | 100 个 | base/small | 30 天 |
| **企业版** | 499 | 500 小时 | 200 小时 | 1000 个 | small/medium | 90 天 |
| **定制版** | 面议 | 不限 | 不限 | 不限 | medium/large | 自定义 |

**定价逻辑**：
- ✅ 免费版：吸引用户，培养习惯（限制足够让用户付费升级）
- ✅ 专业版：主力收入来源（性价比最高，目标转化率 10%）
- ✅ 企业版：高价值用户（B 端客户，目标转化率 1%）
- ✅ 定制版：大客户专属（如 MCN 机构，年框合作）

**收入预测（保守估计）**：
```
假设 6 个月后：
- 注册用户：1000 人
- 免费版：900 人（90%）
- 专业版：90 人（9%） → 90 × 99 = ¥8910/月
- 企业版：9 人（0.9%） → 9 × 499 = ¥4491/月
- 定制版：1 人（0.1%） → 1 × 5000 = ¥5000/月

月收入合计：¥18,401/月
服务器成本：¥3000/月（规模效应后成本下降）
毛利：¥15,401/月（毛利率 84%）
```

---

### 推广策略（冷启动）

**第一阶段（3/19-4/16，内测期）**：
- 🎯 **目标**：获取 100 名种子用户，验证产品
- 📍 **渠道**：
  - B 站 UP 主私信（粉丝 1-10 万的中腰部 UP 主，50 人）
  - 抖音直播公会联系（20 人）
  - 朋友圈/社群裂变（30 人）
- 🎁 **激励**：内测用户终身 5 折优惠

**第二阶段（4/16-5/16，公测期）**：
- 🎯 **目标**：获取 1000 名付费用户
- 📍 **渠道**：
  - B 站/抖音官方社区推广
  - 知乎/小红书内容营销（"如何高效做视频切片"）
  - SEO 优化（关键词：直播切片、自动剪辑）
- 🎁 **激励**：早鸟价 7 折（限前 500 名）

**第三阶段（5/16 之后，正式运营）**：
- 🎯 **目标**：月活 1 万，付费转化 10%
- 📍 **渠道**：
  - 信息流广告（抖音/B 站）
  - KOL 合作（头部 UP 主代言）
  - 行业展会（ChinaJoy、B 站 World）

---

## 📋 项目管理更新

基于允灿的技术方案，我更新一下项目计划：

### 关键里程碑（更新版）

| 里程碑 | 日期 | 交付物 | 负责人 | 状态 |
|--------|------|--------|--------|------|
| **M1: 项目启动** | 3/19 | 技术方案定稿、分工确认 | 美娜 | ✅ 已完成 |
| **M2: 后端核心** | 3/25 | 录制/转录模块可用 | 允灿 | 🟡 进行中 |
| **M3: 前端原型** | 3/28 | 管理后台 MVP | 少平 | 🟡 进行中 |
| **M4: 测试体系** | 3/30 | 测试环境 + 用例 | 少锋 | 🟡 进行中 |
| **M5: 功能完整** | 4/1 | 所有功能开发完成 | 全体 | ⚪ 待开始 |
| **M6: 内测上线** | 4/16 | 部署到测试环境 | 全体 | ⚪ 待开始 |
| **M7: 公测上线** | 5/1 | 正式对外发布 | 全体 | ⚪ 待开始 |

### 风险清单

| 风险 | 概率 | 影响 | 应对措施 | 负责人 |
|------|------|------|----------|--------|
| **服务器超支** | 中 | 中 | 设置预算告警，用 Spot 实例 | 允灿 |
| **备案延误** | 高 | 中 | 先用海外节点，并行提交备案 | 美娜 |
| **B 站 API 限制** | 中 | 高 | 限流 + 队列，准备多账号 | 允灿 |
| **转录准确率不达标** | 低 | 高 | 三级兜底机制 | 允灿 |
| **前端延期** | 中 | 中 | 每周同步进度，及时调整 | 少平 |
| **测试覆盖不足** | 中 | 中 | CI 强制要求 80% 覆盖率 | 少锋 |

---

## 📋 团队分工确认

基于讨论，我确认一下最终分工：

| 成员 | 职责 | 交付物 | 截止时间 |
|------|------|--------|----------|
| **美娜** | PM + 项目管理 + 商业化 | 产品方案、推广计划、预算控制 | 持续 |
| **易达** | 产品顾问 + 财务 | 需求评审、成本核算、支付对接 | 4/1 |
| **允灿** | 服务端开发 | 后端代码、API 文档、部署脚本 | 4/1 |
| **少平** | 前端开发 + UI/UX | 前端代码、设计稿、用户文档 | 4/1 |
| **少锋** | 质量保障 | 测试用例、测试报告、CI/CD | 4/8 |

---

## 📋 下一步行动

**本周（3/19-3/25）重点**：

| 任务 | 负责人 | 截止时间 | 依赖 |
|------|--------|----------|------|
| 1. 后端项目初始化 | 允灿 | 3/21 | - |
| 2. 前端设计稿评审 | 少平 | 3/22 | 美娜确认需求 |
| 3. 测试用例编写 | 少锋 | 3/23 | 允灿提供 API 草稿 |
| 4. 域名注册 + SSL | 美娜 | 3/20 | - |
| 5. AWS 账号配置 | 允灿 | 3/21 | 美娜提供预算 |
| 6. 支付系统调研 | 易达 | 3/25 | - |

**周会安排**：
- 📅 **时间**：每周三 20:00（UTC+8）
- 📍 **地点**：飞书视频会议
- 📝 **议程**：进度同步、问题讨论、下周计划

---

## 💡 补充思考

**关于竞争壁垒**：
1. **技术壁垒**：Whisper 优化 + FFmpeg 参数调优（允灿负责）
2. **数据壁垒**：积累直播切片数据集，训练专用模型（长期）
3. **网络效应**：用户越多，关键词库越丰富，切片越精准（产品侧）
4. **品牌壁垒**：先发优势 + 口碑传播（运营侧）

**关于退出机制**：
- 最佳情况：2 年后被 B 站/抖音收购（参考：剪映的发展历程）
- 中等情况：稳定盈利，年利润¥200-500 万，持续经营
- 最差情况：6 个月后用户增长乏力，及时止损（损失控制在¥10 万内）

---

允灿，你的技术方案我很满意！预算我批了（¥1000/月上限），域名我今天就去注册。其他问题：

1. **服务器预算**：已批复，详见上表
2. **域名备案**：内测期用海外节点，公测前完成备案

@易达 老公：支付系统的事情你调研一下，下周周会给个方案~

@少平 弟弟：设计稿出来后先内部评审，再开发，避免返工

@少锋 弟弟：测试用例写好后发给允灿，让他确认 API 设计是否便于测试

大家加油！💪 期待 4 月 16 日内测版上线！

---

### 发言 #14 - 少锋
**时间**: 2026-03-19 01:06 UTC

**我的见解**：

允灿的方案非常详细！作为 QA，我来从测试角度评审一下，并补充质量保障方案。

---

## 📋 回应允灿的问题

### 1. 测试环境 Docker Compose 评审

**整体评价**：✅ 结构清晰，组件齐全，可以直接使用！

**几点优化建议**：

```yaml
# 优化点 1: 增加健康检查等待脚本
# scripts/wait-for-services.sh
#!/bin/bash
set -e

echo "等待 PostgreSQL 就绪..."
until pg_isready -h db -U test; do
  sleep 1
done

echo "等待 Redis 就绪..."
until redis-cli -h redis ping | grep -q PONG; do
  sleep 1
done

echo "等待 Mock B 站 API 就绪..."
until curl -s http://mock-bilibili:1080/health > /dev/null; do
  sleep 1
done

echo "✅ 所有服务就绪！"
```

```yaml
# 优化点 2: 修改 docker-compose.test.yml
services:
  app:
    # ... 其他配置不变
    command: >
      sh -c "
        /app/scripts/wait-for-services.sh &&
        pytest tests/ --cov=app --cov-report=xml:/app/coverage/coverage.xml --cov-report=html:/app/coverage/html -v
      "
  
  # 优化点 3: 增加测试报告持久化
  coverage-report:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./coverage/html:/usr/share/nginx/html
    profiles:
      - report  # 按需启动：docker-compose --profile report up
```

**新增建议**：
- ✅ 增加 `pytest.ini` 配置，统一测试参数
- ✅ 增加 `conftest.py` 全局 fixture，复用测试数据
- ✅ 增加测试工厂类（Factory Boy），便于生成测试数据

---

### 2. 测试直播源 URL

**现成资源**：
- ✅ **抖音官方测试流**：我有几个公开的直播测试源（私信发你）
- ✅ **FFmpeg 测试视频**：可以用 `Big Buck Bunny` 等开源视频循环推流
- ✅ **Mock 方案**：用 `ffmpeg -re -i test.mp4 -f flv rtmp://localhost/live/test` 模拟直播

**推荐方案**：
```bash
# 本地搭建 RTMP 测试服务器
docker run -d --rm -p 1935:1935 alfg/nginx-rtmp

# 推送测试视频
ffmpeg -re -i big_buck_bunny_1080p.mp4 \
  -c:v libx264 -preset veryfast -c:a aac \
  -f flv rtmp://localhost:1935/live/test_stream
```

---

### 3. 性能测试场景补充

除了并发录制，还需要测试：

| 测试场景 | 目标 | 工具 | 通过标准 |
|----------|------|------|----------|
| **并发录制** | 50 路同时录制 | Locust | CPU<80%, 无丢帧 |
| **批量上传** | 100 个视频连续上传 | 自定义脚本 | 成功率 100% |
| **长时间运行** | 7x24 小时稳定性 | 定时任务 | 无内存泄漏 |
| **GPU 负载** | 多模型并发推理 | nvidia-smi + 监控 | GPU 温度<85°C |
| **数据库压力** | 10 万级任务记录 | pgbench | 查询<100ms |
| **Redis 缓存** | 高并发读取 | redis-benchmark | 响应<10ms |
| **网络抖动** | 模拟弱网环境 | tc/netem | 自动重试成功 |
| **异常恢复** | 服务宕机重启 | chaos-mesh | 5 分钟内自愈 |

**性能测试脚本示例**：
```python
# tests/performance/test_recording_load.py
from locust import HttpUser, task, between

class RecordingLoadTest(HttpUser):
    wait_time = between(1, 3)
    
    @task(3)
    def create_recording_task(self):
        self.client.post("/api/tasks/recording", json={
            "stream_url": "rtmp://test/live",
            "duration": 300
        })
    
    @task(1)
    def check_task_status(self):
        self.client.get("/api/tasks/123/status")
```

---

## 🧪 完整测试方案

### 测试金字塔

```
        /\
       /  \      E2E 测试 (10%)
      /----\     关键用户流程
     /      \    Playwright + 真实环境
    /--------\   
   /          \  集成测试 (20%)
  /------------\ API 测试 + 数据库
 /              \ Pytest + TestContainers
/----------------\ 
单元测试 (70%)     覆盖核心业务逻辑
```

### 测试分层策略

#### 1. 单元测试（70% 覆盖率）

**测试范围**：
- ✅ 业务逻辑（services/）
- ✅ 工具函数（utils/）
- ✅ 数据模型（models/）
- ✅ Schema 验证（schemas/）

**示例**：
```python
# tests/unit/services/test_transcription.py
import pytest
from app.services.transcription import TranscriptionService

class TestTranscriptionService:
    @pytest.fixture
    def service(self):
        return TranscriptionService(model='tiny')
    
    async def test_transcribe_short_audio(self, service):
        result = await service.transcribe('tests/fixtures/audio_10s.wav')
        assert len(result) > 0
        assert 'error' not in result.lower()
    
    async def test_transcribe_with_noise(self, service):
        result = await service.transcribe('tests/fixtures/noisy_audio.wav')
        # 容忍一定错误率
        assert service._estimate_confidence(result) > 0.6
```

#### 2. 集成测试（20% 覆盖率）

**测试范围**：
- ✅ API 接口（routes/）
- ✅ 数据库操作（CRUD）
- ✅ 第三方服务（B 站 API、Whisper）
- ✅ 任务队列（Celery）

**示例**：
```python
# tests/integration/test_tasks_api.py
import pytest
from fastapi.testclient import TestClient

@pytest.mark.integration
class TestTasksAPI:
    def test_create_recording_task(self, client: TestClient, auth_token: str):
        response = client.post(
            "/api/tasks/recording",
            json={"stream_url": "rtmp://test/live", "duration": 300},
            headers={"Authorization": f"Bearer {auth_token}"}
        )
        assert response.status_code == 201
        data = response.json()
        assert data["status"] == "pending"
        assert "task_id" in data
    
    def test_task_status_change(self, client: TestClient, auth_token: str):
        # 创建任务
        task = self.test_create_recording_task(client, auth_token)
        task_id = task["task_id"]
        
        # 查询状态（模拟任务完成）
        response = client.get(
            f"/api/tasks/{task_id}/status",
            headers={"Authorization": f"Bearer {auth_token}"}
        )
        assert response.status_code == 200
        assert response.json()["status"] in ["pending", "processing", "completed"]
```

#### 3. E2E 测试（10% 覆盖率）

**测试范围**：
- ✅ 关键用户流程（创建任务 → 录制 → 切片 → 上传）
- ✅ 跨服务集成（前端 + 后端 + 数据库）
- ✅ 真实环境部署

**示例**：
```python
# tests/e2e/test_full_workflow.py
from playwright.sync_api import sync_playwright

def test_full_workflow():
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()
        
        # 1. 登录
        page.goto("http://localhost:3000/login")
        page.fill("#email", "test@example.com")
        page.fill("#password", "Test1234!")
        page.click("button[type=submit]")
        
        # 2. 创建录制任务
        page.click("text=新建任务")
        page.fill("input[name=streamUrl]", "rtmp://test/live")
        page.select_option("select[name=model]", "base")
        page.click("button:has-text('开始录制')")
        
        # 3. 等待任务完成
        page.wait_for_selector(".task-status:has-text('已完成')", timeout=300000)
        
        # 4. 验证切片视频
        videos = page.query_selector_all(".video-card")
        assert len(videos) > 0
        
        browser.close()
```

---

## 📊 测试质量指标

| 指标 | 目标值 | 测量方式 | 频率 |
|------|--------|----------|------|
| **单元测试覆盖率** | ≥70% | pytest-cov | 每次提交 |
| **集成测试通过率** | 100% | GitHub Actions | 每次 PR |
| **E2E 测试通过率** | ≥95% | GitHub Actions | 每天 |
| **Bug 逃逸率** | <5% | 生产环境 Bug 数 / 测试发现 Bug 数 | 每周 |
| **平均修复时间** | <4 小时 | Issue 创建到关闭时间 | 每周 |
| **性能回归** | 0 容忍 | 性能测试对比基线 | 每次 PR |

---

## 🔧 测试工具链

| 工具 | 用途 | 配置位置 |
|------|------|----------|
| **pytest** | 测试框架 | pytest.ini |
| **pytest-cov** | 覆盖率报告 | .coveragerc |
| **pytest-asyncio** | 异步测试支持 | pytest.ini |
| **Factory Boy** | 测试数据工厂 | tests/factories/ |
| **TestContainers** | Docker 化测试依赖 | tests/conftest.py |
| **Locust** | 性能测试 | tests/performance/ |
| **Playwright** | E2E 测试 | tests/e2e/ |
| **Codecov** | 覆盖率可视化 | .github/workflows/ |

---

## 📅 测试工作计划

| 周次 | 时间 | 测试任务 | 交付物 | 配合开发 |
|------|------|----------|--------|----------|
| **第 1 周** | 3/19-3/25 | 测试环境搭建、单元测试用例编写 | Docker Compose、测试用例 50+ | 评审 API 设计 |
| **第 2 周** | 3/26-4/1 | 单元测试执行、集成测试用例编写 | 覆盖率报告、集成测试 20+ | 验收单元测试 |
| **第 3 周** | 4/2-4/8 | 集成测试执行、E2E 测试用例编写 | Bug 清单、E2E 测试 10+ | 配合联调 |
| **第 4 周** | 4/9-4/15 | 性能测试、Bug 回归 | 性能报告、测试总结 | 验证 Bug 修复 |
| **第 4 周末** | **4/16** | **🚀 内测版验收** | **测试通过报告** | **上线评审** |

---

## 💡 质量保障建议

### 1. 代码审查清单

**开发提交前自查**：
- [ ] 单元测试已添加/更新
- [ ] 覆盖率未下降（≥70%）
- [ ] 无 lint 错误（flake8/black）
- [ ] 类型检查通过（mypy）
- [ ] 变更已自测

**QA 审查重点**：
- [ ] 边界条件处理
- [ ] 异常场景覆盖
- [ ] 日志完整性
- [ ] 安全漏洞（SQL 注入、XSS）
- [ ] 性能影响

### 2. Bug 管理流程

```
发现 Bug → 提交 Issue → 优先级评估 → 分配负责人 → 修复 → 测试验证 → 关闭
```

**优先级定义**：
- **P0**：阻塞性 Bug（立即修复，2 小时内）
- **P1**：严重 Bug（24 小时内）
- **P2**：一般 Bug（本周内）
- **P3**：轻微 Bug（排期修复）

### 3. 自动化测试门禁

**GitHub Actions 检查项**：
```yaml
# .github/workflows/quality-gate.yml
name: Quality Gate

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run linters
        run: |
          flake8 app/
          black --check app/
          mypy app/
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: pytest --cov=app --cov-fail-under=70
  
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run security scan
        run: bandit -r app/
```

---

## 我的疑问

@允灿：
1. **Whisper 模型切换**：模型热切换时，是否需要清空 GPU 缓存？（避免 OOM）
2. **Celery 任务重试**：重试机制是指数退避还是固定间隔？（影响 B 站 API 限流）
3. **日志级别**：生产环境日志级别设为 INFO 还是 WARNING？（平衡调试和性能）

@美娜：
1. **测试服务器预算**：性能测试需要独立环境，是否需要单独申请服务器？（预估 ¥300-500/月）
2. **Bug 优先级决策**：P0/P1 Bug 的优先级由谁最终决定？（产品还是技术）

@少平：
1. **前端 Mock 数据**：前端是否需要 Mock 服务器？（便于前后端并行开发）
2. **E2E 测试配合**：E2E 测试需要前端页面稳定，什么时候可以提供测试环境？

@易达：
1. **验收标准**：内测版的验收标准是什么？（功能完整性、性能指标、Bug 数量）
2. **用户反馈渠道**：内测用户反馈如何收集？（飞书群、问卷、Issue 系统）

---

## 🎯 下一步行动

**本周内完成**：
- [ ] 评审并优化 docker-compose.test.yml
- [ ] 准备测试直播源 URL
- [ ] 编写单元测试用例（50+）
- [ ] 搭建性能测试环境

**需要协调**：
- [ ] 申请测试服务器资源
- [ ] 确认 B 站 API 测试账号
- [ ] 安排 API 接口评审会议

---

以上就是我的测试方案！质量保证我来把关，大家放心冲！💪


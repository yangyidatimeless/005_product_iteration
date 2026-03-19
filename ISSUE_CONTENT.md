
### 发言 #9 - 少锋
**时间**: 2026-03-19 00:28 UTC

**我的见解**：

感谢允灿和少平的详细方案！两位的技术设计非常专业，我来从 QA 角度回应问题并补充测试策略：

---

## 📋 回应允灿的测试问题

### 1. 单元测试覆盖率 80% 标准
**决策：整体覆盖率 80%+，核心模块 90%+**

| 模块 | 覆盖率要求 | 理由 |
|------|------------|------|
| 录制模块 | 90%+ | 核心功能，涉及文件 IO 和网络，易出错 |
| 转录模块 | 90%+ | 核心功能，依赖外部模型，需充分测试边界情况 |
| 切片模块 | 90%+ | 核心功能，涉及 FFmpeg 调用和视频处理 |
| 上传模块 | 85%+ | 核心功能，涉及 API 调用和错误处理 |
| 用户/配额模块 | 85%+ | 涉及权限控制和计费逻辑 |
| 工具函数 | 70%+ | 辅助功能，风险较低 |

**执行策略**：
- CI 中设置覆盖率门槛：整体 80%，核心模块单独检查
- 覆盖率报告用 `coverage report --fail-under=80`
- 核心模块在 `pytest.ini` 中单独配置：`--fail-under=90`

### 2. E2E 测试：真实账号 vs Mock
**决策：混合策略**

| 测试类型 | 数据源 | 执行频率 | 说明 |
|----------|--------|----------|------|
| 日常 CI | Mock | 每次 commit | 快速反馈，不依赖外部服务 |
|  nightly build | 真实账号 | 每晚 2:00 UTC | 验证真实流程，发现集成问题 |
| 发布前 | 真实账号 | 每次 release | 最终验收，确保上线质量 |

**真实账号测试注意事项**：
- 使用**测试专用账号**（允灿提前准备，不要用生产账号）
- B 站 API 限流处理：测试账号设置独立的 API key，避免影响开发
- 测试视频**不公开**：上传时设置为"仅自己可见"，测试完成后自动删除
- 限流保护：如果 API 返回 429，测试自动跳过，不阻塞 CI

**Mock 测试建议**：
```python
# tests/conftest.py - pytest fixtures
@pytest.fixture
def mock_whisper():
    with patch('services.transcription.WhisperTranscriptionService') as mock:
        mock.return_value.transcribe.return_value = "测试转录文本"
        yield mock

@pytest.fixture
def mock_bilibili():
    with patch('services.upload.BilibiliUploaderImpl') as mock:
        mock.return_value.upload.return_value = "BV_test123"
        yield mock

# tests/test_recorder.py
def test_recording_with_mock(mock_whisper, mock_bilibili):
    # 测试逻辑，不依赖真实服务
    ...
```

### 3. 测试用例文档
**决策：我负责编写，你配合评审**

**分工会这样安排**：
- **允灿**：负责服务端代码实现 + 单元测试（最了解代码逻辑）
- **少锋**：负责测试用例设计 + 集成/E2E 测试 + CI/CD 搭建（QA 视角）
- **评审**：测试用例写完后，允灿评审技术准确性，我评审测试覆盖度

**测试用例文档结构**：
```markdown
# 测试用例文档

## 录制模块
- TC-REC-001: 正常录制流程
- TC-REC-002: 网络异常处理
- TC-REC-003: 直播中断恢复
- ...

## 转录模块
- TC-TRX-001: 正常转录流程
- TC-TRX-002: 长音频处理
- TC-TRX-003: 背景噪音处理
- ...
```

**交付时间**：
- 第 1 周：完成录制、转录模块测试用例
- 第 2 周：完成切片、上传模块测试用例
- 第 3 周：完成集成测试用例 + E2E 测试脚本

---

## 📋 回应少平的测试问题

### 1. 前端单元测试粒度
**决策：组件级 + 函数级 双层测试**

| 测试层级 | 测试对象 | 工具 | 覆盖率目标 |
|----------|----------|------|------------|
| 函数级 | composables、工具函数 | Vitest | 80%+ |
| 组件级 | Vue 组件（渲染、交互） | Vue Test Utils + Vitest | 70%+ |
| 页面级 | 完整页面流程 | Playwright | 核心流程 100% |

**测试优先级**：
- **P0（必须测）**：核心业务逻辑（composables）、关键组件（任务创建、状态展示）
- **P1（建议测）**：UI 组件（按钮、表单）、工具函数
- **P2（可选）**：纯展示组件（布局、静态页面）

**测试示例**：
```typescript
// tests/composables/useTaskStream.test.ts
import { describe, it, expect, vi } from 'vitest'
import { useTaskStream } from '@/composables/useTaskStream'

describe('useTaskStream', () => {
  it('should update taskStatus on SSE message', async () => {
    // 测试 SSE 状态更新逻辑
    ...
  })
})

// tests/components/TaskCard.test.ts
import { mount } from '@vue/test-utils'
import TaskCard from '@/components/TaskCard.vue'

describe('TaskCard', () => {
  it('should display correct status label', () => {
    const wrapper = mount(TaskCard, {
      props: { status: 'recording' }
    })
    expect(wrapper.text()).toContain('录制中')
  })
})
```

### 2. E2E 测试工具：Playwright vs Cypress
**决策：Playwright**

| 对比项 | Playwright | Cypress | 选择理由 |
|--------|------------|---------|----------|
| 浏览器支持 | Chromium/Firefox/WebKit | 主要 Chromium | Playwright 覆盖更全 |
| 并行执行 | ✅ 原生支持 | ⚠️ 需额外配置 | Playwright CI 更快 |
| 多 Tab 支持 | ✅ 原生支持 | ❌ 不支持 | 测试多窗口场景 |
| 移动端测试 | ✅ 原生支持 | ⚠️ 有限支持 | 测试响应式布局 |
| 学习曲线 | 中等 | 较低 | 但团队更看重功能 |
| 社区生态 | 微软背书，增长快 | 成熟稳定 | 两者都不错 |

**Playwright 测试示例**：
```typescript
// tests/e2e/task-creation.spec.ts
import { test, expect } from '@playwright/test'

test('should create a new task successfully', async ({ page }) => {
  // 登录
  await page.goto('/login')
  await page.fill('[data-testid="email"]', 'test@example.com')
  await page.fill('[data-testid="password"]', 'password123')
  await page.click('[data-testid="login-btn"]')
  
  // 创建任务
  await page.click('[data-testid="new-task-btn"]')
  await page.fill('[data-testid="stream-url"]', 'https://live.douyin.com/123456')
  await page.fill('[data-testid="keyword-1"]', '精彩')
  await page.click('[data-testid="create-task-btn"]')
  
  // 验证结果
  await expect(page.locator('[data-testid="task-status"]')).toHaveText('录制中')
})
```

### 3. 前端代码审查标准
**决策：关注以下维度**

| 审查维度 | 检查项 | 工具 |
|----------|--------|------|
| **代码规范** | ESLint 规则、Prettier 格式化 | eslint + prettier |
| **类型安全** | TypeScript 类型定义、no any | tsc --noEmit |
| **性能** | 组件渲染次数、大列表虚拟化 | Vue Devtools |
| **可访问性** | 语义化 HTML、键盘导航、颜色对比度 | axe-core |
| **测试覆盖** | 单元测试、E2E 测试 | Vitest + Playwright |
| **安全性** | XSS 防护、CSP、敏感信息处理 | eslint-plugin-security |

**CI 检查项**：
```yaml
# .github/workflows/frontend-ci.yml
- name: Lint
  run: npm run lint
  
- name: Type Check
  run: npm run type-check
  
- name: Test
  run: npm run test:coverage
  
- name: Build
  run: npm run build
  
- name: E2E Test
  run: npx playwright test
```

---

## 🎯 补充：整体质量保障策略

结合允灿和少平的方案，我从 QA 角度补充整体质量保障体系：

### 质量门禁（Quality Gates）

| 阶段 | 门禁项 | 通过标准 | 执行时机 |
|------|--------|----------|----------|
| **开发中** | ESLint + Prettier | 0 错误 | 每次保存（IDE 插件） |
| **提交前** | 单元测试 | 通过率 100% | git commit hook |
| **PR 合并** | CI 流水线 | 全部通过 | GitHub Actions |
| **发布前** | E2E 测试 | 核心流程 100% 通过 | 手动触发 |
| **上线后** | 监控告警 | 错误率 < 1% | Prometheus + Grafana |

### 测试金字塔（资源分配）

```
          /\
         /  \        E2E 测试 (10%)
        /----\       少量、慢、覆盖核心流程
       /      \
      /--------\    集成测试 (20%)
     /          \   中等数量、中等速度
    /------------\
   /              \  单元测试 (70%)
  /----------------\ 大量、快、覆盖细节
```

### 缺陷预防策略

| 策略 | 实施方法 | 负责人 |
|------|----------|--------|
| **代码审查** | 所有 PR 必须经过至少 1 人 review | 全员 |
| **静态分析** | ESLint、TypeScript、bandit（Python） | CI 自动 |
| **依赖扫描** | GitHub Dependabot 自动检查漏洞 | CI 自动 |
| **测试驱动** | 核心功能先写测试再写实现 | 允灿 + 少锋 |
| **灰度发布** | 先让种子用户用 1 周，再全面开放 | 美娜决策 |

---

## 📅 更新后的测试计划

根据姐姐的里程碑和各位的方案，我更新测试计划：

| 周次 | 任务 | 交付物 | 负责人 |
|------|------|--------|--------|
| **第 1 周** | 服务端单元测试 + 测试用例设计 | 测试用例文档 v1、单元测试覆盖率 60%+ | 允灿 + 少锋 |
| **第 2 周** | 服务端集成测试 + CI/CD 搭建 | GitHub Actions 流水线、集成测试脚本 | 允灿 + 少锋 |
| **第 3 周** | 前端单元测试 + E2E 测试 | 前端测试脚本、Playwright E2E 用例 | 少平 + 少锋 |
| **第 4 周** | 全链路测试 + 性能测试 | 性能测试报告、Bug 修复清单 | 全员 |
| **第 5 周** | 种子用户内测 + 反馈迭代 | 用户反馈报告、稳定性修复 | 全员 |

---

## 我的疑问

@允灿 外甥：
1. **测试环境搭建**：你计划用 Docker Compose 统一管理测试环境吗？（数据库、Redis、Mock 服务）如果是，我可以帮忙写测试用的 docker-compose.test.yml
2. **性能测试工具**：服务端性能测试你有用过 locust 或 k6 吗？（我想设计并发录制场景的性能测试）
3. **日志收集**：结构化日志你计划输出到文件还是直接 stdout？（这会影响测试时的日志收集方式）

@少平 弟弟：
1. **前端 Mock 方案**：在允灿的 API 没好之前，你计划用 Mock Service Worker (MSW) 还是硬编码 Mock 数据？（我建议 MSW，可以复用允灿的 OpenAPI spec）
2. **视觉回归测试**：需不需要做 UI 截图对比测试？（用 Playwright 的 screenshot 功能，检测 UI 意外变更）
3. **多浏览器测试**：除了 Chrome，需要测试 Safari 和 Firefox 吗？（影响 Playwright 配置）

@美娜 姐姐：
1. **种子用户招募**：3-5 个种子用户你打算怎么招募？（需要我帮忙设计反馈问卷吗？）
2. **反馈收集方式**：用飞书群直接反馈，还是用专门的工单系统？（我建议前期用飞书群，快速响应）
3. **上线标准**：你希望看到哪些指标达标后才正式对外发布？（比如：连续 7 天无 P0 故障、用户满意度≥80% 等）

---

## 💡 补充建议

**关于测试数据管理**：
- 建议允灿准备一套**标准测试数据集**（测试直播源、测试音频文件、测试关键词）
- 测试数据放在 `tests/fixtures/` 目录，版本管理
- 敏感信息（API Key、账号密码）用 GitHub Secrets 管理

**关于 Bug 管理流程**：
```
发现 Bug → 提交 Issue（GitHub） → 美娜评估优先级 → 分配负责人 → 修复 → 测试验证 → 关闭
```

**Bug 优先级定义**：
- **P0**：阻塞发布（如：录制失败、上传失败）→ 24h 内修复
- **P1**：影响核心功能（如：切片不准确）→ 3 天内修复
- **P2**：体验问题（如：UI 错位、文案错误）→ 1 周内修复
- **P3**：优化建议（如：性能优化、代码重构）→ 后续迭代

---

以上就是我的测试方案！期待大家的反馈！🧪

---

### 发言 #10 - 允灿
**时间**: 2026-03-19 00:34 UTC

**我的见解**：

感谢少锋舅舅详细的测试方案！非常专业和系统，我来回应你的问题并补充服务端测试的实现细节：

---

## 📋 回应少锋舅舅的测试问题

### 1. 测试环境搭建：Docker Compose 统一管理

**决策：是的，用 Docker Compose 统一管理测试环境**

我计划创建两个 Compose 文件：
- `docker-compose.yml`：开发环境（持久化数据、便于调试）
- `docker-compose.test.yml`：测试环境（临时数据、隔离干净）

**docker-compose.test.yml 示例**：
```yaml
version: '3.8'
services:
  # 被测服务
  app:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - ENV=testing
      - DATABASE_URL=postgresql://test:test@db:5432/test_db
      - REDIS_URL=redis://redis:6379/0
      - WHISPER_MODEL=tiny  # 测试用轻量模型
    depends_on:
      - db
      - redis
    volumes:
      - ./tests/fixtures:/app/tests/fixtures
  
  # 测试数据库
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
      - POSTGRES_DB=test_db
    tmpfs:
      - /var/lib/postgresql/data  # 测试数据不持久化
  
  # 缓存服务
  redis:
    image: redis:7-alpine
    tmpfs:
      - /data  # 测试数据不持久化
  
  # Mock 服务（可选）
  mock-bilibili:
    image: mockserver/mockserver:latest
    ports:
      - "1080:1080"
    volumes:
      - ./tests/mocks/bilibili:/config
```

**测试执行脚本**：
```bash
# scripts/run-tests.sh
#!/bin/bash
set -e

echo "🚀 启动测试环境..."
docker-compose -f docker-compose.test.yml up -d

echo "⏳ 等待服务就绪..."
sleep 10

echo "🧪 运行单元测试..."
docker-compose -f docker-compose.test.yml run app pytest tests/unit --cov=app --cov-report=term-missing

echo "🔗 运行集成测试..."
docker-compose -f docker-compose.test.yml run app pytest tests/integration

echo "🛑 清理测试环境..."
docker-compose -f docker-compose.test.yml down

echo "✅ 测试完成！"
```

**少锋舅舅如果需要帮忙写 docker-compose.test.yml，非常感谢！我可以先写初稿，你来评审优化。**

---

### 2. 性能测试工具：locust vs k6

**决策：用 locust 做服务端性能测试**

| 对比项 | locust | k6 | 选择理由 |
|--------|--------|-----|----------|
| 语言 | Python | JavaScript | 团队更熟悉 Python |
| 学习曲线 | 低 | 中等 | locust 更直观 |
| 分布式压测 | ✅ 原生支持 | ✅ 需配置 | 两者都支持 |
| Web UI | ✅ 实时可视化 | ⚠️ 需集成 | locust 更方便 |
| 协议支持 | HTTP/WebSocket | HTTP/GraphQL/gRPC | 我们主要用 HTTP |
| 社区生态 | 成熟 | 增长快 | 两者都不错 |

**locust 测试脚本示例**：
```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between
import random

class RecordingUser(HttpUser):
    wait_time = between(1, 3)  # 用户间隔 1-3 秒
    
    @task(3)
    def create_task(self):
        """创建录制任务"""
        self.client.post("/api/tasks", json={
            "stream_url": "https://live.douyin.com/" + str(random.randint(100000, 999999)),
            "keywords": ["精彩", "高能"]
        })
    
    @task(2)
    def check_task_status(self):
        """查询任务状态"""
        task_id = random.choice(["task_001", "task_002", "task_003"])
        self.client.get(f"/api/tasks/{task_id}")
    
    @task(1)
    def list_tasks(self):
        """任务列表"""
        self.client.get("/api/tasks?page=1&size=20")

# 执行命令：
# locust -f tests/performance/locustfile.py --host=http://localhost:8000
```

**性能测试场景设计**：
| 场景 | 并发用户 | 持续时间 | 目标 |
|------|----------|----------|------|
| 日常负载 | 10 用户 | 10 分钟 | 验证基础功能 |
| 峰值负载 | 50 用户 | 15 分钟 | 模拟高峰期 |
| 压力测试 | 100 用户 | 20 分钟 | 寻找系统瓶颈 |
| 稳定性测试 | 20 用户 | 2 小时 | 验证长时间运行 |

**性能指标目标**：
- API 响应时间 P95 < 500ms
- 错误率 < 1%
- CPU 使用率 < 80%
- 内存使用率 < 70%

---

### 3. 日志收集：结构化日志输出

**决策：stdout 输出 JSON 格式结构化日志**

**理由**：
- ✅ 符合 12-Factor App 原则（日志作为事件流）
- ✅ 便于 Docker/Kubernetes 收集（docker logs 直接捕获）
- ✅ 方便集成 ELK、Loki 等日志系统
- ✅ 测试时可通过 `docker-compose logs` 统一查看

**Python 日志配置示例**：
```python
# app/core/logging_config.py
import logging
import sys
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_entry, ensure_ascii=False)

def setup_logging():
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    
    logging.basicConfig(
        level=logging.INFO,
        handlers=[handler],
        force=True
    )
```

**测试时的日志收集**：
```bash
# 查看实时日志
docker-compose -f docker-compose.test.yml logs -f app

# 导出日志到文件
docker-compose -f docker-compose.test.yml logs app > test-logs.txt

# 只查看错误日志
docker-compose -f docker-compose.test.yml logs app | grep '"level": "ERROR"'
```

---

## 📋 补充：服务端测试实现计划

结合少锋舅舅的方案，我细化服务端测试的实现计划：

### 单元测试（第 1 周）

| 模块 | 测试文件 | 覆盖率目标 | 优先级 |
|------|----------|------------|--------|
| 录制模块 | `tests/unit/test_recorder.py` | 90%+ | P0 |
| 转录模块 | `tests/unit/test_transcription.py` | 90%+ | P0 |
| 切片模块 | `tests/unit/test_clipper.py` | 90%+ | P0 |
| 上传模块 | `tests/unit/test_uploader.py` | 85%+ | P0 |
| 用户模块 | `tests/unit/test_user.py` | 85%+ | P1 |
| 配额模块 | `tests/unit/test_quota.py` | 85%+ | P1 |

**单元测试示例**：
```python
# tests/unit/test_recorder.py
import pytest
from unittest.mock import patch, MagicMock
from app.services.recorder import LiveRecorder

class TestLiveRecorder:
    @pytest.fixture
    def recorder(self):
        return LiveRecorder(output_dir="/tmp/test_recordings")
    
    def test_start_recording_success(self, recorder):
        """测试正常启动录制"""
        with patch('app.services.recorder.subprocess.run') as mock_run:
            mock_run.return_value = MagicMock(returncode=0)
            result = recorder.start("https://live.douyin.com/123456")
            assert result.success is True
            assert result.task_id is not None
    
    def test_start_recording_invalid_url(self, recorder):
        """测试无效 URL 处理"""
        with pytest.raises(ValueError, match="Invalid stream URL"):
            recorder.start("not-a-valid-url")
    
    def test_stop_recording_success(self, recorder):
        """测试正常停止录制"""
        task_id = "test_task_001"
        with patch('app.services.recorder.os.path.exists', return_value=True):
            with patch('app.services.recorder.subprocess.run') as mock_run:
                result = recorder.stop(task_id)
                assert result.success is True
                assert result.file_path is not None
```

### 集成测试（第 2 周）

| 测试场景 | 测试文件 | 依赖服务 | 优先级 |
|----------|----------|----------|--------|
| 完整录制流程 | `tests/integration/test_recording_flow.py` | DB, Redis | P0 |
| 转录 + 切片流程 | `tests/integration/test_transcription_flow.py` | Whisper 模型 | P0 |
| 上传到 B 站 | `tests/integration/test_upload_flow.py` | B 站 API（Mock） | P0 |
| 用户配额限制 | `tests/integration/test_quota_limit.py` | DB, Redis | P1 |

**集成测试示例**：
```python
# tests/integration/test_recording_flow.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

@pytest.mark.integration
def test_full_recording_flow(db_session, redis_client):
    """测试完整录制流程：创建任务 → 录制 → 停止 → 转录 → 切片"""
    # 1. 创建任务
    response = client.post("/api/tasks", json={
        "stream_url": "https://live.douyin.com/test_stream",
        "keywords": ["测试", "关键词"]
    })
    assert response.status_code == 201
    task_id = response.json()["task_id"]
    
    # 2. 查询任务状态（应为录制中）
    response = client.get(f"/api/tasks/{task_id}")
    assert response.json()["status"] == "recording"
    
    # 3. 模拟录制完成（触发回调）
    # ... 模拟 FFmpeg 完成录制
    
    # 4. 验证转录结果
    # ... 等待转录完成
    
    # 5. 验证切片结果
    # ... 检查生成的视频片段
    
    # 6. 清理测试数据
    client.delete(f"/api/tasks/{task_id}")
```

---

## 📋 回应少平舅舅的问题（间接回应）

看到少锋舅舅转述了少平舅舅的一些测试问题，我也补充一下：

### 关于前端 Mock 方案

**建议：用 OpenAPI Spec + Mock Service Worker (MSW)**

**理由**：
- ✅ 前后端契约一致：基于同一份 OpenAPI spec 生成 Mock 数据
- ✅ 自动同步：后端 API 变更，前端 Mock 自动更新
- ✅ 复用性强：MSW 既可用于开发，也可用于测试

**实现方案**：
```bash
# 1. 后端生成 OpenAPI spec
# 在 FastAPI 中自动生成：http://localhost:8000/openapi.json

# 2. 前端用 openapi-typescript-codegen 生成类型定义
npx openapi-typescript-codegen --input http://localhost:8000/openapi.json --output src/api

# 3. 用 MSW 生成 Mock handler
npx msw init public/ --save
```

---

## 📅 更新后的服务端测试计划

| 时间 | 任务 | 交付物 | 状态 |
|------|------|--------|------|
| **第 1 周** (3/19-3/25) | 单元测试 + 测试用例设计 | 测试用例文档 v1、单元测试覆盖率 60%+ | 🔄 进行中 |
| **第 2 周** (3/26-4/1) | 集成测试 + CI/CD 搭建 | GitHub Actions 流水线、集成测试脚本 | ⏳ 待开始 |
| **第 3 周** (4/2-4/8) | 配合前端联调 + 性能测试 | 性能测试报告、Bug 修复 | ⏳ 待开始 |

---

## 我的疑问

@少锋 舅舅：
1. **测试用例文档模板**：你有现成的测试用例模板吗？（如果没有，我可以先起草一个）
2. **CI/CD 平台选择**：我们是用 GitHub Actions 还是 GitLab CI？（我倾向于 GitHub Actions，和代码仓库集成更好）
3. **测试报告格式**：你希望测试报告是 HTML、Markdown 还是直接集成到 PR 评论中？

@美娜 妈妈：
1. **测试资源预算**：性能测试可能需要额外服务器资源，有预算限制吗？（比如用 AWS EC2 按量付费）
2. **上线时间窗口**：你希望什么时候完成测试并上线？（这会影响测试计划的紧张程度）

@少平 舅舅：
1. **前后端联调时间**：你预计前端什么时候可以开始联调？（我好安排后端 API 冻结时间）
2. **API 文档偏好**：你喜欢 Swagger UI 还是 Redoc？（我可以配置相应的文档界面）

---

## 💡 补充建议

**关于测试数据管理**：
- 我会在 `tests/fixtures/` 目录下准备：
  - 测试直播源 URL 列表（安全的、可用于测试的公开直播）
  - 测试音频文件（不同长度、不同音质）
  - 测试关键词列表
  - Mock B 站 API 响应数据

**关于代码覆盖率报告**：
```bash
# 生成 HTML 报告
pytest --cov=app --cov-report=html

# 在浏览器查看
open htmlcov/index.html
```

**关于 CI/CD 集成**：
```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests with coverage
        run: pytest --cov=app --cov-report=xml
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
```

---

以上就是我的测试方案！期待和大家一起把质量做好！💪

---

### 发言 #11 - 美娜
**时间**: 2026-03-19 00:42 UTC

**我的见解**：

感谢少锋和允灿的详细测试方案！非常专业和系统化，我来从 PM 角度回应问题并做出决策：

---

## 📋 回应少锋的问题

### 1. 种子用户招募方案

**决策：多渠道招募 5-8 个种子用户（预留缓冲）**

| 渠道 | 目标人数 | 招募方式 | 筛选标准 |
|------|----------|----------|----------|
| **B 站 UP 主** | 3-4 人 | 私信联系中小 UP 主（粉丝 1-10 万） | 有直播切片需求、愿意反馈、活跃度高 |
| **抖音主播** | 2-3 人 | 通过 MCN 机构介绍 | 日均直播 2h+、有切片分发需求 |
| **内部推荐** | 1-2 人 | 团队成员推荐 | 信任度高、反馈质量好 |

**招募话术模板**：
```
你好！我们是 [团队名称]，正在开发一款「直播自动切片上传工具」，
可以自动录制直播→AI 转录→智能切片→一键上传 B 站。

现邀请 5-8 位种子用户免费内测（限 1 个月），权益包括：
✅ 免费使用全部功能（正式版 ¥99/月）
✅ 专属客服群，问题 1h 内响应
✅ 功能需求优先排期

如果你有兴趣，欢迎回复我，拉你进内测群！
```

**反馈问卷设计**：
- **少锋主动提出帮忙设计，非常感谢！** 我来协调，你负责设计问卷结构，我负责联系用户填写
- 问卷工具：用飞书问卷（团队已有，数据好整理）
- 反馈频率：每周一次简短问卷（5 分钟内完成）+ 随时群内反馈

---

### 2. 反馈收集方式

**决策：飞书群直接反馈 + 工单系统结合**

| 阶段 | 反馈方式 | 工具 | 负责人 |
|------|----------|------|--------|
| **内测期（1-4 周）** | 飞书群直接反馈 | 飞书群聊 | 美娜 |
| **公测期（5-8 周）** | 飞书群 + 简易工单 | 飞书多维表格 | 少锋 |
| **正式上线** | 工单系统 + 客服邮箱 | 待选型 | 客服团队 |

**飞书群反馈规则**：
- 群名：「切片工具内测反馈群」
- 群成员：5-8 个种子用户 + 我们 5 人团队
- 反馈格式：`【问题】/【建议】+ 描述 + 截图`
- 响应 SLA：P0 问题 1h 内响应，P1 问题 4h 内响应，P2 问题 24h 内响应

**Bug 流转流程**：
```
用户反馈（飞书群） → 美娜整理到多维表格 → 评估优先级 → 分配负责人 → 修复 → 测试验证 → 用户确认 → 关闭
```

---

### 3. 上线标准（质量门禁）

**决策：分阶段上线，每阶段有不同标准**

| 阶段 | 时间 | 上线标准 | 决策人 |
|------|------|----------|--------|
| **内测版** | 第 4 周末 | 核心功能可用（录制、转录、切片、上传都能跑通） | 美娜 |
| **公测版** | 第 8 周末 | 连续 7 天无 P0 故障、P1 问题<5 个、用户满意度≥80% | 美娜 + 易达 |
| **正式版** | 第 12 周末 | 连续 14 天无 P0 故障、P1 问题<3 个、用户满意度≥90%、付费转化率≥10% | 全员 |

**详细指标定义**：
| 指标 | 目标值 | 测量方式 |
|------|--------|----------|
| **稳定性** | 连续 7/14 天无 P0 故障 | GitHub Issue 统计 |
| **Bug 数量** | P1<5 个（公测）、P1<3 个（正式） | 未关闭的 P1 Issue 数量 |
| **用户满意度** | ≥80%（公测）、≥90%（正式） | 飞书问卷（1-5 分制，≥4 分为满意） |
| **付费转化率** | ≥10% | 付费用户数 / 活跃用户数 |
| **API 响应时间** | P95<500ms | Prometheus 监控 |
| **错误率** | <1% | 错误请求数 / 总请求数 |

**上线决策会议**：
- 内测版上线：第 4 周末，团队内部会议（30 分钟）
- 公测版上线：第 8 周末，团队内部会议 + 种子用户代表参与（1h）
- 正式版上线：第 12 周末，全员会议（2h）

---

## 📋 回应允灿的问题

### 1. 测试资源预算

**决策：预算 ¥2000/月，优先用云服务按需付费**

| 资源项 | 预算 | 使用场景 | 备注 |
|--------|------|----------|------|
| **AWS EC2** | ¥800/月 | 性能测试、E2E 测试 | 按量付费，测试完即停 |
| **B 站测试账号** | ¥0 | API 测试 | 允灿准备测试专用账号 |
| **域名/SSL** | ¥200/年 | 测试环境域名 | 用阿里云/腾讯云 |
| **监控服务** | ¥500/月 | Prometheus + Grafana 云托管 | 前期用免费 tier，不够再升级 |
| **备用金** | ¥500/月 | 应急使用 | 如需要额外压测资源 |

**成本控制原则**：
- ✅ 测试环境用完后立即关闭（避免闲置费用）
- ✅ 优先用免费 tier（AWS 免费 12 个月、GitHub Actions 免费 2000 分钟/月）
- ✅ 性能测试集中在夜间执行（电费低、网络空闲）

---

### 2. 上线时间窗口

**决策：分三阶段上线，具体时间如下**

| 阶段 | 时间窗口 | 目标 | 风险缓冲 |
|------|----------|------|----------|
| **内测版** | 4 月 16 日（第 4 周末） | 5-8 个种子用户开始使用 | 缓冲 1 周（最晚 4/23） |
| **公测版** | 5 月 14 日（第 8 周末） | 开放注册，100-200 用户 | 缓冲 1 周（最晚 5/21） |
| **正式版** | 6 月 4 日（第 12 周末） | 全面推广，商业化运营 | 缓冲 1 周（最晚 6/11） |

**关键里程碑**：
```
3/19 - 4/15:  开发 + 单元测试（第 1-4 周）
4/16:        内测版上线（第 4 周末）
4/17 - 5/13:  内测反馈迭代 + 集成测试（第 5-8 周）
5/14:        公测版上线（第 8 周末）
5/15 - 6/03:  公测反馈迭代 + 性能优化（第 9-12 周）
6/04:        正式版上线（第 12 周末）
```

**风险预案**：
- 如果第 4 周末无法完成内测版：优先保证核心功能（录制→转录→切片→上传），次要功能（用户管理、配额系统）延后
- 如果测试发现重大 Bug：启动「Bug 修复周」，全员暂停新功能开发，专注修复

---

## 📋 回应允灿关于前后端联调的问题（代少平回应）

看到允灿问了少平一些问题，我帮少平先初步回应（少平可以补充）：

### 前后端联调时间

**初步评估：第 3 周开始联调（4 月 2 日左右）**

| 周次 | 前端任务 | 联调需求 |
|------|----------|----------|
| **第 1 周** | 项目初始化、UI 框架搭建 | 无需联调 |
| **第 2 周** | 页面开发、组件实现 | 无需联调（用 Mock 数据） |
| **第 3 周** | API 对接、状态管理 | **开始联调**（需要后端 API） |
| **第 4 周** | 联调修复、性能优化 | 持续联调 |

**建议**：
- 允灿在**第 2 周末**提供第一版可用的 API（至少包含：创建任务、查询任务状态、任务列表）
- 少平在**第 2 周末**提供前端 Mock 数据（基于 OpenAPI spec）
- 第 3 周开始，每天下午 4 点固定 30 分钟联调同步会（线上）

---

### API 文档偏好

**决策：Swagger UI + Redoc 双文档**

| 文档类型 | 用途 | 地址 |
|----------|------|------|
| **Swagger UI** | 开发调试（可交互） | http://localhost:8000/docs |
| **Redoc** | 对外文档（美观、易读） | http://localhost:8000/redoc |

**理由**：
- Swagger UI：允灿开发时自测、少平联调时调试，非常方便
- Redoc：给种子用户看，界面更专业、易读

---

## 🎯 PM 视角的补充建议

### 1. 测试优先级调整建议

结合允灿和少锋的方案，我建议测试优先级这样安排：

| 优先级 | 测试类型 | 执行时间 | 负责人 |
|--------|----------|----------|--------|
| **P0（必须）** | 核心功能单元测试 | 第 1-2 周 | 允灿 + 少锋 |
| **P0（必须）** | 核心功能集成测试 | 第 2-3 周 | 允灿 + 少锋 |
| **P1（重要）** | E2E 测试（核心流程） | 第 3-4 周 | 少平 + 少锋 |
| **P1（重要）** | 性能测试 | 第 4 周 | 允灿 + 少锋 |
| **P2（可选）** | 边界情况测试 | 第 5-6 周 | 少锋 |
| **P2（可选）** | 视觉回归测试 | 第 5-6 周 | 少平 |

**原则**：先保证核心功能稳定，再追求覆盖率和边缘情况。

---

### 2. 风险管理

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| **B 站 API 变更** | 中 | 高 | 允灿每周检查一次 B 站 API 文档，提前适配 |
| **Whisper 转录速度慢** | 高 | 中 | 准备备选方案（商用 API 如 Azure Speech） |
| **种子用户招募困难** | 中 | 中 | 准备 Plan B：降低招募标准、延长招募时间 |
| **性能不达标** | 中 | 高 | 提前做性能测试，预留 2 周优化时间 |
| **团队成员生病/请假** | 低 | 高 | 关键模块双人熟悉（允灿 + 少锋互相 backup） |

---

### 3. 沟通机制

**决策：固定沟通节奏，减少临时打扰**

| 会议类型 | 时间 | 参与人 | 时长 | 内容 |
|----------|------|--------|------|------|
| **每日站会** | 每天 10:00 | 全员 | 15 分钟 | 昨日进展、今日计划、阻塞问题 |
| **联调同步会** | 第 3-4 周，每天 16:00 | 允灿 + 少平 | 30 分钟 | 前后端联调问题 |
| **周例会** | 每周五 15:00 | 全员 | 1h | 本周总结、下周计划、风险同步 |
| **测试评审会** | 第 2、4、8 周末 | 全员 | 1h | 测试结果评审、上线决策 |

**沟通工具**：
- 日常沟通：飞书群（现有项目群）
- 文档协作：飞书文档（测试用例、会议记录）
- Bug 跟踪：GitHub Issues
- 进度看板：飞书多维表格（任务状态、负责人、截止时间）

---

## 📊 更新后的整体计划

结合大家的方案，我更新了整体计划：

| 周次 | 时间 | 服务端（允灿） | 前端（少平） | 测试（少锋） | PM（美娜） |
|------|------|---------------|-------------|-------------|-----------|
| **第 1 周** | 3/19-3/25 | 单元测试 60%+ | 项目初始化 | 测试用例设计 | 种子用户招募 |
| **第 2 周** | 3/26-4/1 | 单元测试 100%、CI/CD 搭建 | 页面开发 | 单元测试评审 | 反馈问卷设计 |
| **第 3 周** | 4/2-4/8 | 集成测试、API 冻结 | 开始联调 | 集成测试 | 联调协调 |
| **第 4 周** | 4/9-4/15 | 性能测试、Bug 修复 | E2E 测试、Bug 修复 | E2E 测试 | 内测准备 |
| **第 4 周末** | **4/16** | **🚀 内测版上线** | **🚀 内测版上线** | **🚀 内测版上线** | **🚀 内测版上线** |

---

## 我的疑问

@允灿 儿子：
1. **Whisper 模型选择**：你计划用哪个 Whisper 模型？（tiny/base/small/medium/large）转录速度和准确率的权衡考虑是什么？
2. **FFmpeg 参数优化**：录制和切片时，FFmpeg 参数你有现成的优化方案吗？（比如码率、分辨率、关键帧间隔）

@少平 弟弟：
1. **前端技术栈确认**：最终确定用 Vue 3 + TypeScript + Vite 吗？有没有其他偏好？
2. **UI 组件库**：用 Element Plus、Ant Design Vue 还是其他？（这会影响开发效率）

@少锋 大弟弟：
1. **测试用例文档**：你什么时候能给出测试用例文档初稿？（我好安排评审时间）
2. **CI/CD 平台**：你倾向于 GitHub Actions 还是其他？（如果需要，我可以帮忙配置）

@易达 老公：
1. **商业化定价**：你对正式版定价有什么想法？（比如 ¥99/月、¥999/年、阶梯定价等）
2. **推广渠道**：正式上线后，你计划通过哪些渠道推广？（B 站、抖音、技术社区等）

---

## 💡 PM 总结

测试方案非常完善，我这边主要做以下决策：

✅ **种子用户**：5-8 人，多渠道招募，我负责联系  
✅ **反馈方式**：飞书群直接反馈 + 多维表格跟踪  
✅ **上线标准**：分三阶段，每阶段有明确指标  
✅ **预算**：¥2000/月，优先用云服务按需付费  
✅ **时间窗口**：4/16 内测、5/14 公测、6/4 正式  
✅ **沟通机制**：每日站会 + 周例会 + 专项评审会  

接下来大家按计划执行，有问题随时在群里同步！辛苦各位了！💪

---

---

### 发言 #12 - 易达
**时间**: 2026-03-19 00:47 UTC

**我的见解**：

感谢美娜、少锋、允灿的详细方案！作为产品经理，我从产品定位和商业化角度回应问题并补充建议：

---

## 📋 回应美娜的问题

### 1. 商业化定价策略

**决策：阶梯定价 + 早鸟优惠，降低用户决策门槛**

| 版本 | 价格 | 功能权益 | 目标用户 |
|------|------|----------|----------|
| **免费版** | ¥0/月 | 每月 5 小时录制时长、720P 画质、基础转录 | 个人用户、尝鲜用户 |
| **专业版** | ¥99/月 或 ¥999/年（省¥189） | 每月 50 小时、1080P 画质、智能切片、批量上传 | 中小 UP 主、兼职主播 |
| **企业版** | ¥499/月 或 ¥4999/年（省¥989） | 无限时长、4K 画质、自定义切片规则、API 接入、专属客服 | MCN 机构、直播团队 |

**定价逻辑**：
- **对标竞品**：市面上类似工具（如「剪映自动切片」）定价约 ¥128/月，我们有价格优势
- **成本核算**：Whisper 转录成本约 ¥0.5/小时，B 站 API 免费，服务器成本约 ¥20/用户/月
- **毛利空间**：专业版毛利率约 60%-70%，企业版约 80%+

**早鸟优惠（内测期专属）**：
- 前 100 名付费用户：专业版 ¥69/月（限首年）
- 内测种子用户：免费使用 1 个月，之后享 5 折优惠（¥49.5/月）

**付费转化策略**：
| 策略 | 实施方式 | 预期转化率 |
|------|----------|------------|
| **免费试用** | 注册用户送 7 天专业版体验 | 15%-20% |
| **功能限制** | 免费版带水印、无法批量上传 | 10%-15% |
| **时长限制** | 免费版每月 5 小时，超限需升级 | 8%-12% |
| **限时优惠** | 首月 5 折、年付 8 折 | 20%-25% |

---

### 2. 推广渠道规划

**决策：线上为主、线下为辅，分阶段投放**

| 阶段 | 时间 | 渠道 | 目标 | 预算 |
|------|------|------|------|------|
| **内测期** | 4/16-5/13 | B 站私信、MCN 推荐 | 5-8 种子用户 | ¥0 |
| **公测期** | 5/14-6/3 | B 站动态、UP 主互推 | 100-200 用户 | ¥5000 |
| **正式版** | 6/4 起 | B 站广告、抖音信息流、技术社区 | 1000+ 用户 | ¥20000/月 |

**详细渠道策略**：

#### B 站（主战场）
- **官方账号运营**：发布产品教程、用户案例、更新日志
- **UP 主合作**：找 10-15 个游戏/直播区 UP 主（粉丝 5-50 万），免费换推广
- **B 站广告**：信息流广告（CPM 约 ¥30-50），定向游戏/直播区用户
- **B 站专栏**：发布技术文章「如何用 AI 自动切片，1 小时搞定 10 小时直播」

#### 抖音/快手
- **短视频引流**：发布「1 分钟演示」短视频，挂 B 站链接
- **主播合作**：找游戏主播演示工具，直播间挂推广链接
- **信息流广告**：CPM 约 ¥20-40，定向直播相关兴趣人群

#### 技术社区
- **掘金/知乎**：发布技术文章，建立专业形象
- **GitHub**：开源部分工具（如 FFmpeg 封装库），引流到商业版
- **V2EX/少数派**：发布产品体验文，吸引早期采用者

#### 私域流量
- **微信群/QQ 群**：加入 UP 主群、直播交流群，软性推广
- **邮件营销**：收集意向用户邮箱，发送产品更新和优惠信息

**推广预算分配（正式版首月 ¥20000）**：
| 渠道 | 预算 | 预期获客 | 获客成本 |
|------|------|----------|----------|
| B 站广告 | ¥10000 | 300-400 人 | ¥25-33/人 |
| 抖音广告 | ¥5000 | 200-300 人 | ¥17-25/人 |
| UP 主合作 | ¥3000 | 100-150 人 | ¥20-30/人 |
| 内容营销 | ¥2000 | 50-100 人 | ¥20-40/人 |

---

## 🎯 产品定位补充

结合大家的讨论，我明确一下产品定位：

### 目标用户画像

| 用户类型 | 特征 | 需求痛点 | 付费意愿 |
|----------|------|----------|----------|
| **游戏主播** | 日均直播 2-4h，B 站/抖音双平台 | 手动切片耗时、错过热点 | 高（¥100-300/月） |
| **带货主播** | 日均直播 4-6h，多平台分发 | 需要快速产出短视频引流 | 中（¥50-150/月） |
| **MCN 机构** | 管理 10-50 个主播，批量运营 | 人力成本高、效率低 | 高（¥500-2000/月） |
| **个人 UP 主** | 周直播 1-2 次，兼职 | 预算有限、功能够用即可 | 低（¥0-50/月） |

### 核心价值主张

**一句话介绍**：
> 「直播自动切片工具」— 1 小时直播，10 分钟搞定切片分发

**核心价值**：
- ⏱️ **节省时间**：从 3-4 小时手动切片 → 10 分钟自动完成
- 💰 **降低成本**：从 ¥3000-5000/月 剪辑师人力 → ¥99/月 工具成本
- 📈 **提升效率**：从 1 天发布 1-2 个切片 → 1 天发布 10-20 个切片
- 🎯 **抓住热点**：直播结束 30 分钟内发布切片，蹭热点流量

### 竞品分析

| 竞品 | 价格 | 优势 | 劣势 | 我们的差异化 |
|------|------|------|------|-------------|
| **剪映自动切片** | ¥128/月 | 品牌知名度高、功能全 | 切片规则固定、不够智能 | AI 语义理解、自定义规则 |
| **快影切片助手** | ¥88/月 | 价格便宜、快手生态 | 仅支持快手、功能单一 | 多平台支持、B 站深度集成 |
| **手动剪辑** | ¥3000-5000/月（人力） | 灵活、质量可控 | 耗时、成本高、依赖人工 | 自动化、成本低、7x24 运行 |

---

## 📋 回应允灿的技术问题（间接回应美娜）

### Whisper 模型选择建议

**决策：默认用 `base` 模型，提供切换选项**

| 模型 | 显存占用 | 转录速度 | 准确率 | 适用场景 |
|------|----------|----------|--------|----------|
| **tiny** | ~1GB | 最快（约 10x 实时） | 70%-75% | 测试、对准确率要求低 |
| **base** | ~1GB | 快（约 8x 实时） | 80%-85% | **默认推荐** |
| **small** | ~2GB | 中等（约 4x 实时） | 85%-90% | 对准确率有要求 |
| **medium** | ~5GB | 慢（约 2x 实时） | 90%-95% | 专业场景、付费用户 |
| **large** | ~10GB | 很慢（约 1x 实时） | 95%+ | 不推荐（性价比低） |

**建议配置**：
- 免费版：`base` 模型（平衡速度和准确率）
- 专业版：`small` 模型（更高准确率）
- 企业版：`medium` 模型 + 可自定义

**成本核算**（按 base 模型计）：
- 转录 1 小时音频：约 7-8 分钟 GPU 时间
- GPU 成本（AWS g4dn.xlarge）：约 $0.526/小时 ≈ ¥3.8/小时
- 1000 个付费用户 × 50 小时/月 × ¥3.8/小时 = ¥190,000/月 转录成本
- 考虑规模效应和批量处理，实际成本可控制在 ¥100,000-150,000/月

---

## 📋 回应少平的前端问题

### 前端技术栈确认

**决策：Vue 3 + TypeScript + Vite + Element Plus**

| 技术选型 | 版本 | 理由 |
|----------|------|------|
| **框架** | Vue 3.4+ | 团队熟悉、生态成熟、性能好 |
| **语言** | TypeScript 5.3+ | 类型安全、减少 Bug、IDE 支持好 |
| **构建工具** | Vite 5.0+ | 开发启动快（秒开）、HMR 快、打包优化好 |
| **UI 组件库** | Element Plus 2.4+ | 中文文档、组件全、社区活跃 |
| **状态管理** | Pinia 2.1+ | Vuex 官方继任者、更简洁、TypeScript 友好 |
| **HTTP 客户端** | Axios 1.6+ | 成熟稳定、拦截器方便 |
| **CSS 方案** | Tailwind CSS 3.4+ | 原子化 CSS、开发效率高 |

**理由**：
- ✅ 少平熟悉 Vue 生态（之前项目用过）
- ✅ Element Plus 组件全，快速搭建后台界面
- ✅ TypeScript 减少前后端联调时的类型错误
- ✅ Vite 开发体验好，编译速度快

---

## 📋 回应少锋的测试问题

### CI/CD 平台选择

**决策：GitHub Actions**

| 对比项 | GitHub Actions | GitLab CI | Jenkins |
|--------|---------------|-----------|---------|
| **集成度** | ✅ 与代码仓库无缝集成 | ⚠️ 需自建 GitLab | ❌ 需单独部署 |
| **免费额度** | ✅ 2000 分钟/月 | ✅ 400 分钟/月 | ❌ 自建成本 |
| **学习曲线** | ✅ 低（YAML 配置） | ✅ 低 | ❌ 高 |
| **社区生态** | ✅ 丰富（大量 Action） | ⚠️ 一般 | ✅ 丰富 |
| **成本** | ✅ 免费 tier 够用 | ✅ 免费 | ❌ 服务器成本 |

**GitHub Actions 配置建议**：
```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install backend dependencies
        run: pip install -r requirements.txt
      
      - name: Install frontend dependencies
        run: cd frontend && npm ci
      
      - name: Run backend tests
        run: pytest --cov=app --cov-report=xml
      
      - name: Run frontend tests
        run: cd frontend && npm run test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

## 🎯 产品功能优先级

结合美娜的时间节点，我明确功能优先级：

### 内测版（4/16）功能清单

| 模块 | 功能 | 优先级 | 负责人 |
|------|------|--------|--------|
| **录制** | 支持抖音直播录制、断线重连 | P0 | 允灿 |
| **转录** | Whisper 转录、生成字幕文件 | P0 | 允灿 |
| **切片** | 基于关键词自动切片、FFmpeg 处理 | P0 | 允灿 |
| **上传** | B 站上传、设置标题/标签 | P0 | 允灿 |
| **前端** | 任务创建、状态展示、结果查看 | P0 | 少平 |
| **测试** | 核心功能测试用例、CI/CD 搭建 | P0 | 少锋 |
| **用户** | 简单登录（邮箱验证码）、配额显示 | P1 | 允灿 |

### 公测版（5/14）新增功能

| 模块 | 功能 | 优先级 | 负责人 |
|------|------|--------|--------|
| **用户** | 完整用户系统、配额管理、付费入口 | P0 | 允灿 |
| **切片** | 智能切片（基于语义、热度检测） | P0 | 允灿 |
| **前端** | 数据看板、任务历史、设置页面 | P0 | 少平 |
| **测试** | E2E 测试、性能测试、安全测试 | P0 | 少锋 |
| **运营** | 邀请码系统、优惠券、推广链接 | P1 | 允灿 |

### 正式版（6/4）新增功能

| 模块 | 功能 | 优先级 | 负责人 |
|------|------|--------|--------|
| **支付** | 微信支付、支付宝、发票 | P0 | 允灿 |
| **企业版** | API 接入、自定义切片规则、白标 | P0 | 允灿 |
| **前端** | 多账号管理、团队协作 | P1 | 少平 |
| **监控** | 完整监控体系、告警系统 | P0 | 允灿 |
| **客服** | 工单系统、帮助中心、FAQ | P1 | 美娜 |

---

## 📅 产品里程碑

结合美娜的时间表，我细化产品里程碑：

```
📋 3/19-3/25（第 1 周）
├─ 需求评审完成（美娜）
├─ 技术方案评审完成（允灿 + 少平）
├─ 测试用例初稿完成（少锋）
└─ 种子用户招募启动（美娜）

📋 3/26-4/1（第 2 周）
├─ 服务端单元测试 60%+（允灿）
├─ 前端项目初始化完成（少平）
├─ CI/CD 流水线搭建完成（少锋）
└─ 反馈问卷设计完成（美娜 + 少锋）

📋 4/2-4/8（第 3 周）
├─ 前后端开始联调（允灿 + 少平）
├─ 集成测试启动（少锋）
├─ 种子用户确定（5-8 人）（美娜）
└─ 产品 Demo 完成（全员）

📋 4/9-4/15（第 4 周）
├─ 性能测试完成（允灿 + 少锋）
├─ E2E 测试完成（少平 + 少锋）
├─ Bug 修复冲刺（全员）
└─ 🚀 4/16 内测版上线（全员）

📋 4/17-5/13（第 5-8 周）
├─ 内测反馈收集（美娜）
├─ 每周迭代 1-2 个版本（允灿 + 少平）
├─ 用户满意度≥80%（美娜）
└─ 🚀 5/14 公测版上线（全员）

📋 5/15-6/3（第 9-12 周）
├─ 公测反馈收集（美娜）
├─ 性能优化、稳定性提升（允灿）
├─ 支付系统接入（允灿）
└─ 🚀 6/4 正式版上线（全员）
```

---

## 💡 商业化建议

### 收入预测（保守估计）

| 时间 | 用户数 | 付费转化率 | 付费用户 | ARPU | 月收入 |
|------|--------|------------|----------|------|--------|
| **内测期**（4/16-5/13） | 5-8 人 | 0%（免费） | 0 | ¥0 | ¥0 |
| **公测期**（5/14-6/3） | 100-200 人 | 10% | 10-20 | ¥99 | ¥990-1980 |
| **正式版首月**（6/4-7/3） | 500-800 人 | 12% | 60-96 | ¥99 | ¥5940-9504 |
| **正式版 3 月后** | 1000-2000 人 | 15% | 150-300 | ¥99 | ¥14850-29700 |

**成本结构（月）**：
| 成本项 | 金额 | 说明 |
|--------|------|------|
| 服务器（EC2 + RDS） | ¥5000-8000 | 随用户量增长 |
| GPU（转录） | ¥10000-20000 | 主要成本 |
| 带宽/CDN | ¥2000-5000 | 视频上传/下载 |
| 推广费用 | ¥20000 | 正式版投入 |
| 人力成本 | ¥0（家庭团队） | 暂不计入 |
| **总成本** | ¥37000-53000 | |

**盈亏平衡点**：
- 需要约 400-500 个付费用户（按 ¥99/月计）
- 预计正式版上线后 3-6 个月达到

---

## 我的疑问

@美娜：
1. **种子用户筛选标准**：你倾向于找头部 UP 主（粉丝 50 万+，影响力大但难合作）还是中小 UP 主（粉丝 1-10 万，更容易合作）？
2. **用户协议/隐私政策**：需要找法务审核吗？还是先用模板？（涉及用户数据、付费条款）

@允灿：
1. **B 站 API 限制**：B 站对上传频率、视频时长有限制吗？需要提前准备什么资质？
2. **转录准确率兜底**：如果 Whisper 转录准确率低于 80%，有什么兜底方案？（比如人工校对入口）

@少平：
1. **前端上线部署**：你计划用 Vercel、Netlify 还是自建服务器？（影响部署流程）
2. **移动端适配**：需要做移动端 H5 吗？还是先聚焦 PC 端？

@少锋：
1. **测试账号准备**：B 站测试账号、抖音测试直播源，需要我帮忙协调资源吗？
2. **Bug 管理流程**：你建议用 GitHub Projects 还是飞书多维表格管理 Bug 跟踪？

---

## 🎯 总结

作为产品经理，我明确以下几点：

✅ **产品定位**：直播自动切片工具，主打「省时、省钱、高效」  
✅ **目标用户**：游戏主播、带货主播、MCN 机构、个人 UP 主  
✅ **定价策略**：阶梯定价（免费/专业版/企业版），早鸟优惠  
✅ **推广渠道**：B 站为主，抖音/技术社区为辅  
✅ **上线时间**：4/16 内测、5/14 公测、6/4 正式  
✅ **收入目标**：正式版 3 月后达到 150-300 付费用户，月收入 ¥1.5-3 万  

接下来大家按计划执行，有问题随时在群里同步！辛苦各位了！💪

---


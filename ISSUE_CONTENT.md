
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

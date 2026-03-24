---
name: flutter-state-screenshot
description: >
  Flutter 功能模块全状态截图生成工具。当用户说"帮我把xx功能全部状态列出来"、"截图xx页面所有状态"、"帮我看看xx模块的状态"、"把xx的状态矩阵输出给我"时触发。
  通过真实 iOS 模拟器截图 + HTML 汇总，自动梳理指定模块的所有 UI 状态组合（订阅/非订阅、加载中/成功/失败/空态、有网/无网、弹窗等），输出带状态文案标注的可视化 HTML 文档。
  适用于：功能开发完成后的视觉走查、设计评审、QA 测试用例对照。
---

# Flutter 全状态截图生成

## 核心原则

> **只做截图，不做分析。** 输出物只有：模拟器真实截图 + 每张截图的状态标注。
> 不要在 HTML 中输出任何额外数据面板（如 API 返回数据、账号分析、指标卡片等）。
> HTML 唯一展示内容 = 手机外框套真实截图 + 简短状态标签。

## 环境要求

- macOS（需要 Xcode 和 iOS 模拟器）
- Flutter SDK
- Cursor IDE

## 工作流程

### Phase 1: 分析目标模块状态维度

1. 确认用户指定的功能模块（如"首页"、"模板库"、"设置页"等）
2. 读取目标页面的核心 dart 文件，梳理所有状态变量：
   - 布尔开关（如 `_isLoading`、`_isVip`、`_isConnected`）
   - 枚举/状态机（如 loading/success/error/empty）
   - 条件分支（如 `Platform.isAndroid`、`isVip()`）
   - 弹窗覆盖层（Dialog、BottomSheet、Toast）
3. 读取关联文件：状态管理（Provider/GetX/Bloc）、订阅管理器、网络层
4. 输出完整的**状态维度清单**（每个维度有哪些可能值、当前代码是否可达）

### Phase 2: 准备模拟器环境

1. 检查 iOS 模拟器是否已启动：
   ```bash
   xcrun simctl list devices booted
   ```
2. 检查 Flutter app 是否在运行：
   ```bash
   ps aux | grep "flutter.*run"
   ```
3. 如未运行，启动模拟器和 Flutter app：
   ```bash
   open -a Simulator
   xcrun simctl boot <DEVICE_ID>
   cd <PROJECT_ROOT> && flutter run -d <DEVICE_ID>
   ```
4. 等待 app 完全加载（检查 terminal 输出中出现 "Flutter run key commands"）
5. 在 Skill 所在目录下创建截图输出目录：
   ```bash
   mkdir -p <SKILL_DIR>/output/<模块名>/screenshots/
   ```
   > `<SKILL_DIR>` 为本 SKILL.md 所在文件夹的绝对路径，由 AI 自动识别。

### Phase 3: 逐状态截图

对每个可达的状态组合：

1. **切换状态**（按优先级排列）：

   **方式 A — 模拟器 UI 操作**（优先，无需重启 app）：
   ```bash
   # 获取模拟器窗口位置
   osascript -e 'tell application "System Events"
       tell process "Simulator"
           set winPos to position of window 1
           set winSize to size of window 1
           return {winPos, winSize}
       end tell
   end tell'
   
   # Swift CGEvent 模拟点击
   swift -e '
   import Foundation; import CoreGraphics
   func click(x: CGFloat, y: CGFloat) {
       let p = CGPoint(x: x, y: y)
       CGEvent(mouseEventSource: nil, mouseType: .leftMouseDown, mouseCursorPosition: p, mouseButton: .left)!.post(tap: .cghidEventTap)
       Thread.sleep(forTimeInterval: 0.1)
       CGEvent(mouseEventSource: nil, mouseType: .leftMouseUp, mouseCursorPosition: p, mouseButton: .left)!.post(tap: .cghidEventTap)
   }
   click(x: <X>, y: <Y>)
   '
   
   # Swift CGEvent 模拟滑动
   swift -e '
   import Foundation; import CoreGraphics
   func drag(sx: CGFloat, sy: CGFloat, ey: CGFloat) {
       let sp = CGPoint(x: sx, y: sy)
       CGEvent(mouseEventSource: nil, mouseType: .leftMouseDown, mouseCursorPosition: sp, mouseButton: .left)!.post(tap: .cghidEventTap)
       Thread.sleep(forTimeInterval: 0.05)
       let steps = 30
       for i in 1...steps {
           let y = sy + (ey - sy) * CGFloat(i) / CGFloat(steps)
           CGEvent(mouseEventSource: nil, mouseType: .leftMouseDragged, mouseCursorPosition: CGPoint(x: sx, y: y), mouseButton: .left)!.post(tap: .cghidEventTap)
           Thread.sleep(forTimeInterval: 0.015)
       }
       CGEvent(mouseEventSource: nil, mouseType: .leftMouseUp, mouseCursorPosition: CGPoint(x: sx, y: ey), mouseButton: .left)!.post(tap: .cghidEventTap)
   }
   drag(sx: <X>, sy: <START_Y>, ey: <END_Y>)
   '
   ```

   **方式 B — 临时改代码**（适用于无法通过 UI 触达的状态）：
   修改状态变量初始值 → terminate app → 重新 flutter run → 截图 → 还原代码

   **方式 C — 修改 SharedPreferences**：
   ```bash
   APP_DATA=$(xcrun simctl get_app_container <DEVICE_ID> <BUNDLE_ID> data)
   PLIST="$APP_DATA/Library/Preferences/<BUNDLE_ID>.plist"
   defaults write "$PLIST" "flutter.<KEY>" -bool true
   xcrun simctl terminate <DEVICE_ID> <BUNDLE_ID>
   ```

2. **处理系统弹窗**：
   ```bash
   # 清空剪贴板避免 iOS 粘贴权限弹窗
   pbcopy < /dev/null
   ```
   > iOS 模拟器中 `Clipboard.getData()` 会触发系统粘贴权限弹窗。如果项目中有自动剪贴板检测逻辑，建议在 debug 模式下跳过，或在截图前清空剪贴板。

3. **截图**：
   ```bash
   xcrun simctl io <DEVICE_ID> screenshot <OUTPUT_PATH>.png
   ```

4. **验证截图**：用 Read 工具查看截图内容，确认状态正确、没有弹窗遮挡

5. **滚动截取长页面**：如果页面内容超出一屏，使用 Swift CGEvent 滑动后多次截图

6. **命名规范**：`<序号>_<模块>_<状态描述>.png`，如 `01_plan_disconnected.png`

### Phase 4: 生成 HTML 文档

将所有截图整合为一个 HTML 文件，输出到 `<SKILL_DIR>/output/<模块名>/` 目录下：
```
output/<模块名>/
├── <模块名>_states.html
└── screenshots/
    ├── 01_xxx.png
    └── ...
```
**不要把截图和 HTML 放到项目源码目录中。**

**HTML 输出的严格约束：**

- **只展示截图和状态标注**，不要添加任何 API 数据面板、账号信息卡片、指标分析等额外内容
- 每张截图套一个 **CSS 手机外框**（圆角 + 边框模拟 iPhone），不要有多余的黑边
- 截图宽度统一，外框使用纯 CSS 实现（无需外部图片）
- 每个状态维度一个 section，卡片横向排列
- 每张截图只配有：**状态名称**（简短标题）+ **触发条件**（代码级，如 `isVip() == true`）
- 底部包含状态维度覆盖总览表
- 截图使用相对路径引用（`screenshots/xxx.png`），HTML 和 screenshots 文件夹同级
- 深色背景（#0a0a0b）

**手机外框 CSS 参考样式：**
```css
.phone-frame {
  width: 280px;
  border: 3px solid #333;
  border-radius: 36px;
  overflow: hidden;
  background: #000;
  box-shadow: 0 0 0 2px #1a1a1a, 0 8px 32px rgba(0,0,0,0.5);
  flex-shrink: 0;
}
.phone-frame img {
  width: 100%;
  display: block;
}
```

### Phase 5: 展示结果

1. 用 `open <html_path>` 在浏览器中打开
2. 向用户汇报：总共多少个维度、多少种状态、多少张截图、哪些状态当前代码不可达

## 交互链路全覆盖原则

> **每次截图一个功能时，必须覆盖该功能完整交互链路中涉及的所有细节状态。**
> 
> 不仅截静态页面，还要截取用户操作过程中经过的每个中间态、反馈态和分支路径。

### 链路拆解方法

1. **入口状态**：功能区域的初始呈现（空输入框、默认占位符）
2. **输入状态**：用户输入/粘贴内容后的变化
3. **操作触发**：点击按钮后的即时响应
4. **加载中间态**：进度指示器、骨架屏、百分比进度
5. **成功结果态**：数据返回后的展示
6. **操作反馈态**：Copy、刷新、展开/折叠等二级操作的视觉反馈
7. **弹窗/浮层态**：BottomSheet、Dialog、Toast 等覆盖层
8. **失败/异常态**：网络错误、超时、服务端错误的 UI 反馈
9. **权限拦截态**：非订阅/未登录用户触发受限功能时的引导
10. **外部唤起态**：从其他页面/深链接跳入该功能时的表现

### 截图优先级

- 如果状态数量过多（>15张），先列出完整状态清单让用户确认
- 优先截图**本次改动点**涉及的状态
- 其次截图**核心主流程**（正常操作链路）
- 最后截图**边缘 case**（错误态、权限拦截等）

## 状态梳理 Checklist

分析页面时，务必检查以下维度：

| 维度 | 检查点 |
|------|--------|
| 认证/订阅 | 已登录/未登录、VIP/非VIP、试用期/过期 |
| 网络状态 | 请求成功/失败/超时 → Toast/全页错误/重试 |
| 加载状态 | loading indicator / skeleton / shimmer |
| 数据状态 | 有数据 / 空态 / 首次加载 |
| 平台差异 | iOS vs Android 的条件分支 |
| 账号绑定 | 已绑定/未绑定第三方账号 |
| 弹窗覆盖 | Dialog / BottomSheet / Toast / Snackbar |
| 权限状态 | 通知权限、相册权限、剪贴板权限等 |
| 营销活动 | 限时折扣、倒计时、促销 banner |
| 引导状态 | 新手引导、功能引导（首次 vs 非首次） |
| 操作反馈 | Copy/Paste/刷新/展开折叠等交互响应 |
| 外部唤起 | 深链接/剪贴板检测跳入功能的表现 |

## 注意事项

- **iOS 粘贴权限弹窗**：iOS 16+ 中 `Clipboard.getData()` 会触发系统级弹窗。截图前用 `pbcopy < /dev/null` 清空剪贴板；如项目有自动剪贴板检测逻辑，建议在 debug 模式下加开关跳过。
- 修改代码截图后**务必还原**（先 `git stash` 保存当前改动，截图后 `git checkout -- <files>` 还原，再 `git stash pop`）
- 如果某些状态需要后端配合才能触发（如折扣倒计时），在 HTML 中标注为"需后端配合"
- 对于代码中存在但当前不可达的状态（如写死的 Mock 数据），标注为"代码预留"
- **不要在 HTML 中添加任何数据分析面板**，输出物只有截图 + 状态标注
- 项目的 Bundle ID 从 `ios/Runner.xcodeproj/project.pbxproj` 中的 `PRODUCT_BUNDLE_IDENTIFIER` 获取

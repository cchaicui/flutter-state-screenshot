---
name: flutter-state-screenshot
description: >
  Flutter 功能模块全状态截图生成工具。当用户说"帮我把xx功能全部状态列出来"、"截图xx页面所有状态"、"帮我看看xx模块的状态"、"把xx的状态矩阵输出给我"、"帮我截图xx的极端情况"、"截图xx的边界case"时触发。
  通过真实 iOS 模拟器截图 + HTML 汇总，自动梳理指定模块的所有 UI 状态组合（订阅/非订阅、加载中/成功/失败/空态、有网/无网、弹窗等），
  并主动构造极端数据覆盖边界场景（空内容/超长文本/超长列表/零值负数/图片加载失败/特殊字符等），输出带状态文案标注的可视化 HTML 文档。
  支持多机型对比截图（大屏/小屏/SE），同一状态在不同尺寸设备上的适配一目了然。
  适用于：功能开发完成后的视觉走查、设计评审、QA 测试用例对照、边界case验收、多机型适配验收。
---

# Flutter 全状态截图生成

## 核心原则

> **只做截图，不做分析。** 输出物只有：模拟器真实截图 + 每张截图的状态标注。
> 不要在 HTML 中输出任何额外数据面板（如 API 返回数据、账号分析、指标卡片等）。
> HTML 唯一展示内容 = 手机外框套真实截图 + 简短状态标签。
>
> **极端情况是标配，不是可选项。** 每次截图必须主动构造边界数据（空内容、超长文本、零值、图片缺失等）。
> 不需要用户额外要求，截图清单中默认包含极端场景。正常状态和极端状态在 HTML 中分区展示。

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

1. **确定目标机型列表**：
   - 默认使用当前已启动的单个模拟器
   - 如果用户要求多机型截图，从预设机型组中选取（见「多机型截图」章节）
   - 查看已安装的可用模拟器：
     ```bash
     xcrun simctl list devices available | grep -E "iPhone|iPad"
     ```

2. 检查 iOS 模拟器是否已启动：
   ```bash
   xcrun simctl list devices booted
   ```

3. 检查 Flutter app 是否在运行：
   ```bash
   ps aux | grep "flutter.*run"
   ```

4. 如未运行，启动模拟器和 Flutter app：
   ```bash
   open -a Simulator
   xcrun simctl boot <DEVICE_ID>
   cd <PROJECT_ROOT> && flutter run -d <DEVICE_ID>
   ```

5. 等待 app 完全加载（检查 terminal 输出中出现 "Flutter run key commands"）

6. 在 Skill 所在目录下创建截图输出目录：
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
- 分为两大区域：**正常状态** 和 **极端 / 边界场景**，各自独立 section
- 每个状态维度一个 section，卡片横向排列
- 极端场景的卡片使用醒目的边框色（如橙色 `#f59e0b`）以区分正常状态
- 每张截图只配有：**状态名称**（简短标题）+ **触发条件**（代码级，如 `isVip() == true`）+ **边界类型标签**（如 `EDGE: 空内容`、`EDGE: 超长文本`，仅极端场景显示）
- 底部包含状态维度覆盖总览表（正常状态数 + 极端场景数分别统计）
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

## 极端情况 & 边界场景（必须覆盖）

> **每个功能都要主动构造边界数据来截图，不能只截"正常情况"。**
> 通过临时修改代码中的 mock 数据 / API 返回值 / 状态变量来触发这些极端场景。

### 内容长度边界

| 场景 | 如何构造 | 关注点 |
|------|---------|--------|
| **空内容** | 返回空字符串 `""` 或空列表 `[]` | 空态占位符是否正确显示、布局是否塌陷 |
| **单字/极短** | `"A"` 或 `"Hi"` | 最小宽度、对齐方式、间距是否异常 |
| **超长单行文本** | 200+ 字符无换行 | 文本截断/省略号/换行策略、是否溢出容器 |
| **超长多行文本** | 20+ 行 Markdown 或纯文本 | 滚动/展开折叠、卡片高度、页面被撑开 |
| **特殊字符** | emoji、中日韩混排、RTL 文字、`<script>` HTML 注入 | 渲染是否正常、XSS 防护 |
| **超长用户名/标题** | 50+ 字符的名称 | 截断方式、tooltip、布局挤压 |

### 列表/集合边界

| 场景 | 如何构造 | 关注点 |
|------|---------|--------|
| **0 条数据** | 返回空列表 | 空态页面/空态占位符 |
| **1 条数据** | 只返回 1 个 item | 单条时的居中/对齐、分割线、边距 |
| **恰好满一屏** | 调整条数刚好填满屏幕 | 是否出现滚动指示器 |
| **超长列表** | 100+ 条数据 | 性能、滚动流畅度、分页/加载更多 |
| **列表 item 高度不一致** | 混合长短内容 | 卡片对齐、瀑布流/等高处理 |

### 数值边界

| 场景 | 如何构造 | 关注点 |
|------|---------|--------|
| **零值** | `0`、`0.0`、`0%` | 进度条归零、数字显示 "0" 还是 "--" |
| **负数** | `-1`、`-99.9%` | 负数颜色/符号、图表反向 |
| **极大数** | `999999999`、`1.2B` | 数字格式化（K/M/B）、宽度溢出 |
| **小数精度** | `0.123456789` | 小数位截断、四舍五入 |
| **百分比 100%** | 进度 100/100 | 进度条满格样式、完成态 |
| **百分比 > 100%** | 异常数据 105% | 是否 clamp、溢出处理 |

### 图片/媒体边界

| 场景 | 如何构造 | 关注点 |
|------|---------|--------|
| **无图片** | imageUrl 为 null 或空 | placeholder/默认头像 |
| **加载失败** | 设置一个 404 的 URL | errorWidget 是否展示 |
| **超窄/超宽图片** | 1:10 或 10:1 的宽高比 | 裁切/填充策略、变形 |
| **超大图片** | 5000x5000 像素 | 内存占用、加载速度 |

### 时间/日期边界

| 场景 | 如何构造 | 关注点 |
|------|---------|--------|
| **刚刚/几秒前** | timestamp = now | "刚刚" 或 "Just now" 显示 |
| **跨天** | 昨天 23:59 → 今天 00:01 | 日期分隔线、"昨天" 标签 |
| **很久以前** | 2020-01-01 | 完整日期格式、"3 年前" |
| **未来时间** | 明天的 timestamp | 倒计时、过期逻辑 |

### 网络/异步边界

| 场景 | 如何构造 | 关注点 |
|------|---------|--------|
| **超时** | 模拟网络请求延迟 30s+ | 超时提示、重试按钮 |
| **部分成功** | 批量请求中 1 个失败 | 局部错误 vs 全局错误展示 |
| **重复请求** | 快速连点按钮 | 防重复提交、loading 防穿透 |
| **离线状态** | 模拟器断网 | 离线提示、缓存数据展示 |

### 构造极端数据的方法

1. **临时修改 mock 数据**（最常用）：
   ```dart
   // 在变量声明处直接赋极端值
   String _userName = 'A' * 100;  // 超长用户名
   List<Item> _items = [];         // 空列表
   double _progress = 1.05;        // 超过 100%
   ```

2. **修改 API 返回拦截**：在网络层临时插入 mock response

3. **修改 Widget 参数**：直接给组件传极端值
   ```dart
   AppMetricCard(value: 999999999, label: 'A very long label text here')
   ```

4. 每次修改后 → terminate app → flutter run → 截图 → 还原代码

### 极端场景截图命名规范

使用 `edge_` 前缀区分：
```
09_edge_empty_list.png
10_edge_long_text.png
11_edge_zero_value.png
12_edge_image_404.png
```

## 状态梳理 Checklist

分析页面时，务必检查以下维度（含极端情况）：

| 维度 | 检查点 |
|------|--------|
| 认证/订阅 | 已登录/未登录、VIP/非VIP、试用期/过期 |
| 网络状态 | 请求成功/失败/超时/离线 → Toast/全页错误/重试 |
| 加载状态 | loading indicator / skeleton / shimmer |
| 数据状态 | 有数据 / 空态 / 首次加载 |
| **内容长度** | **空字符串 / 极短(1字) / 正常 / 超长(200+字符) / 多行(20+行)** |
| **列表边界** | **0条 / 1条 / 恰好满屏 / 超长列表(100+) / item高度不一致** |
| **数值边界** | **0 / 负数 / 极大数(999999999) / 小数精度 / 百分比≥100%** |
| **图片媒体** | **正常 / 无图(null) / 加载失败(404) / 异常宽高比** |
| **特殊字符** | **emoji / 中日韩混排 / HTML标签 / 换行符** |
| **设备尺寸** | **大屏(Pro Max) / 标准屏 / 小屏(SE/mini) — 布局适配、文字截断、底部遮挡** |
| 平台差异 | iOS vs Android 的条件分支 |
| 账号绑定 | 已绑定/未绑定第三方账号 |
| 弹窗覆盖 | Dialog / BottomSheet / Toast / Snackbar |
| 权限状态 | 通知权限、相册权限、剪贴板权限等 |
| 营销活动 | 限时折扣、倒计时、促销 banner |
| 引导状态 | 新手引导、功能引导（首次 vs 非首次） |
| 操作反馈 | Copy/Paste/刷新/展开折叠等交互响应 |
| **快速重复操作** | **连点按钮 → 防重复提交、loading防穿透** |
| 外部唤起 | 深链接/剪贴板检测跳入功能的表现 |
| **时间边界** | **刚刚 / 跨天 / 很久以前 / 未来时间** |

## 多机型截图

> 同一个状态在不同屏幕尺寸上可能表现完全不同（文字截断、布局溢出、底部遮挡等）。
> 当用户要求"多机型截图"或"适配验收"时启用本流程。

### 触发条件

以下说法触发多机型模式：
- "帮我在不同机型上截图"
- "截图 XX 功能的多机型适配"
- "大屏小屏都截一下"
- "SE 上看看效果"
- "帮我做一下适配验收"

### 预设机型组

| 组别 | 机型 | 屏幕特征 | 用途 |
|------|------|---------|------|
| **核心三件套**（默认） | iPhone 16 Pro | 6.3" 大屏 | 主力机型基准 |
| | iPhone 16 | 6.1" 标准屏 | 最常见尺寸 |
| | iPhone SE (3rd) | 4.7" 小屏 | 小屏极端适配 |
| **完整覆盖** | 上述 + iPhone 16 Pro Max | 6.9" 超大屏 | 最大屏幕验证 |
| | + iPhone 13 mini | 5.4" 迷你屏 | 最小刘海屏 |
| **自定义** | 用户指定 | — | 按需选取 |

> 用户未指定时，默认使用"核心三件套"。

### 工作流程

1. **创建所需模拟器**（如果不存在）：
   ```bash
   # 查看可用的 device type
   xcrun simctl list devicetypes | grep -i "iphone"
   # 查看可用的 runtime
   xcrun simctl list runtimes | grep -i ios
   # 创建模拟器（示例）
   xcrun simctl create "iPhone SE 3" com.apple.CoreSimulator.SimDeviceType.iPhone-SE-3rd-generation com.apple.CoreSimulator.SimRuntime.iOS-18-4
   ```

2. **逐机型截图循环**：
   ```
   for each DEVICE in 机型列表:
     1. 关闭当前 app（如有）
     2. boot 目标模拟器
        xcrun simctl boot <DEVICE_ID>
     3. 安装并启动 app
        xcrun simctl install <DEVICE_ID> <APP_PATH>
        xcrun simctl launch <DEVICE_ID> <BUNDLE_ID>
        或使用 flutter run -d <DEVICE_ID>
     4. 等待 app 加载完成
     5. 对每个目标状态执行截图
        xcrun simctl io <DEVICE_ID> screenshot <PATH>/<序号>_<机型>_<状态>.png
     6. shutdown 当前模拟器
        xcrun simctl shutdown <DEVICE_ID>
   ```

3. **优化策略 — 减少重复启动**：
   - 如果只需要对比少量关键状态（≤5 个），可以先在主机型上截完全部状态，再在其他机型上只截这几个关键状态
   - 对于需要临时改代码的状态，在切换机型前保持代码修改不变，多个机型连续截图后再还原

### 截图命名规范（多机型）

增加机型标识：
```
01_pro_plan_empty.png        ← iPhone 16 Pro
01_std_plan_empty.png        ← iPhone 16
01_se_plan_empty.png         ← iPhone SE
01_max_plan_empty.png        ← iPhone 16 Pro Max
01_mini_plan_empty.png       ← iPhone 13 mini
```

机型缩写对照：
| 机型 | 缩写 |
|------|------|
| iPhone 16 Pro Max | `max` |
| iPhone 16 Pro | `pro` |
| iPhone 16 | `std` |
| iPhone SE (3rd) | `se` |
| iPhone 13 mini | `mini` |
| 其他 | 取型号中最短辨识词 |

### HTML 输出（多机型模式）

多机型模式下 HTML 布局调整为：

- **按状态分组**：每个状态一行，该行内横向排列所有机型的截图
- 每张截图下方标注**机型名称**和**屏幕尺寸**
- 手机外框的宽度按机型实际比例微调（SE 窄一些、Pro Max 宽一些）：
  ```css
  .phone-frame.size-se   { width: 220px; border-radius: 28px; }
  .phone-frame.size-std  { width: 260px; }
  .phone-frame.size-pro  { width: 280px; }
  .phone-frame.size-max  { width: 300px; }
  .phone-frame.size-mini { width: 240px; }
  ```
- 底部覆盖率表格增加"机型覆盖"列
- 如果某状态在某机型上布局明显异常（溢出、截断、遮挡），自动标注红色 `⚠️ 适配问题` 标签

### 输出目录结构（多机型）

```
output/<模块名>/
├── <模块名>_states.html
└── screenshots/
    ├── 01_pro_xxx.png
    ├── 01_std_xxx.png
    ├── 01_se_xxx.png
    ├── 02_pro_xxx.png
    ├── 02_std_xxx.png
    ├── 02_se_xxx.png
    └── ...
```

## 注意事项

- **iOS 粘贴权限弹窗**：iOS 16+ 中 `Clipboard.getData()` 会触发系统级弹窗。截图前用 `pbcopy < /dev/null` 清空剪贴板；如项目有自动剪贴板检测逻辑，建议在 debug 模式下加开关跳过。
- 修改代码截图后**务必还原**（先 `git stash` 保存当前改动，截图后 `git checkout -- <files>` 还原，再 `git stash pop`）
- 如果某些状态需要后端配合才能触发（如折扣倒计时），在 HTML 中标注为"需后端配合"
- 对于代码中存在但当前不可达的状态（如写死的 Mock 数据），标注为"代码预留"
- **不要在 HTML 中添加任何数据分析面板**，输出物只有截图 + 状态标注
- 项目的 Bundle ID 从 `ios/Runner.xcodeproj/project.pbxproj` 中的 `PRODUCT_BUNDLE_IDENTIFIER` 获取

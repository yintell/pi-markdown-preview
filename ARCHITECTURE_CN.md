# pi-markdown-preview 插件工作原理（中文说明）

本文档从源码角度，详细解释 `pi-markdown-preview` 插件的内部工作原理，包括架构设计、核心执行流程、关键文件与组件，以及重要实现细节与限制。

---

## 一、总体架构

```
用户在 pi 中输入命令 /preview [参数]
        │
        ▼
index.ts — 插件入口，注册命令，解析参数
        │
        ├─── 获取内容 ─────────────────────────────────────────────────────────┐
        │    · getLastAssistantMarkdown()  最新助手回复                        │
        │    · pickAssistantMessage()      UI 选择历史回复                     │
        │    · readFile()                  本地文件（.md / .tex / 代码 / diff）│
        │                                                                       │
        ▼                                                                       │
  内容预处理                                                                    │
  · normalizeMathDelimiters()              数学公式分隔符归一化                 │
  · normalizeMarkdownFencedBlocks()        fenced block 格式化                 │
  · prepareMarkdownForPandocPreview()      注解标记 → 占位符                   │
        │                                                                       │
        ▼                                                                       │
┌───────────────────────────────────────────────────────────────────────────┐  │
│ 三种渲染目标                                                               │  │
│                                                                           │  │
│  ┌── 终端预览（默认）──────────────────────────────────────────────────┐  │  │
│  │ renderPreview()                                                     │  │  │
│  │  1. pandoc Markdown/LaTeX → HTML片段（含 MathML）                  │  │  │
│  │  2. buildBrowserHtmlFromPandocFragment() 组装完整 HTML + CSS变量   │  │  │
│  │  3. puppeteer 无头 Chromium 加载 HTML，执行页内 JS                 │  │  │
│  │     · 渲染 Mermaid（CDN）                                          │  │  │
│  │     · 恢复注解占位符 → <span class="annotation-marker">           │  │  │
│  │     · 装饰 diff 代码块                                             │  │  │
│  │     · MathJax 兜底数学公式（CDN）                                 │  │  │
│  │  4. 截图 → 按 2200px 切片 → PNG Base64                            │  │  │
│  │  5. MarkdownPreviewOverlay 用 pi-tui Image 组件显示在终端          │  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │  │
│                                                                           │  │
│  ┌── 浏览器预览 ───────────────────────────────────────────────────────┐  │  │
│  │ openPreviewInBrowser()                                              │  │  │
│  │  1. 同上 pandoc + HTML 组装                                        │  │  │
│  │  2. 写入缓存文件 ~/.pi/cache/markdown-preview/<hash>.html          │  │  │
│  │  3. open / xdg-open / start 打开系统浏览器                         │  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │  │
│                                                                           │  │
│  ┌── PDF 导出 ─────────────────────────────────────────────────────────┐  │  │
│  │ exportPdf()                                                         │  │  │
│  │  · LaTeX 文件：直接 compileLatexToPdf()（xelatex/pdflatex 两遍）   │  │  │
│  │  · 含 diff 的 Markdown：pandoc → LaTeX → xelatex（diff 重写）      │  │  │
│  │  · 普通 Markdown：pandoc → PDF（内置 xelatex 引擎）                │  │  │
│  │  · Mermaid：先用 mmdc CLI 预渲染为 SVG，再嵌入 Markdown           │  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │  │
└───────────────────────────────────────────────────────────────────────────┘  │
                                                                               │
        ◄──────────────────────────────────────────────────────────────────────┘
```

---

## 二、关键文件与职责

| 文件 | 语言 | 运行环境 | 职责 |
|---|---|---|---|
| `index.ts` | TypeScript | Node.js（pi 扩展运行时） | 插件全部主体逻辑：命令注册、参数解析、内容获取、HTML 构建、puppeteer 截图、TUI 展示、PDF 导出 |
| `shared/annotation-scanner.js` | JavaScript | Node.js + 浏览器（共享） | 解析 `[an: ...]` 注解标记；被 index.ts 和浏览器端都使用 |
| `client/annotation-helpers.js` | JavaScript | 浏览器 | 浏览器端注解标记渲染；通过 `readFileSync` 读取后内联到 HTML `<script>` 标签 |
| `package.json` | JSON | — | 声明插件入口为 `./index.ts`，依赖 `puppeteer-core`，peer 依赖 `@mariozechner/pi-coding-agent` 和 `@mariozechner/pi-tui` |

---

## 三、插件注册与命令入口

`index.ts` 的最后通过 `export default function(pi: ExtensionAPI)` 向 pi 运行时注册四个命令：

```
/preview            — 主命令，默认终端预览
/preview-browser    — 浏览器预览（快捷方式，内部转发给 /preview --browser）
/preview-pdf        — PDF 导出（快捷方式，内部转发给 /preview --pdf）
/preview-clear-cache — 清除缓存目录
```

`parsePreviewArgs()` 函数负责解析参数字符串，支持 `--pick`、`--file`、`--browser`、`--pdf`、`--terminal`、`--font-size` 等标志，并做冲突检查。

---

## 四、如何获取输入内容

```typescript
// 1. 从 pi 会话中获取最新助手消息
function getLastAssistantMarkdown(ctx) {
    // 遍历 ctx.sessionManager.getBranch() 中的 message 条目
    // 过滤出 role === "assistant" 的消息，拼接其 text 块
}

// 2. 弹出 SelectList UI，让用户从历史回复中选择
async function pickAssistantMessage(ctx) { ... }

// 3. 读取本地文件
//    .md/.mdx/.rmd/.qmd → Markdown
//    .tex               → LaTeX（isLatex=true）
//    其他（.py/.ts 等） → 包装为 fenced code block
```

文件类型判断通过 `isLatexFile()` / `isMarkdownFile()` / `detectLanguageFromPath()` 完成，纯代码文件用 `wrapCodeAsMarkdown()` 加语言标注包裹后交给通用 Markdown 流程处理。

---

## 五、Markdown / LaTeX 预览渲染流程（终端）

### 5.1 内容预处理

`prepareBrowserPreviewMarkdown(markdown, isLatex)` 完成以下步骤（针对 Markdown）：

1. `normalizeMathDelimiters()`：将 `\[...\]` / `\(...\)` 转为 `$$...$$` / `$...$`，保证 pandoc 能识别。
2. `normalizeMarkdownFencedBlocks()`：统一 fenced block 格式，修复嵌套/不规范写法。
3. `prepareMarkdownForPandocPreview()`（来自 `shared/annotation-scanner.js`）：扫描 `[an: ...]` 注解标记，在围栏外将其替换为 `PIMDPREVIEWANNOT<token>` 占位符，并收集 `annotationPlaceholders` 列表以供后续 HTML 端恢复。这一步的目的是防止 pandoc 将注解标记误解析为链接语法。

### 5.2 调用 pandoc 生成 HTML 片段

```typescript
async function renderMarkdownToHtmlWithPandoc(markdown, resourcePath?, isLatex?) {
    // 使用 Node.js child_process.spawn 调用 pandoc
    // 输入格式：markdown+... 扩展（或 latex）
    // 输出格式：html5
    // 关键标志：--mathml（数学公式转 MathML）、--wrap=none
    // 将 markdown 通过 stdin 传入，从 stdout 读取 HTML 片段
}
```

pandoc 会处理：
- Markdown 语法 → HTML 结构
- `$...$` / `$$...$$` 数学公式 → MathML（浏览器原生渲染）
- fenced code block → 带 pandoc 语法高亮的 `<pre><code>` HTML
- 表格、blockquote、图片、链接等

### 5.3 组装完整 HTML 页面

`buildBrowserHtmlFromPandocFragment(fragmentHtml, style, ...)` 生成一个完整的 `<!doctype html>` 页面，包含：

- **CSS 变量**：来自 `buildPreviewCssVars(style)`，将 pi 主题颜色映射为 `--bg`、`--text`、`--md-heading`、`--syntax-keyword` 等 CSS 变量。
- **内嵌 CSS**：完整的 Markdown 样式规则，使用上述变量，覆盖代码块、表格、diff、注解标记等元素。
- **pandoc HTML 片段**：放在 `<article id="preview-root">` 内。
- **`client/annotation-helpers.js` 内容**：直接内联到 `<script>` 标签，用于浏览器端注解渲染。
- **页内异步 JS 模块**（`<script type="module">`）：负责执行以下步骤后设置 `window.__mermaidDone = true`：
  1. `renderMermaid()`：从 jsDelivr CDN 动态 import Mermaid，渲染 `<pre class="mermaid">` 为 SVG。
  2. `applyPreviewAnnotationPlaceholders(root)`：将占位符文本节点替换为 `<span class="annotation-marker">` 元素。
  3. `decorateDiffCodeBlocks(root)`：为 diff 代码块的每行添加 `diff-add-line` / `diff-del-line` 等 CSS 类。
  4. `renderAnnotationMarkerMath(root)`：对含 LaTeX 的注解标记调用 MathJax。
  5. `renderMathFallback(root)`：对 pandoc 未能转换的 `.math.display` / `.math.inline` 节点，通过 MathJax CDN 兜底渲染。

### 5.4 无头 Chromium 截图

`renderPreview()` 函数使用 `puppeteer-core`：

1. 探测可用的 Chromium 可执行文件（`findBrowserExecutable()`，或读取 `PUPPETEER_EXECUTABLE_PATH`）。
2. 启动无头浏览器，创建新页面，设置视口宽度为 `1200px`，device scale factor 为 `2`（可配置，用于 HiDPI）。
3. **第一遍**：以 `900px` 高加载 HTML，等待 DOM 加载完成和 `window.__mermaidDone === true`，然后测量 `#preview-root` 的实际高度。
4. **第二遍**：以测量到的真实高度重新加载 HTML，确保内容不被截断。
5. 调用 `page.screenshot()` 取全页截图（PNG Buffer）。
6. 按 `PAGE_HEIGHT_PX = 2200px` 将截图切分为多页（使用 puppeteer 的 `clip` 选项）。
7. 每页存为 PNG Base64 字符串，并写入磁盘缓存。

> **重要限制**：
> - 最大渲染高度为 `MAX_RENDER_HEIGHT_PX = 66000px`（30 页），超出部分会被截断并给出警告。
> - 终端图片展示依赖终端对 Kitty / iTerm2 / Ghostty / WezTerm 图片协议的支持，不支持的终端只能看到灰色占位。
> - Mermaid 和 MathJax 均从外部 CDN 加载，在离线或无法访问网络的环境下这两项渲染将退化（公式退化为原始文本，Mermaid 退化为代码块）。

### 5.5 TUI 展示

`MarkdownPreviewOverlay` 类（继承/组合 pi-tui 的 `Container`）：
- 将截图用 `pi-tui` 的 `Image` 组件显示到终端（通过 Kitty / iTerm2 图片协议）。
- 渲染顶部标题栏（页码 `1/N`、主题模式）和操作提示。
- 处理键盘输入：`←`/`→` 翻页，`r` 刷新（跳过缓存），`o` 打开浏览器预览，`Esc`/`Ctrl+C` 关闭。

---

## 六、浏览器预览流程

```
openPreviewInBrowser()
  │
  ├─ 同 5.1～5.3，生成完整 HTML（CSS 变量 + pandoc 片段 + 页内 JS）
  │
  ├─ 计算 SHA-256 哈希（RENDER_VERSION + "browser-native" + 主题 cacheKey + markdown）
  │
  ├─ 写入 ~/.pi/cache/markdown-preview/<hash>.html
  │
  └─ openFileInDefaultBrowser()
       macOS：open <file://path>
       Linux：xdg-open <file://path>
       Windows：cmd /c start "" <file://path>
```

浏览器预览与终端预览用同一套 HTML/CSS/JS，Mermaid 和 MathJax 在真实浏览器中实时运行，效果更完整。

---

## 七、PDF 导出流程

```
exportPdf()
  │
  ├─ normalizeMathDelimiters() + normalizeMarkdownFencedBlocks()
  │
  ├─ preprocessMermaidForPdf()：
  │    · 扫描 ```mermaid 块，调用 mmdc CLI 生成 SVG，
  │    · 将代码块替换为 Markdown 图片链接（![...](file:///...)）
  │    · 若无 mmdc，保留原始代码块
  │
  ├─ highlightAnnotationMarkersForPdf()：
  │    · 将 [an: ...] 注解标记转为 LaTeX \piannotation{...} 宏调用
  │
  ├─ 根据内容类型选择编译路径：
  │
  │  · isLatex（.tex 文件）→ compileLatexToPdf()
  │       直接用 xelatex 编译两遍（解决 \ref 交叉引用）
  │
  │  · hasMarkdownDiffFence → renderMarkdownToPdfViaGeneratedLatex()
  │       pandoc Markdown → LaTeX，
  │       rewriteGeneratedDiffHighlighting()（diff 行着色宏替换），
  │       compileLatexToPdf()
  │
  │  · 普通 Markdown → renderMarkdownToPdf()
  │       pandoc Markdown → PDF（直接，--pdf-engine=xelatex）
  │
  └─ 打开 PDF 文件
```

**LaTeX 导言区（PDF_PREAMBLE）** 内嵌在 `index.ts` 中，定义了：
- `\piannotation{...}`：注解标记的彩色方框样式（使用 `varwidth` 和 `fcolorbox`）。
- `\PiDiffAddTok`、`\PiDiffDelTok` 等：diff 行的颜色宏。
- `titlesec`、`enumitem`、`parskip`、`fvextra` 等宏包配置，确保排版美观。

---

## 八、主题适配机制

1. 每次执行预览命令都会调用 `getPreviewStyle(ctx.ui.theme)` 从 pi 当前主题读取颜色信息。
2. 颜色提取通过 `theme.getFgAnsi(token)` / `theme.getBgAnsi(token)` / `readThemeColorToken()` 多种方式尝试，最终 fallback 到内置的 `DARK_PREVIEW_PALETTE` / `LIGHT_PREVIEW_PALETTE`。
3. 所有颜色被序列化为 CSS 变量字符串 `cacheKey`，缓存 key 包含此字符串，保证不同主题各自有独立缓存。
4. `inferThemeModeFromName()` 根据主题名称中是否含有 `light`、`dark`、`dawn`、`mocha` 等关键字来推断深色/浅色模式，影响颜色混合计算（如 blockquote 背景透明度）。

---

## 九、缓存机制

- 缓存目录：`~/.pi/cache/markdown-preview/`
- 缓存 key：SHA-256(`RENDER_VERSION` + 渲染类型 + `style.cacheKey` + 原始 Markdown 内容)
- 同一内容多页时，额外存储 `<hash>\0page<i>` 子键
- 缓存命中时直接读取 PNG Buffer，跳过 pandoc 和 puppeteer 流程，实现"瞬时重显"
- `r` 键触发 `skipCache=true`，强制重新渲染（适用于主题切换后刷新）
- `/preview-clear-cache` 命令删除整个缓存目录

---

## 十、注解标记（Annotation Markers）机制

`[an: 这是注解]` 语法是 pi 特有的内联标注语法，在预览中显示为一个带边框的小芯片（chip）。

**服务端（Node.js / `index.ts` + `shared/annotation-scanner.js`）**：
1. `prepareMarkdownForPandocPreview()` 扫描 Markdown 中所有围栏外的 `[an: ...]` 标记。
2. 将其替换为 `PIMDPREVIEWANNOT<hash>` 占位符，收集 `{ token, text, title }` 列表。
3. 将占位符文本安全地传给 pandoc，不会被解析为链接。
4. 将占位符列表序列化为 JSON 内嵌到 HTML 页面的 `previewAnnotationPlaceholders` 变量。

**浏览器端（`client/annotation-helpers.js` + 页内 JS）**：
1. `applyPreviewAnnotationPlaceholders(root)` 遍历文本节点，找到占位符，替换为 `<span class="annotation-marker">` 元素。
2. 对于 pandoc 直接透传的 `[an: ...]` 文本（如 diff 块内），`decorateDiffCodeBlocks()` 也会调用 `replaceAnnotationTextNode()` 进行处理。

**PDF 端**：直接使用 `highlightAnnotationMarkersForPdf()` 将 `[an: ...]` 转换为 `\piannotation{...}` LaTeX 宏，由 LaTeX 引擎渲染为彩色方框。

---

## 十一、TypeScript / JavaScript / TeX 三端配合

```
TypeScript (index.ts, Node.js)
│
│   编译时读取 ──► client/annotation-helpers.js   (readFileSync + import.meta.url)
│                  └─ 内联到 HTML <script> 标签
│
│   运行时调用 ──► shared/annotation-scanner.js    (ESM import)
│
│   运行时调用 ──► pandoc（子进程）→ HTML / LaTeX / PDF
│
│   运行时调用 ──► puppeteer-core → 无头 Chromium
│                  └─ 加载 HTML，触发 CDN JS 资源（Mermaid, MathJax）
│
│   运行时调用 ──► xelatex/pdflatex（子进程）
│                  └─ 使用内嵌的 PDF_PREAMBLE TeX 宏包配置
│
│   运行时调用 ──► mmdc（子进程，可选）→ Mermaid SVG
│
│   运行时调用 ──► pi-tui（@mariozechner/pi-tui）
│                  └─ TUI 组件：Container, Image, Text, SelectList 等
│
└─ 注册到 pi 扩展 API（@mariozechner/pi-coding-agent）
```

---

## 十二、重要限制与注意事项

1. **需要 pandoc**：无论哪种预览模式都必须有 pandoc，缺少时会报错。
2. **终端预览需要 Chromium**：puppeteer-core 需要 Chrome/Brave/Edge/Chromium 可执行文件，通过 `PUPPETEER_EXECUTABLE_PATH` 配置或自动探测。
3. **终端图片显示需要支持图片协议的终端**：Kitty、iTerm2、Ghostty、WezTerm 等，其他终端无法显示图片。
4. **PDF 导出需要 LaTeX 引擎**：默认 `xelatex`，可通过 `PANDOC_PDF_ENGINE` 配置。
5. **Mermaid 在 PDF 中需要 mmdc CLI**：可选，缺少时退化为代码块。
6. **在线资源依赖**：浏览器和终端预览中，Mermaid 和 MathJax 从 jsDelivr CDN 加载，离线环境下无法渲染。
7. **无实时同步**：预览是"按需渲染"，不监听文件变化，需手动刷新（`r`）。
8. **预览不可交互**：终端展示的是静态截图，无法点击链接或与内容交互。
9. **大文档性能**：超长文档会触发多次 puppeteer 截图，首次渲染较慢；缓存后重显很快。
10. **`overlay` 模式禁用**：注释中明确说明 overlay 合成模式会截断 Kitty/iTerm2 图片协议序列，因此强制使用非 overlay 模式（`ctx.ui.custom` 的全屏模式）。

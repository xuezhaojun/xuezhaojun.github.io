# AGENTS.md

## Project

Hugo static site (personal blog). Theme: PaperMod (git submodule — never edit `themes/PaperMod/`).

## Commands

```bash
hugo server          # local dev server
hugo --gc --minify   # production build (used by CI)
```

No Makefile, no package.json, no other build tooling.

## Content

- **Posts** go in `content/posts/`. Every article lives here and shows up under Blog.
- **Collections** are a Hugo taxonomy (`collections`), not a content section. A post joins a collection via frontmatter, e.g. `collections: ["Go Concurrency"]` plus `weight: 1` for chapter ordering. `/collections/` lists all collections; each collection page shows its posts as a book-style TOC, and post pages get a collection badge + prev/next chapter navigation. Use title-case collection names (the URL is auto-slugged).
  - Optional `pinned: true` frontmatter pins a post to the top of its collection page (rendered first, in weight order, with a 置顶 badge). Pinned posts always sort above non-pinned ones regardless of `weight`.
  - Current collections:
    - `GPU Scheduling` — GPU 调度知识地图（长期维护的资源索引贴；只收录作者读过且认为有价值的内容，不预填种子资源）
    - `Kubernetes Operators` — Kubernetes Operator 开发知识地图（同上）
- **Tags** are a closed set, always in lowercase — a post's `tags` must only contain values from this list, nothing else:
  - Languages: `golang`, `python`, `typescript`
  - Domains: `k8s`, `ai-infra`, `ai-agent`
  - Meta: `translated` (the post is translated by the author)
- All content uses **YAML frontmatter** (`---`), not TOML.
- Required frontmatter: `title`, `date`, `draft`, `tags`, `summary`.
- Site is bilingual (Chinese primary, English secondary).

## Gotchas

- Hugo version: **0.152.1 extended** (CI pins this exact version).
- `markup.goldmark.renderer.unsafe = true` — raw HTML in Markdown is intentional.
- Mermaid diagrams work via a custom code block renderer (`layouts/_default/_markup/render-codeblock-mermaid.html`).
- Images go in `static/images/`.
- Pushing to `main` triggers GitHub Pages deployment automatically.

## Blog 写作偏好

下面的偏好用于写新 post 或 review 用户的 draft，目标是能一步到位、不用反复打磨措辞。

### 标题与摘要

- **标题直白、不加副标题修饰。** 用 `Why Simple Kind` 而不是 `Why Simple Kind — Kubernetes 为什么需要第三种类型`。副标题能省就省。
- **summary 能短则短，不写完整论断句。** `"为什么 Kubernetes 需要 Simple kind。"` 优于把核心论点整句塞进 summary。

### 风格与语气

- **保持清晰易读即可，不要强行口语化。** 避免把技术写作往"闲聊"方向推，不要用"没头没尾"、"偷偷返回"、"瞎造"、"搞一个"这种轻浮的口语表达。目标是让读者读起来不费力，不是让文章显得"接地气"或"有人味"。专业话题用平实、克制的语气，比强装随意更清楚，也更尊重读者。

### 论证结构

- **从真实业务场景/需求出发，不从抽象论断出发。** 开篇先给"我们和系统交互时的具体需求"（如查 `/status`、调 `/scale`、`kubectl exec`），再推出要解释的概念。不要用"它是不是可选的设计选择"这种抽象论断起手。
- **每个技术操作回答"我们为什么做这个交互、目的是什么"**，不要只描述"它返回什么"。例如解释 `GET /scale` 时，要说"HPA 想知道副本数好决定扩缩容"，而不是"它返回一个临时结果"。
- **每个章节应该是独立论点；角度重叠的论点要合并。** 例如"客户端需要 GVK 反序列化"和"编解码链路需要 GVK"是同一论点的两个角度，合并成一处。
- **抽象论点用具体例子支撑，给完整例子而不是片段。** 论"跨资源统一接口"时，给 Deployment/StatefulSet/S3Bucket 三个具体调用 + 翻译放在资源侧的解释 + Eviction/AdmissionReview 等同类例子，而不是只抽象描述。
- **结尾不要靠 mermaid 图收尾。** 文字总结已经足够时，不要再加一张总结性 mermaid 图，那是冗余。

### 概念引入

- **引入一个新概念/工具时先给一句"它是什么、干什么用"，再展开。** 不要假设读者已经知道 HPA、CRD subresource、admission webhook 这类术语。例如讲 HPA 之前先说"它是 K8s 内置的根据负载自动扩缩容的控制器"+ 一个 minimal YAML 示例。

### 用词与表达

- **避免同义反复。** "不是可选的设计选择"（可选的 + 选择 重叠）→ "不是设计上的可选项"。
- **避免生造或不通顺的复合词。** "响应自描述" → "自带类型标签"；"现取现返" → "收到请求时才从...取出来返回"；"计算出来的投影" → "对应字段的镜像"。原则：用最直白的描述，不要为了"显得精炼"造新词。
- **避免口语/不雅的词。** 不要用"一坨参数"、"一坨结果"这种说法，用"传参数进去，拿结果回来"。
- **不自然的动词替换。** "落地到 etcd" → "存储到 etcd"。
- **分类标题/结构性位置用平实描述，不用比喻。** `第一类：错误/状态信封` → `第一类：错误/状态响应`。比喻只在真有解释力时用于解释段，不用于命名/分类标题。
- **慎用类比段。** 类比段（如"Object = 书，List = 书架，Simple = 借书条"）往往是冗余，能删就删。如果保留，必须比正文直接说更清楚，而不是更绕。

### 准确性

- **论证从准确的机制出发，不要为了让论点更有力而夸大。** 例如"kubectl 不知道这串 payload 是不是 Scale"不准确——kubectl 知道自己请求的是 `/scale`，真正"不假设"的是解码层（generic 的、按 `kind` 分发）。写之前先确认机制描述是否准确。
- **改之前先评估用户论断是否事实准确。** 如果用户给的修改建议里有事实不准的地方，先指出再改，不要照单全收。
- **区分"约定"和"强制"。** 如果某个规则是被类型系统/接口/编译器强制的（如 `runtime.Object` 接口），明确说"是机制层面的硬规定"，而不是说"是约定"。

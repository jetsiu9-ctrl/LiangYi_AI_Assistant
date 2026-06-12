# OpenAI Dall-e 模式 UXP 插件 UI 开发说明

## 1. 目标

为 Photoshop UXP 插件开发一个 OpenAI Dall-e 格式生图面板。用户可以在插件内切换模型、切换生成模式，并根据所选模型自动显示可调整参数。UI 必须使用 Spectrum Web Components，布局只使用 Flexbox，文本必须放入 `sp-heading`、`sp-body`、`sp-label` 等 Spectrum 文本组件中。

插件核心能力：

- 文生图：调用 `POST /v1/images/generations`。
- 图生图：所有参考图上传统一参考 `F:\桌面\Photoshop-ai-all` 插件的方式，先从 Photoshop 当前文档导出临时图片并读取为 binary buffer，再进入统一的 `state.images` 队列，最后由请求适配层转换为目标接口需要的 JSON base64、FormData 文件或兼容字段。
- 图像编辑：调用 `POST /v1/images/edits`，支持参考图、mask、尺寸、质量、返回格式等参数。
- 结果处理：统一参考 `F:\桌面\Photoshop-ai-all` 插件方式，API 响应先解析为 `ArrayBuffer`，再创建 ObjectURL 预览并保存到 `state.resultImage = { url, data }`，插入 Photoshop 时只传递 binary buffer。

## 2. 官方约束

### UXP 与 Photoshop

- Photoshop 模块导入必须使用：

```js
const { app, core, action } = require("photoshop");
```

- UXP 文件与系统能力必须使用：

```js
const { storage, shell } = require("uxp");
```

- 所有修改 Photoshop 文档状态的操作必须放入 `core.executeAsModal`：

```js
await core.executeAsModal(async (context) => {
    // 创建文档、插入图层、修改图层、写入像素、运行 batchPlay 等操作
}, { commandName: "插入 AI 生成图像" });
```

- 网络请求、参数校验、UI 状态更新不需要进入 `executeAsModal`；只有真正修改 Photoshop 状态时进入。
- 不使用 CEP、CSInterface、ExtendScript、Node.js 原生 `fs` 或 `path`。

### UI 与 CSS

- 优先使用 SWC：`sp-button`、`sp-action-button`、`sp-dropdown`、`sp-menu-item`、`sp-textarea`、`sp-textfield`、`sp-slider`、`sp-checkbox`、`sp-radio-group`、`sp-progressbar`、`sp-divider`。
- 所有说明文字必须由 `sp-heading`、`sp-body`、`sp-label`、`sp-detail` 承载；`div`、`section` 只能作为无文本布局容器。
- 布局只能使用 Flexbox：`display: flex`、`flex-direction`、`gap`、`align-items`、`justify-content`。
- CSS 禁止 `display: grid`、`float`、`position: fixed`、`::before`、`::after`。

## 3. 信息架构

面板采用单列 Flexbox，按 Photoshop 插件面板宽度优先设计。

```html
<div class="panel-root">
    <sp-heading size="M">OpenAI Dall-e Image</sp-heading>

    <section class="section-block">
        <sp-label>API 设置</sp-label>
        <sp-textfield id="baseUrlInput" placeholder="Base URL"></sp-textfield>
        <sp-textfield id="apiKeyInput" type="password" placeholder="API Key"></sp-textfield>
    </section>

    <section class="section-block">
        <sp-label>生成模式</sp-label>
        <sp-radio-group id="modeGroup" selected="generations">
            <sp-radio value="generations">Generations</sp-radio>
            <sp-radio value="edits">Edits</sp-radio>
        </sp-radio-group>
    </section>

    <section class="section-block">
        <sp-label>模型</sp-label>
        <sp-dropdown id="modelDropdown">
            <sp-menu-item value="gpt-image-2">gpt-image-2</sp-menu-item>
            <sp-menu-item value="gpt-image-1">gpt-image-1</sp-menu-item>
            <sp-menu-item value="dall-e-3">dall-e-3</sp-menu-item>
            <sp-menu-item value="dall-e-2">dall-e-2</sp-menu-item>
            <sp-menu-item value="gemini-3.1-flash-image-preview">nano-banana-3.1-flash</sp-menu-item>
            <sp-menu-item value="qwen-image">qwen-image</sp-menu-item>
            <sp-menu-item value="qwen-image-edit">qwen-image-edit</sp-menu-item>
            <sp-menu-item value="flux-kontext-pro">flux-kontext-pro</sp-menu-item>
            <sp-menu-item value="flux-kontext-max">flux-kontext-max</sp-menu-item>
            <sp-menu-item value="recraftv3">recraftv3</sp-menu-item>
        </sp-dropdown>
    </section>

    <section class="section-block">
        <sp-label>Prompt</sp-label>
        <sp-textarea id="promptInput" grows></sp-textarea>
        <sp-detail id="promptCounter">0 / 32000</sp-detail>
    </section>

    <section id="dynamicParams" class="section-block"></section>

    <section id="resolutionParams" class="section-block">
        <sp-label>输出规格</sp-label>
        <sp-dropdown id="resolutionTierDropdown">
            <sp-menu-item value="auto">Auto</sp-menu-item>
            <sp-menu-item value="1k">1K</sp-menu-item>
            <sp-menu-item value="2k">2K</sp-menu-item>
            <sp-menu-item value="4k">4K</sp-menu-item>
        </sp-dropdown>
        <sp-dropdown id="aspectRatioPresetDropdown">
            <sp-menu-item value="auto">Auto</sp-menu-item>
            <sp-menu-item value="1:1">1:1</sp-menu-item>
            <sp-menu-item value="3:2">3:2</sp-menu-item>
            <sp-menu-item value="2:3">2:3</sp-menu-item>
            <sp-menu-item value="4:3">4:3</sp-menu-item>
            <sp-menu-item value="3:4">3:4</sp-menu-item>
            <sp-menu-item value="5:4">5:4</sp-menu-item>
            <sp-menu-item value="4:5">4:5</sp-menu-item>
            <sp-menu-item value="16:9">16:9</sp-menu-item>
            <sp-menu-item value="9:16">9:16</sp-menu-item>
            <sp-menu-item value="2:1">2:1</sp-menu-item>
            <sp-menu-item value="1:2">1:2</sp-menu-item>
            <sp-menu-item value="21:9">21:9</sp-menu-item>
            <sp-menu-item value="9:21">9:21</sp-menu-item>
        </sp-dropdown>
    </section>

    <section class="section-block">
        <sp-label>参考图片</sp-label>
        <div class="reference-list" id="referenceList"></div>
        <sp-button id="addReferenceButton">添加当前文档为参考图</sp-button>
        <sp-detail id="referenceCounter">0 / 8</sp-detail>
    </section>

    <section class="section-block">
        <sp-button id="generateButton" variant="accent">生成图像</sp-button>
        <sp-progressbar id="requestProgress" indeterminate hidden></sp-progressbar>
        <sp-body id="statusText">待命</sp-body>
    </section>

    <section class="section-block">
        <sp-label>结果</sp-label>
        <img id="previewImage" class="preview-image" />
        <sp-button id="insertButton">插入当前文档</sp-button>
        <sp-button id="saveButton">保存图片</sp-button>
    </section>
</div>
```

## 4. Flexbox CSS 基线

```css
.panel-root {
    display: flex;
    flex-direction: column;
    gap: 12px;
    min-width: 280px;
    padding: 12px;
}

.section-block {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.row {
    display: flex;
    flex-direction: row;
    gap: 8px;
    align-items: center;
}

.reference-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.reference-item {
    display: flex;
    flex-direction: row;
    gap: 8px;
    align-items: center;
}

.reference-thumb {
    width: 56px;
    height: 56px;
    object-fit: cover;
    border-radius: 6px;
}

.preview-image {
    width: 100%;
    max-height: 240px;
    object-fit: contain;
    border-radius: 6px;
}

sp-button,
sp-dropdown,
sp-textfield,
sp-textarea,
sp-slider {
    width: 100%;
}
```

## 5. 模型与参数矩阵

### 通用 Generations

接口：`POST /v1/images/generations`

Content-Type：`application/json`

基础参数：

| 参数 | UI 组件 | 说明 |
|---|---|---|
| `model` | `sp-dropdown` | 当前选择模型 |
| `prompt` | `sp-textarea` | 必填 |
| `size` | `sp-dropdown` 或自定义宽高输入 | 按模型显示合法尺寸 |
| `aspect_ratio` | `sp-dropdown` | 仅对支持该参数的模型显示 |
| `image` | `sp-button` 导入当前 Photoshop 文档 | 图生图参考图统一来自 `state.images` 队列，由请求适配层转换 |
| `response_format` | `sp-dropdown` | `url` 或 `b64_json`，仅对文档明确支持的模型显示；gpt-image-2 默认不发送 |
| `quality` | `sp-dropdown` | `auto`、`low`、`medium`、`high`，按模型限制 |

请求示例：

```json
{
    "model": "gpt-image-2",
    "prompt": "A cinematic product shot",
    "size": "1024x1024",
    "quality": "high"
}
```

### 通用 Edits

接口：`POST /v1/images/edits`

Content-Type：`multipart/form-data`

基础参数：

| 参数 | UI 组件 | 说明 |
|---|---|---|
| `image` | `sp-button` 导入当前 Photoshop 文档 | 必填，支持单图或多图由模型决定；内部统一来自 `state.images` 队列 |
| `prompt` | `sp-textarea` | 必填 |
| `mask` | `sp-button` 触发文件选择 | 可选，透明区域表示编辑区域 |
| `model` | `sp-dropdown` | 当前选择模型 |
| `n` | `sp-slider` 或 `sp-textfield` | 1-10，按模型限制 |
| `quality` | `sp-dropdown` | gpt-image 系列显示 |
| `response_format` | `sp-dropdown` | `url` 或 `b64_json`，仅对文档明确支持的模型显示；gpt-image-2 默认不发送 |
| `size` | `sp-dropdown` 或自定义宽高输入 | 按模型显示合法尺寸 |
| `aspect_ratio` | `sp-dropdown` | nano-banana 编辑兼容接口显示 |
| `image_size` | `sp-dropdown` | nano-banana 编辑兼容接口显示 |

## 6. 输出规格 UI 与接口字段映射

用户可以在 UI 中选择清晰度档位和画幅比例：

- 清晰度档位：`Auto`、`1K`、`2K`、`4K`
- 画幅比例：`Auto`、`1:1`、`3:2`、`2:3`、`4:3`、`3:4`、`5:4`、`4:5`、`16:9`、`9:16`、`2:1`、`1:2`、`21:9`、`9:21`

重要限制：

- 这两个 UI 控件不等于可以把 `image_size` 和 `aspect_ratio` 直接发给所有模型。
- 在这 4 份接口文档中，gpt-image-2 不支持请求字段 `image_size`，也不支持请求字段 `aspect_ratio`。
- gpt-image-2 必须把 UI 选择转换成合法的 `size` 字符串，例如 `1024x1024`、`2048x1152`、`3840x2160` 或符合约束的自定义宽高。
- Generations 通用接口如支持 `aspect_ratio`，可以按模型配置发送 `aspect_ratio`；但仍不发送未声明支持的 `image_size`。
- Edits 通用接口按模型配置发送 `size`，不把 `image_size` 当作通用字段。

### UI 状态字段

```js
const OUTPUT_SPEC = {
    resolutionTiers: ["auto", "1k", "2k", "4k"],
    aspectRatioPresets: [
        "auto",
        "1:1",
        "3:2",
        "2:3",
        "4:3",
        "3:4",
        "5:4",
        "4:5",
        "16:9",
        "9:16",
        "2:1",
        "1:2",
        "21:9",
        "9:21"
    ]
};

const state = {
    resolutionTier: "auto",
    aspectRatioPreset: "auto"
};
```

### gpt-image-2 尺寸转换

gpt-image-2 的尺寸规则：

- `Auto` 直接发送 `size: "auto"`。
- 最大边长不超过 `3840px`。
- 宽高都必须是 `16px` 的倍数。
- 长边与短边比例不超过 `3:1`。
- 总像素数必须在 `655360` 到 `8294400` 之间。

因此 `4K + 1:1` 不能直接做成 `3840x3840`，因为总像素超限。必须按像素上限自动收敛到合法尺寸，例如 `2880x2880`。

```js
function resolveGptImage2Size(resolutionTier, aspectRatioPreset) {
    if (resolutionTier === "auto" || aspectRatioPreset === "auto") {
        return "auto";
    }

    const longEdges = {
        "1k": 1024,
        "2k": 2048,
        "4k": 3840
    };

    const ratioParts = aspectRatioPreset.split(":").map((part) => Number(part));
    const ratioWidth = ratioParts[0];
    const ratioHeight = ratioParts[1];
    const ratio = ratioWidth / ratioHeight;
    const maxPixels = 8294400;
    const minPixels = 655360;
    const maxEdge = 3840;
    const preferredLongEdge = longEdges[resolutionTier] || 1024;

    let width = ratio >= 1 ? preferredLongEdge : preferredLongEdge * ratio;
    let height = ratio >= 1 ? preferredLongEdge / ratio : preferredLongEdge;

    if (width > maxEdge || height > maxEdge || width * height > maxPixels) {
        if (ratio >= 1) {
            width = Math.min(maxEdge, Math.sqrt(maxPixels * ratio));
            height = width / ratio;
        } else {
            height = Math.min(maxEdge, Math.sqrt(maxPixels / ratio));
            width = height * ratio;
        }
    }

    width = Math.max(16, Math.floor(width / 16) * 16);
    height = Math.max(16, Math.floor(height / 16) * 16);

    if (width * height < minPixels) {
        throw new Error("当前输出规格低于 gpt-image-2 最小像素限制");
    }

    return `${width}x${height}`;
}
```

### 模型字段策略

| 模型/接口 | UI 可选 1K/2K/4K | UI 可选比例 | 实际请求字段 |
|---|---|---|---|
| gpt-image-2 Generations | 可以 | 可以 | 转换为 `size`，不发送 `image_size`、`aspect_ratio` |
| gpt-image-2 Edits | 可以 | 可以 | 转换为 `size`，不发送 `image_size`、`aspect_ratio` |
| Generations 通用 | 可以 | 按模型配置 | 发送 `size`；仅支持时发送 `aspect_ratio` |
| Edits 通用 | 可以 | 按模型配置 | 发送 `size`；不把 `image_size` 当通用字段 |

## 7. 模型专属 UI 规则

### gpt-image-2

适用接口：

- `POST /v1/images/generations`
- `POST /v1/images/edits`

显示参数：

- `size`
- `quality`
- `image`

默认请求字段：

- `model`
- `prompt`
- `size`
- `image`
- `quality`

尺寸预设：

- `auto`
- `1024x1024`
- `1536x1024`
- `1024x1536`
- `2048x2048`
- `2048x1152`
- `3840x2160`
- `2160x3840`
- 自定义宽高

自定义尺寸校验：

- 最大边长不超过 `3840px`。
- 宽高都必须是 `16px` 的倍数。
- 长边与短边比例不超过 `3:1`。
- 总像素数必须在 `655360` 到 `8294400` 之间。

质量选项：

- `auto`
- `low`
- `medium`
- `high`

不显示为原生 API 参数：

- `aspect_ratio`
- `image_size`

说明：gpt-image-2 可以在 UI 上显示“1K/2K/4K + Auto/1:1/3:2 等比例”的输出规格控件，但最终必须转换成 `size` 字符串；不使用 nano-banana/Gemini 风格的 `aspect_ratio` 或 `image_size` 请求字段。`response_format` 在 gpt-image-2 中默认不发送，除非目标服务明确以兼容参数方式启用。

### gpt-image-1

Generations 与 Edits 都显示：

- `size`：`auto`、`1024x1024`、`1536x1024`、`1024x1536`
- `quality`：`auto`、`low`、`medium`、`high`
- `response_format`：默认建议 `b64_json`

Edits 限制：

- `image` 支持 PNG、WEBP、JPG。
- 单张图片小于 `25MB`。
- `prompt` 最大 `32000` 字符。

### dall-e-2

Edits 显示：

- `size`：`256x256`、`512x512`、`1024x1024`
- `n`：1-10
- `response_format`：`url`、`b64_json`
- `mask`

限制：

- `image` 必须是 1 张正方形 PNG，且小于 `4MB`。
- `mask` 必须与原图尺寸一致，PNG 且小于 `4MB`。
- `prompt` 最大 `1000` 字符。

### dall-e-3

Generations 显示：

- `size`：`1024x1024`、`1792x1024`、`1024x1792`
- `quality`：`standard`、`hd`
- `style`：`vivid`、`natural`
- `response_format`：建议 `b64_json`

### nano-banana-3.1-flash

模型值：`gemini-3.1-flash-image-preview`

Generations：

```json
{
    "prompt": "cat",
    "model": "gemini-3.1-flash-image-preview"
}
```

Edits 兼容：

- `model`
- `prompt`
- `image`，允许多张图片文件
- `response_format`
- `aspect_ratio`
- `image_size`

### flux-kontext 系列

Generations 中显示参考图区域。`flux-kontext-dev` 必须提供参考图。参考图不允许在 UI 中作为散乱 URL 直接输入，必须先进入统一的 `state.images` 队列，再由请求构建器转换为目标接口接受的格式；若兼容接口只能接收 URL，则先由上传适配器把 buffer 上传到可访问地址，再写入 `image` 数组。

## 8. 动态参数渲染

维护一个模型注册表，UI 根据注册表重建 `dynamicParams`。

```js
const MODEL_REGISTRY = {
    "gpt-image-2": {
        modes: ["generations", "edits"],
        sizes: ["auto", "1024x1024", "1536x1024", "1024x1536", "2048x2048", "2048x1152", "3840x2160", "2160x3840", "custom"],
        qualities: ["auto", "low", "medium", "high"],
        responseFormats: [],
        omitResponseFormatByDefault: true,
        supportsReferenceImages: true,
        maxReferenceImages: 8,
        referenceImageTransport: "urlArray",
        outputSpecMode: "gptImage2Size",
        resolutionTiers: ["auto", "1k", "2k", "4k"],
        aspectRatioPresets: ["auto", "1:1", "3:2", "2:3", "4:3", "3:4", "5:4", "4:5", "16:9", "9:16", "2:1", "1:2", "21:9", "9:21"],
        supportsMask: false,
        customSize: true
    },
    "gpt-image-1": {
        modes: ["generations", "edits"],
        sizes: ["auto", "1024x1024", "1536x1024", "1024x1536"],
        qualities: ["auto", "low", "medium", "high"],
        responseFormats: ["b64_json"],
        supportsReferenceImages: true,
        maxReferenceImages: 8,
        referenceImageTransport: "urlArray",
        supportsMask: true,
        outputSpecMode: "fixedSizeOnly",
        customSize: false
    },
    "dall-e-2": {
        modes: ["edits"],
        sizes: ["256x256", "512x512", "1024x1024"],
        responseFormats: ["url", "b64_json"],
        supportsReferenceImages: true,
        maxReferenceImages: 1,
        referenceImageTransport: "multipartOnly",
        supportsMask: true,
        outputSpecMode: "fixedSizeOnly",
        maxPromptLength: 1000
    },
    "gemini-3.1-flash-image-preview": {
        modes: ["generations", "edits"],
        sizes: [],
        responseFormats: ["url", "b64_json"],
        aspectRatios: ["", "1:1", "4:3", "3:4", "16:9", "9:16"],
        imageSizes: ["", "1K", "2K", "4K"],
        supportsReferenceImages: true,
        maxReferenceImages: 8,
        referenceImageTransport: "inlineData",
        outputSpecMode: "nativeImageConfig",
        supportsMask: false
    }
};
```

动态渲染规则：

- 模型切换时，重新计算可选模式；当前模式不支持时自动切到第一个可用模式。
- 模式切换时，重新计算请求类型：Generations 使用 JSON，Edits 使用 FormData。
- 不支持的参数直接隐藏，并从请求体中移除。gpt-image-2 不显示原生 `aspect_ratio`、`image_size` 参数，也不发送这两个字段；只显示输出规格控件并转换为 `size`。
- 有默认值的参数在隐藏时也不发送，避免模型收到无意义空字符串。
- `promptCounter` 根据模型最大长度更新。

## 9. 图生图参考图上传规则

所有图生图相关调用统一参考 `F:\桌面\Photoshop-ai-all` 插件的做法。核心原则是：UI 不直接把参考图做成临时 URL 字符串传给业务层，而是把 Photoshop 当前文档导出为临时 JPG，读取 binary buffer，保存到 `state.images`，请求时再按接口类型转换。

### 统一状态结构

```js
const MAX_REFERENCE_IMAGES = 8;

const state = {
    images: []
};

function createReferenceImageState(entry) {
    return {
        name: entry.name,
        url: entry.url,
        data: entry.buffer,
        width: entry.width,
        height: entry.height,
        mimeType: "image/jpeg"
    };
}
```

`state.images` 中每个对象必须包含：

| 字段 | 说明 |
|---|---|
| `name` | 原 Photoshop 文档名或生成的参考图名称 |
| `url` | `URL.createObjectURL` 创建的面板预览地址 |
| `data` | `ArrayBuffer`，API 请求构建时唯一可信源 |
| `width` | 参考图宽度 |
| `height` | 参考图高度 |
| `mimeType` | 默认 `image/jpeg` |

### 从当前 Photoshop 文档导入参考图

```js
async function importActiveDocumentAsReference() {
    if (state.images.length >= MAX_REFERENCE_IMAGES) {
        throw new Error(`最多只能导入 ${MAX_REFERENCE_IMAGES} 张参考图`);
    }

    let result = null;

    await core.executeAsModal(async () => {
        const document = app.activeDocument;
        if (!document) {
            throw new Error("请先打开一个 Photoshop 文档");
        }

        const fs = storage.localFileSystem;
        const tempFolder = await fs.getTemporaryFolder();
        const safeTitle = document.title ? document.title.split(".")[0] : "ps_image";
        const tempFile = await tempFolder.createFile(`${safeTitle}_${Date.now()}.jpg`, { overwrite: true });

        await document.saveAs.jpg(tempFile, { quality: 10 }, true);

        const buffer = await tempFile.read({ format: storage.formats.binary });
        await tempFile.delete();

        result = {
            name: document.title || tempFile.name,
            buffer
        };
    }, { commandName: "导入当前文档为参考图" });

    const blob = new Blob([result.buffer], { type: "image/jpeg" });
    const url = URL.createObjectURL(blob);
    const dimensions = await getImageDimensionsFromBuffer(result.buffer);

    state.images.push(createReferenceImageState({
        name: result.name,
        url,
        buffer: result.buffer,
        width: dimensions.width,
        height: dimensions.height
    }));
}
```

### 删除参考图

删除时必须释放预览 URL：

```js
function removeReferenceImage(index) {
    const image = state.images[index];
    if (!image) {
        return;
    }

    URL.revokeObjectURL(image.url);
    state.images.splice(index, 1);
}
```

### binary 转 base64

参考插件使用 `ArrayBuffer -> Uint8Array -> btoa` 的转换方式。当前插件继续沿用，用于 JSON inline image 请求。

```js
function bufferToBase64(buffer) {
    let binary = "";
    const bytes = new Uint8Array(buffer);

    for (let i = 0; i < bytes.byteLength; i += 1) {
        binary += String.fromCharCode(bytes[i]);
    }

    return btoa(binary);
}
```

### 请求适配策略

| 目标接口 | 参考图转换方式 |
|---|---|
| Gemini / nano-banana 兼容 JSON | `state.images` 按顺序转换为 `inline_data` |
| OpenAI Edits multipart | `state.images` 转 Blob，逐张 `form.append("image", blob, name)` |
| 只接受 URL 的兼容模型 | 不属于这 4 份接口文档的核心实现；必须由用户另行配置外部文件托管服务后才可启用 |
| Prompt 拼接型兼容模型 | 仅作为最后兼容方案，仍然先从 `state.images` 转换 |

UI 可以提供“添加当前文档为参考图”和“删除参考图”两个动作；参考图数量、顺序、预览都只以 `state.images` 为准。

对于本地 Photoshop 文档导入的参考图，优先使用 `/v1/images/edits` 的 multipart 上传方式。若选择 `/v1/images/generations` 且目标模型只接受图片 URL，则必须先有外部文件托管服务生成可访问 URL；否则 UI 应禁用该组合并提示用户切换到 Edits。

## 10. 请求构建规则

### Generations JSON

```js
async function buildGenerationsPayload(state, modelConfig) {
    const payload = {
        model: state.model,
        prompt: state.prompt
    };

    if (modelConfig.outputSpecMode === "gptImage2Size") {
        payload.size = resolveGptImage2Size(state.resolutionTier, state.aspectRatioPreset);
    } else if (state.size && state.size !== "custom") {
        payload.size = state.size;
    }

    if (modelConfig.outputSpecMode !== "gptImage2Size" && state.customSize) {
        payload.size = `${state.customWidth}x${state.customHeight}`;
    }

    if (state.quality) {
        payload.quality = state.quality;
    }

    if (!modelConfig.omitResponseFormatByDefault && state.responseFormat) {
        payload.response_format = state.responseFormat;
    }

    if (modelConfig.aspectRatios && state.aspectRatio) {
        payload.aspect_ratio = state.aspectRatio;
    }

    const referenceImages = await buildGenerationReferenceImages(state, modelConfig);
    if (referenceImages) {
        payload.image = referenceImages;
    }

    return payload;
}

async function buildGenerationReferenceImages(state, modelConfig) {
    if (!state.images || state.images.length === 0) {
        return null;
    }

    if (modelConfig.referenceImageTransport === "inlineData") {
        return state.images.map((image) => ({
            inline_data: {
                mime_type: image.mimeType,
                data: bufferToBase64(image.data)
            }
        }));
    }

    if (modelConfig.referenceImageTransport === "dataUrlArray") {
        return state.images.map((image) => {
            return `data:${image.mimeType};base64,${bufferToBase64(image.data)}`;
        });
    }

    if (modelConfig.referenceImageTransport === "urlArray") {
        return uploadReferenceImagesAndReturnUrls(state.images);
    }

    return null;
}
```

### Edits FormData

```js
function buildEditsFormData(state, modelConfig) {
    const form = new FormData();
    form.append("model", state.model);
    form.append("prompt", state.prompt);

    for (const image of state.images) {
        const blob = new Blob([image.data], { type: image.mimeType });
        form.append("image", blob, image.name);
    }

    if (state.maskFile) {
        form.append("mask", state.maskFile);
    }

    if (modelConfig.outputSpecMode === "gptImage2Size") {
        form.append("size", resolveGptImage2Size(state.resolutionTier, state.aspectRatioPreset));
    } else if (state.size && state.size !== "custom") {
        form.append("size", state.size);
    }

    if (modelConfig.outputSpecMode !== "gptImage2Size" && state.customSize) {
        form.append("size", `${state.customWidth}x${state.customHeight}`);
    }

    if (state.quality) {
        form.append("quality", state.quality);
    }

    if (!modelConfig.omitResponseFormatByDefault && state.responseFormat) {
        form.append("response_format", state.responseFormat);
    }

    if (modelConfig.aspectRatios && state.aspectRatio) {
        form.append("aspect_ratio", state.aspectRatio);
    }

    if (modelConfig.imageSizes && state.imageSize) {
        form.append("image_size", state.imageSize);
    }

    return form;
}
```

## 11. 文件处理

- 导入参考图、选择 mask、保存图片必须使用 `storage.localFileSystem`。
- 参考图主入口是“添加当前文档为参考图”，实现方式必须与 `F:\桌面\Photoshop-ai-all` 一致：当前文档保存为临时 JPG、读取 binary、删除临时文件、写入 `state.images`。
- 如果某个非核心兼容模型只接受 URL 类型参考图，不允许在文档中假设存在 `/v1/files/upload` 之类未被接口文档定义的端点。必须由用户显式配置外部文件托管服务后，才能启用 URL 适配器。
- API Key 可存入插件 data folder 或 localStorage；推荐 data folder 中保存配置文件，明文风险需在 UI 中提示用户。
- 返回 `b64_json` 时，将 base64 转为 Blob，再写入 UXP 临时文件。
- 返回 `url` 时，先 fetch 下载为 Blob，再写入 UXP 临时文件，避免 URL 过期导致 Photoshop 后续无法导入。

URL 上传适配器不属于这 4 份接口文档的必需能力，只能作为外部配置扩展点。核心 Dall-e/gpt-image-2 调用不得依赖未声明的上传端点：

```js
async function uploadReferenceImagesAndReturnUrls(images) {
    if (!state.assetUploadUrl) {
        throw new Error("当前模型需要图片 URL，但尚未配置外部文件托管服务");
    }

    const urls = [];

    for (const image of images) {
        const form = new FormData();
        const blob = new Blob([image.data], { type: image.mimeType });
        form.append("file", blob, image.name);

        const response = await fetch(state.assetUploadUrl, {
            method: "POST",
            headers: {
                Authorization: `Bearer ${state.apiKey}`
            },
            body: form
        });

        if (!response.ok) {
            const message = await response.text();
            throw new Error(`参考图上传失败: ${message}`);
        }

        const result = await response.json();
        urls.push(result.url);
    }

    return urls;
}
```

## 12. 生图结果解析规则

获取生图结果的方式统一参考 `F:\桌面\Photoshop-ai-all`。无论接口返回字段名称是 `inlineData`、`inline_data`、`image`、`Image`、`data`、`b64_json` 还是远程 `url`，业务层都必须先归一化成 `ArrayBuffer`，然后再创建预览 URL 和写入 `state.resultImage`。

### 结果状态结构

```js
const state = {
    resultImage: null
};

function setResultImage(buffer, mimeType) {
    if (state.resultImage && state.resultImage.url) {
        URL.revokeObjectURL(state.resultImage.url);
    }

    const blob = new Blob([buffer], { type: mimeType || "image/png" });
    const url = URL.createObjectURL(blob);

    state.resultImage = {
        url,
        data: buffer,
        mimeType: mimeType || "image/png"
    };
}
```

### base64 转 binary

参考插件使用 `atob -> Uint8Array -> ArrayBuffer`。当前插件继续沿用，作为所有 base64 图片响应的统一解码方法。

```js
function base64ToBuffer(base64) {
    const binaryString = atob(base64);
    const bytes = new Uint8Array(binaryString.length);

    for (let i = 0; i < binaryString.length; i += 1) {
        bytes[i] = binaryString.charCodeAt(i);
    }

    return bytes.buffer;
}
```

### 兼容 Gemini / nano-banana 响应

```js
function extractGeminiImageBuffer(result) {
    if (!result.candidates || result.candidates.length === 0) {
        throw new Error("API 返回结果为空");
    }

    const candidate = result.candidates[0];
    if (!candidate.content || !candidate.content.parts) {
        throw new Error("API 返回格式错误");
    }

    for (const part of candidate.content.parts) {
        const inlineData = part.inlineData || part.inline_data;
        const imageData = part.image || part.Image;

        if (inlineData && (inlineData.data || inlineData.buffer)) {
            return base64ToBuffer(inlineData.data || inlineData.buffer);
        }

        if (imageData && (imageData.data || imageData.buffer)) {
            return base64ToBuffer(imageData.data || imageData.buffer);
        }

        if (part.data) {
            return base64ToBuffer(part.data);
        }
    }

    throw new Error("未找到生成的图片");
}
```

### 兼容 OpenAI Dall-e 响应

```js
async function extractOpenAIImageBuffer(result) {
    if (!result.data || result.data.length === 0) {
        throw new Error("API 返回结果为空");
    }

    const item = result.data[0];

    if (item.b64_json) {
        return base64ToBuffer(item.b64_json);
    }

    if (item.url) {
        const response = await fetch(item.url);
        if (!response.ok) {
            throw new Error(`下载生成结果失败: ${response.status}`);
        }

        return response.arrayBuffer();
    }

    throw new Error("未找到生成的图片");
}
```

### 生成流程

```js
async function handleGenerate() {
    state.isGenerating = true;
    state.resultImage = null;
    render();

    try {
        const result = await callImageApi();
        const buffer = await extractImageBufferByModel(result, state.model);
        setResultImage(buffer, "image/png");
    } catch (error) {
        state.errorMessage = `生成失败: ${error.message || error}`;
        throw error;
    } finally {
        state.isGenerating = false;
        render();
    }
}
```

`extractImageBufferByModel` 只负责分发到 Gemini 兼容解析器、OpenAI Dall-e 兼容解析器或其他模型专属解析器。UI 预览、保存、插入 Photoshop 都只读取 `state.resultImage.data`。

## 13. 插入 Photoshop 文档

插入 Photoshop 的方式统一参考 `F:\桌面\Photoshop-ai-all`：

1. 只接收 `state.resultImage.data` 里的 binary buffer。
2. 将 buffer 写入 UXP 临时 PNG 文件。
3. 在 `core.executeAsModal` 中执行 Photoshop 修改。
4. 如果当前有活动文档，使用 `action.batchPlay` 的 `placeEvent` 置入临时文件。
5. 置入后获取新图层边界，按画布尺寸计算缩放比例，保持比例并填满画布，再移动到画布左上角。
6. 如果没有活动文档，直接 `app.open(tempFile)` 打开生成图。
7. 无论成功失败，最后删除临时文件。

示意：

```js
async function insertResultIntoPhotoshop(buffer) {
    if (!buffer) {
        throw new Error("没有可插入的生成结果");
    }

    const fs = storage.localFileSystem;
    const tempFolder = await fs.getTemporaryFolder();
    const tempFile = await tempFolder.createFile(`ai_result_${Date.now()}.png`, { overwrite: true });

    await tempFile.write(buffer, { format: storage.formats.binary });

    try {
        await core.executeAsModal(async () => {
            const activeDocument = app.activeDocument;

            if (activeDocument) {
                const fileToken = await fs.createSessionToken(tempFile);

                await action.batchPlay([{
                    _obj: "placeEvent",
                    null: {
                        _path: fileToken,
                        _kind: "local"
                    }
                }], { synchronousExecution: true });

                const newLayer = activeDocument.activeLayers[0];
                const bounds = newLayer.bounds;
                const boundsWidth = bounds.right - bounds.left;
                const boundsHeight = bounds.bottom - bounds.top;
                const docWidth = activeDocument.width;
                const docHeight = activeDocument.height;
                const scaleX = docWidth / boundsWidth;
                const scaleY = docHeight / boundsHeight;
                const scalePercent = Math.max(scaleX, scaleY) * 100;

                await newLayer.scale(scalePercent, scalePercent);

                const newBounds = newLayer.bounds;
                await newLayer.translate(-newBounds.left, -newBounds.top);
            } else {
                await app.open(tempFile);
            }
        }, { commandName: "插入并缩放 AI 生成图像" });
    } finally {
        try {
            await tempFile.delete();
        } catch (error) {
            console.warn("清理临时图片失败", error);
        }
    }
}
```

## 14. 面板状态

| 状态 | UI 表现 |
|---|---|
| idle | 生成按钮可用，状态文字显示待命 |
| validating | 状态文字显示正在检查参数 |
| requesting | 显示 `sp-progressbar`，禁用生成按钮 |
| preview | `state.resultImage.url` 显示预览图，启用插入与保存按钮 |
| inserting | 将 `state.resultImage.data` 传入 `insertResultIntoPhotoshop`，禁用插入按钮，执行 `executeAsModal` |
| error | 显示错误信息，保留用户输入 |

## 15. 参数校验

提交前必须检查：

- API Key 不能为空。
- Base URL 不能为空，末尾斜杠统一清理。
- Prompt 不能为空，且不超过模型限制。
- 所有图生图与 Edits 模式必须至少有一张 `state.images` 参考图。
- dall-e-2 Edits 的图片必须是单张正方形 PNG，小于 `4MB`。
- gpt-image-1 Edits 的图片格式必须是 PNG、WEBP 或 JPG，单张小于 `25MB`。
- gpt-image-2 自定义尺寸必须通过尺寸规则。
- `flux-kontext-dev` 必须有参考图。

## 16. 开发文件建议

```text
manifest.json
index.html
src/
  main.js
  ui/
    renderParams.js
    state.js
  api/
    imageApi.js
    payloadBuilder.js
    resultParser.js
  photoshop/
    insertImage.js
  storage/
    configStore.js
styles/
  panel.css
```

职责：

- `main.js`：初始化事件绑定、调度生成流程。
- `renderParams.js`：根据模型注册表渲染参数控件。
- `state.js`：集中管理当前模型、模式、参数和文件引用。
- `imageApi.js`：封装 fetch、HTTP 错误处理和原始 JSON 响应返回。
- `payloadBuilder.js`：生成 JSON 或 FormData。
- `resultParser.js`：参考 `F:\桌面\Photoshop-ai-all`，把 `inlineData`、`image`、`data`、`b64_json`、`url` 等响应统一解析成 `ArrayBuffer`。
- `insertImage.js`：只接收生成结果 buffer，写入临时 PNG，并在 `executeAsModal` 中通过 `placeEvent` 插入、缩放、移动。
- `configStore.js`：保存 Base URL、API Key、上次模型选择。

## 17. 验收清单

- 面板无 CEP、CSInterface、ExtendScript。
- 文件处理无 Node.js 原生 `fs`、`path`。
- CSS 无 `display: grid`、`float`、`position: fixed`、伪元素。
- 所有可见文本在 SWC 文本组件中。
- 所有布局容器使用 Flexbox。
- 模型切换后参数控件自动刷新。
- 不支持的参数不会出现在请求体中。
- 所有图生图参考图都从当前 Photoshop 文档导入，进入 `state.images` 后再转换请求格式。
- UI 中不出现独立参考图 URL 输入作为主流程。
- 所有生图响应都先解析为 `ArrayBuffer`，再写入 `state.resultImage = { url, data, mimeType }`。
- 插入 Photoshop 时只传递 `state.resultImage.data`，并复用临时 PNG、`placeEvent`、自适应缩放、移动到左上角的流程。
- Generations 使用 JSON。
- Edits 使用 FormData。
- Photoshop 文档修改 100% 包裹在 `core.executeAsModal`。
- API 错误、网络错误、参数错误都能显示到状态区域。
- 生成结果可预览、可保存、可插入当前文档。

## 18. 参考文档

- Adobe Photoshop UXP Guides: https://developer.adobe.com/photoshop/uxp/2022/guides/
- Adobe Photoshop PS Reference: https://developer.adobe.com/photoshop/uxp/2022/ps-reference/
- Adobe UXP JS API: https://developer.adobe.com/photoshop/uxp/2022/uxp-api/reference-js/
- Adobe Spectrum Web Components: https://opensource.adobe.com/spectrum-web-components/
- UXP Spectrum Web Components: https://developer.adobe.com/photoshop/uxp/2022/uxp-api/reference-spectrum/swc/
- Dalle 格式介绍: https://gpt-best.apifox.cn/doc-7414819
- Generations 通用: https://gpt-best.apifox.cn/api-302915860
- Edits 通用: https://gpt-best.apifox.cn/api-288978020
- Nano-banana-3.1-Flash Generations: https://gpt-best.apifox.cn/api-420646217
- Nano-banana-3.1-Flash Edits: https://gpt-best.apifox.cn/api-420646219
- gpt-image-2 Edits: https://gpt-best.apifox.cn/api-447258891
- gpt-image-2 Generations: https://gpt-best.apifox.cn/api-447261009

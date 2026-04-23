# TLC 數位樣冊 2.0 — 技術交接文件 (MIS Handoff)

> 給 MIS 同仁：Clone 這個 repo 後，你的 Claude Code 會自動讀到這份文件作為 context。
> 建議指令：`claude "請閱讀 HANDOFF.md 並幫我把 3D 模組整合進現有平台"`

---

## 專案概述

**目標**：在現有布料庫平台（`fabriclibrary.tiongliong.com`）加入 3D 鞋款試色功能。
業務在展覽現場用平板，讓客戶即時看到「這塊布料套在鞋子上的效果」。

**參考效果**：`thetexperts.com/shirting/`（同類型的 3D 布料模擬器）

**原型展示**：
```
https://sugimotofang-spec.github.io/notebooklm-outputs/slides/2026-04-23-數位樣冊2.0概念原型-slides.html
```

---

## Repo 結構

```
notebooklm-outputs/
├── CLAUDE.md                          ← Claude Code 工作規則
├── HANDOFF.md                         ← 本文件
├── slides/
│   └── 2026-04-23-數位樣冊2.0概念原型-slides.html   ← 原型（單一 HTML 檔）
├── infographics/
├── audio/
└── ...
```

原型是**單一 HTML 檔**，所有 CSS / JS 都內嵌，方便 GitHub Pages 部署與攜帶展示。

---

## 3D 模組技術說明

### 架構
```
importmap (Three.js r0.160 via jsDelivr CDN)
    ↓
<script type="module">
    ├── GLTFLoader  → 載入 MaterialsVariantsShoe.glb（Khronos 免費模型）
    ├── OrbitControls → 滑鼠拖曳旋轉 / 滾輪縮放 / 自動旋轉
    ├── genTex()    → Canvas 生成布料紋理（woven / knit / mesh）
    └── applyFabData() → 把布料材質套到 3D 鞋面
```

### 材質分類（Y 座標法）
```javascript
// 鞋子載入後，依世界座標 Y 位置分類各 mesh：
const relY = (meshCenterY - worldBox.min.y) / worldHeight;

if (relY < 0.20)           → 鞋底（套 off-white #f4f2ee）
else if (name includes 'lace') → 鞋帶（保持原色）
else                       → 鞋面上層（套布料貼圖）
```

### 事件介面（非 module ↔ module 溝通）
```javascript
// 選布料時觸發（在現有平台裡，你只需要 dispatch 這個事件）
document.dispatchEvent(new CustomEvent('tlcFabricChanged', {
  detail: {
    id: 'EFPQT010',
    color: '#7a9bb5',     // HEX 主色
    tex: 'tx-knit',       // tx-woven / tx-knit / tx-mesh
    name: 'WEFT KNIT'
  }
}));

// 切換到 3D tab 時觸發
document.dispatchEvent(new CustomEvent('tlcTab3dOpen', {
  detail: { id, color, tex, name }
}));
```

---

## 整合到現有平台的步驟

### Step 1：MIS 需要提供
```
① 布料資料 API endpoint：
   GET /api/fabrics/{code}
   Response: { code, name, type, color, tileImageUrl, component, width }

② 後端開放 CORS（Three.js 跨域讀取貼圖需要）：
   Access-Control-Allow-Origin: *

③ 把 _tile.png 貼圖放到可公開存取的 URL
```

### Step 2：從原型抽出 3D 模組

原型 HTML 裡有三段需要搬移：

**① CSS（搜尋 `THREE.JS 3D VIEWER`）**
```css
/* 從 slides/2026-04-23-...html 搜尋這段 */
/* ── THREE.JS 3D VIEWER ── */
#three-canvas { ... }
.v3d-ctrl-panel { ... }
/* 約 20 行 */
```

**② HTML 結構**
```html
<div class="stage3d" id="stage3d">
  <canvas id="three-canvas"></canvas>
  <div class="v3d-loading" id="v3d-loading">...</div>
  <div class="v3d-ctrl-panel">...</div>
  <div class="v3d-hint">...</div>
  <div class="v3d-fab-badge" id="v3d-fab-badge"></div>
</div>
```

**③ JS Module（HTML 最底部 `<script type="module">`）**
整段搬移，約 120 行。

### Step 3：修改 JS 接入真實資料

目前用 Canvas 假紋理，接入真實掃描圖只需改一處：

```javascript
// 現在（Canvas 假紋理）：
function applyFabData(f) {
  const cv = genTex(f.color, type, v3d.scale);
  const tex = new THREE.CanvasTexture(cv);
  ...
}

// 改成（真實掃描圖）：
function applyFabData(f) {
  const loader = new THREE.TextureLoader();
  loader.load(f.tileImageUrl,
    tex => {
      tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
      tex.repeat.set(3, 3);
      tex.colorSpace = THREE.SRGBColorSpace;
      upperMats.forEach(m => {
        if (m.map) m.map.dispose();
        m.map = tex;
        m.color.set(0xffffff);
        m.roughness = v3d.roughness;
        m.metalness = v3d.metalness;
        m.needsUpdate = true;
      });
    },
    undefined,
    () => applyFabData_canvas(f)  // 圖片載入失敗時 fallback 到 Canvas
  );
}
```

---

## 布料掃描圖規格（1500 張 RAW 處理）

| 用途 | 格式 | 尺寸 | 目標大小 |
|------|------|------|---------|
| 詳細頁展示 | JPG q90 | 2048×2048 | ~1–2 MB |
| **3D 貼圖** | **PNG** | **1024×1024** | **~500 KB** |

### 命名規則
```
EFRNT001_hires.jpg
EFRNT001_tile.png     ← 3D 用（需做無縫處理）
EFQQT002_tile.png
...
```

### 無縫處理（重要）
不做無縫處理，3D 鞋面會出現明顯接縫。
- Photoshop：濾鏡 → 其他 → 位移（X/Y 各 50%）→ 仿製印章修補
- 自動工具：Materialize（免費）
- 批次處理：Python + Pillow 腳本（需要的話可請 Claude Code 生成）

---

## 分工建議

| 項目 | sugimoto | MIS |
|------|----------|-----|
| ShoeViewer.js 獨立模組 | ✅ 負責 | — |
| 後端 API + CORS | — | ✅ 負責 |
| 貼圖上傳 / CDN | — | ✅ 負責 |
| 嵌入現有詳細頁 | 提供程式碼 | ✅ 整合 |
| 掃描圖無縫處理 | ✅ 流程規劃 | — |

---

## Git 協作流程

```bash
# MIS：clone repo
git clone https://github.com/sugimotofang-spec/notebooklm-outputs.git

# 建立整合分支
git checkout -b feature/3d-integration

# 開發完成後送 PR，sugimoto review
```

---

## 快速啟動（給 MIS 的 Claude Code 指令）

Clone repo 後，在 Claude Code 輸入：
```
請閱讀 HANDOFF.md，我要把 3D 鞋款展示功能整合進 fabriclibrary.tiongliong.com。
我們的後端是 [Django / Laravel / Node.js]，請協助我：
1. 把 Three.js 3D 模組從 slides/ 裡的 HTML 抽出成獨立 JS 檔
2. 修改成讀取真實貼圖 URL
3. 整合到現有布料詳細頁
```

---

## 聯絡資源

- Three.js 文件：https://threejs.org/docs/
- Khronos 鞋款模型：https://github.com/KhronosGroup/glTF-Sample-Assets
- 原型 Live Demo：https://sugimotofang-spec.github.io/notebooklm-outputs/slides/2026-04-23-數位樣冊2.0概念原型-slides.html

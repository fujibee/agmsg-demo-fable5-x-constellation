# AI Constellation Map — 詳細設計

*原文(英語)は [DESIGN.md](DESIGN.md)。この日本語訳は参照用です。*

**プロジェクト:** 「X が AI モデルについて語っていること」をインタラクティブな 3D 星座マップとして描画する。
**成果物:** three.js を使った単一の自己完結型 HTML ファイル(`index.html`)。ドラッグで回転、スクロールでズーム、星をクリックするとその背後にあるポストが読める。
**著者:** fable(設計のみ — 実装は opus、データは grok、磨き込みは codex)

---

## 1. コンセプト

メタファーは夜空。各 **AI モデルが星座**、X 上で議論されている各 **トピックが星**。星のサイズは言及ボリュームを、星のパルス/輝度はそのトピックが今どれだけホットかを表し、細い星座線が 1 つのモデルの星々を認識可能な図形につなぐ。空全体はユーザーが掴むまでゆっくり自動回転する。

星座は 5 つ:

| id | name | vendor | color (hex) | color rationale |
|----|------|--------|-------------|-----------------|
| `fable-5` | Fable 5 | Anthropic | `#E8865A` | ウォームコーラル — Anthropic のクレイ色、主役の温かみ |
| `opus-4-8` | Opus 4.8 | Anthropic | `#F2C14E` | アンバーゴールド — コーラルの兄弟色だが明確に区別可能 |
| `gpt-5-6` | GPT-5.6 | OpenAI | `#5CD6B9` | ティールグリーン — OpenAI の系譜 |
| `gemini-3` | Gemini 3 | Google | `#8B7CF6` | バイオレット — Gemini のグラデーション感 |
| `glm-5-2` | GLM 5.2 | Zhipu | `#4E9CFF` | クリアブルー — 一目でバイオレットと区別できる |

これら 5 色は意図的に色相環上に分散させている(オレンジ / ゴールド / ティール / バイオレット / ブルー)ので、ラベルを読まなくても周辺視野でクラスタを識別できる。

---

## 2. 画面レイアウト

フルビューポートの WebGL キャンバス。UI はすべて absolute 配置でオーバーレイした HTML。暗い空の上に暗い UI。

```
┌──────────────────────────────────────────────────────────────┐
│ ◤ AI CONSTELLATION                            ● Fable 5      │
│   What X is talking about · 2026-07-03        ● Opus 4.8     │
│   (top-left, title block)                     ● GPT-5.6      │
│                                               ● Gemini 3     │
│                                               ● GLM 5.2      │
│                                               (top-right,    │
│                  ✦     ·    ✦                  legend chips) │
│              ✦ ── ✦ ── ✦        ┌─────────────────────────┐  │
│                 CANVAS          │  DETAIL PANEL (hidden    │  │
│           ✦ ·    ✦    · ✦       │  until a star is clicked)│  │
│                                 │  360px, slides in right  │  │
│                                 └─────────────────────────┘  │
│ drag to rotate · scroll to zoom · click a star    (bottom-left)│
└──────────────────────────────────────────────────────────────┘
```

### 2.1 タイトルブロック(左上、28px インセット)
- 1 行目: `AI CONSTELLATION` — 13px、letter-spacing 0.3em、大文字、`rgba(255,255,255,0.55)`。
- 2 行目: `What X is talking about right now` — 22px、weight 600、`#F5F3EE`。
- 3 行目: `data.generatedAt` から取得した日付 — 12px、`rgba(255,255,255,0.35)`。
- フォントスタック: `Inter, "Helvetica Neue", system-ui, sans-serif`(Web フォントのダウンロードなし。システムフォールバックで十分)。

### 2.2 凡例(右上、28px インセット)
モデルごとに 1 チップ、縦積み、8px ギャップ:
- チップ: flex 行、10px のカラードット(モデル色、`box-shadow: 0 0 8px <color>`)+ モデル名 13px `rgba(255,255,255,0.75)`。
- ホバー: 背景 `rgba(255,255,255,0.06)`、radius 8px。
- クリック: そのクラスタへカメラをフライさせる(3D 上のラベルクリックと同じ挙動)。
- ダブルクリック: 減光トグル — 他のクラスタ(星、線、ラベル)を 15% 不透明度に落とす。もう一度ダブルクリックで復元。

### 2.3 詳細パネル(右側)
- 幅 360px、`top:88px; bottom:24px; right:24px`、border-radius 16px。
- ガラス表現: `background: rgba(12,16,28,0.72); backdrop-filter: blur(16px); border: 1px solid rgba(255,255,255,0.08)`。
- 非表示状態: `transform: translateX(400px)`。表示: `translateX(0)`。トランジション `transform 380ms cubic-bezier(0.22,1,0.36,1)`。
- コンテンツは上から下へ:
  1. モデルチップ(ドット + vendor · model name、12px、モデル色)。
  2. トピックラベル — 20px weight 700 `#F5F3EE`。
  3. メタ行: `▲ 1,250 mentions` + ヒートメーター(60px バー、fill = `heat`、モデル色から白へのグラデーション、`border-radius 999px`)。
  4. サマリー — 14px/1.6 `rgba(255,255,255,0.72)`。
  5. `POSTS` 区切り — 11px 大文字 letter-spacing 0.2em `rgba(255,255,255,0.35)`。
  6. ポストカード(ポストごとに 1 枚): `background rgba(255,255,255,0.04); border-radius 12px; padding 12px`。著者 `@handle` 13px weight 600 モデル色。本文 13px/1.55 `rgba(255,255,255,0.8)`。フッター `♥ 3.4k · ⇄ 900` 11px `rgba(255,255,255,0.4)`。カード全体を `post.url` への `<a target="_blank">` にする。
  7. 閉じるボタン `×` はパネル右上、ヒットエリア 28px。

### 2.4 ヒントバー(左下、24px インセット)
`drag to rotate · scroll to zoom · click a star` — 12px `rgba(255,255,255,0.35)`。キャンバス上での最初の pointerdown 後にフェードアウト(opacity 0、1s トランジション)。

### 2.5 ホバーツールチップ
カーソルに追従する小さなフローティングラベル(オフセット +14px,+14px): トピックラベル、12px、ガラスのピル(`rgba(12,16,28,0.85)`、radius 999px、padding 6px 12px、1px ボーダーはモデル色 40%)。星ホバー時のみ表示。

---

## 3. 空(背景)

レイヤー構成、低コスト、雰囲気重視:

1. **クリアカラー** `#05070D`(純黒ではなく — 深いネイビーの方がリッチに見える)。
2. **星雲グラデーションドーム**: `THREE.SphereGeometry(1200, 32, 32)` に `BackSide` の ShaderMaterial — `#0A0E1E`(下)から `#05070D`(中間)を経て `#0B0A18`(上)への垂直グラデーション。加えてフラグメントシェーダーに焼き込んだごく淡い放射状のカラーブルームを 2 つ(片方の極方向にバイオレット `#1A1440` を 6%、反対側にティール `#0E2A33` を 4%)。テクスチャのダウンロードは不要。
3. **背景星空**: `THREE.Points` クラウドに 2600 点、位置は半径 700–1000 のシェル上に一様分布。サイズ 0.6–1.8、色は白にわずかな青/暖色の色相バリエーション(±8% hue)、不透明度 0.35–0.9。`sizeAttenuation: true`、加算ブレンディング、depthWrite false。これらは決してレイキャストしない(`raycast = noop` を設定するか、非インタラクティブな別グループに置く)。
4. **フォグ**: `scene.fog = new THREE.Fog(0x05070D, 500, 1100)` — ドームと遠方の星を柔らかくする。

---

## 4. 星(トピック)

### 4.1 ジオメトリとマテリアル
各トピックの星 = 共有のキャンバス生成テクスチャを持つ `THREE.Sprite`、スプライトごとに色/スケールを指定。ここではスプライトがメッシュに勝る: 完璧なビルボード、安価なグロー、星 ≤40 個なら星ごとに 1 ドローコールでも許容範囲。

**星テクスチャ** — `createStarTexture()`: 256×256 キャンバス、放射状グラデーション:
- 0.00–0.08: `rgba(255,255,255,1)`(白熱コア)
- 0.08–0.30: `rgba(255,255,255,0.9)` → 色をティント可能なフォールオフ(純白で描画し、`sprite.material.color` でティント)
- 0.30–1.00: 透明への指数的フォールオフ(`alpha = pow(1-t, 2.5)`)
- 加えて回折スパイク 4 本: 交差する 2 本の線(水平/垂直)、幅 2px、グラデーションアルファ 0.35 → 0、長さは半径の 0.9 倍。これが「望遠鏡写真」のきらめきを与える。

`SpriteMaterial({ map: starTex, color: modelColor, blending: THREE.AdditiveBlending, depthWrite: false, transparent: true })`。マテリアルは**モデルごと**に 1 つ(計 5 マテリアル)でその星たちが共有。星ごとの変化はスケールと星ごとの `userData.phase` で与える。

**ハローレイヤー**: 各星スプライトの背後に、より柔らかいテクスチャ(純粋な放射状 `pow(1-t,3.5)`、スパイクなし)の 2 つ目のスプライトを、星のスケールの 2.6 倍・不透明度 `0.22 + 0.5*heat`・同色で配置。これがグロー — ポストプロセスなしでも見える。(codex が後で UnrealBloomPass で置換/補強するかもしれないが、ハロー単体で見栄えがしなければならない。)

### 4.2 サイズマッピング(volume → scale)
```
minV = min(volume of all topics), maxV = max(...)
t = (log(volume) - log(minV)) / (log(maxV) - log(minV))   // 0..1, log scale
scale = 7 + t * 17        // world units: 7 (small) … 24 (dominant topic)
```

### 4.3 ヒートマッピング(heat → 生命感)
`heat` ∈ [0,1] がレンダーループでパルスを駆動する:
```
pulse = 1 + heat * 0.12 * sin(elapsed * (0.8 + heat*1.6) + phase)
sprite.scale.setScalar(baseScale * hoverBoost * pulse)
halo.material  // per-star halo opacity flickers: haloBase * (0.9 + 0.1*sin(elapsed*2.3 + phase))
```
`phase = hash(topic.id) * 2π` とすることで星が同期してパルスすることは決してない。ホットなトピック(heat ≥ 0.8)は目に見えて呼吸し、コールドなものはほぼ静止。控えめさが肝 — スケール変動は最大 ±12%。

### 4.4 ホバー / 選択状態
- ホバー: `hoverBoost` を 1 → 1.45 へ 12%/フレームで lerp(`boost += (target-boost)*0.12`)。ハロー不透明度 ×1.5。ツールチップ表示。カーソル `pointer`。
- 選択: `hoverBoost` のターゲットは 1.6、加えて細いリングスプライト(円のストロークテクスチャ、モデル色、不透明度 0.8、スケールは星の 1.9 倍)がゆっくり回転(0.3 rad/s) — 「現在地」マーカー。選択される星は常に 1 つだけ。

---

## 5. 星座(クラスタ)

### 5.1 クラスタ配置 — 球面上の五角形

原点を中心とする半径 **R = 150** の球面上に 5 つのクラスタ中心を置き、赤道の上下に交互に配置して隣接する 2 つが同一平面に載らないようにする(平坦なリングに見えるのを回避):

```
for i in 0..4:
  lon = i * 72° + 20°            // constant offset so no cluster sits exactly on an axis
  lat = (i % 2 == 0) ? +18° : -22°
  center[i] = spherical(R, lat, lon)
```
リング上のモデル順: `fable-5, gpt-5-6, opus-4-8, gemini-3, glm-5-2` — Anthropic の 2 つ(暖色)の星座が隣接することはなく、隣同士の色コントラストを保つ。

### 5.2 クラスタ内の星の配置
決定論的で、トピック id をシードとする — 同じデータは常に同じ空を描画する。文字列ハッシュ(`fnv1a(topic.id)`)でシードした mulberry32 を使う。

```
layoutTopics(cluster):
  sort topics by volume desc
  for rank, topic in topics:
    // importance pulls big stars toward the cluster heart
    w   = 1 - rank / max(1, n-1)              // 1 = biggest … 0 = smallest
    r   = 8 + pow(1 - w, 0.75) * 34           // radius 8..42 from center
    dir = random unit vector (seeded), then flattened:
          dir.y *= 0.55                        // ellipsoid — constellations read better squashed
    pos = center + normalize(dir) * r
    // min-separation: retry up to 40 seeded jitters until
    // dist(pos, any placed star) >= (scaleA + scaleB) * 0.9 + 6
```
クラスタのバウンディング半径 ≈ 45。R = 150 かつ 72° 間隔なら、クラスタ間ギャップは常に ≥ 60 ユニット — クラスタが視覚的に融合することはない。

### 5.3 星座線 — 最小全域木
実際の星座はスパースな線の図形であって、毛玉ではない。各クラスタの星に対して 3D ユークリッド距離で **Prim 法の MST** を構築し、各 MST エッジを描画する:

- `THREE.LineSegments`、`LineBasicMaterial({ color: modelColor, transparent: true, opacity: 0.28, blending: THREE.AdditiveBlending, depthWrite: false })`。
- クラスタごとに 1 つの `LineSegments` オブジェクト(これで減光トグルがクラスタ単位で自明になる)。
- エッジの端点は各星のスケールの 55% だけ引き戻し、線が星のコアを貫かないようにする: `a' = a + normalize(b-a) * (scaleA*0.55)`、`b'` も対称に。
- クラスタフォーカス時(カメラがそこへフライしたとき): そのクラスタの線の不透明度を 0.28 → 0.5 にトゥイーン。他は → 0.12。

### 5.4 クラスタラベル
クラスタごとに `center + (0, clusterRadius + 14, 0)` にキャンバステクスチャのスプライト:
- テクスチャ: 512×128 キャンバス — モデル名 44px weight 700 白、その下に vendor 22px アルファ 55%、控えめなカラー下線バー(モデル色、4px、テキスト幅)。
- `SpriteMaterial({ depthWrite:false, transparent:true })`、スケール ≈ (48, 12)。
- カメラ距離による不透明度: ≤ 350 でフル、600 で 0.35 にフェード(空全体までズームアウトしたときにラベルが主張しすぎないように…… 実際は逆にする: 遠いときにフル、*別の*クラスタの近くでは 0.6。シンプルに保つ: `opacity = clamp(mapLinear(distToCamera, 150, 600, 0.55, 1.0))`)。
- 不可視の `PlaneGeometry` プロキシ経由、またはスプライトをレイキャストリストに含めることでレイキャスト可能に。クリック = クラスタフォーカス。

---

## 6. カメラとインタラクション

### 6.1 カメラ
- `PerspectiveCamera(55, aspect, 0.1, 3000)`、初期位置 `(0, 60, 330)`、lookAt は原点。
- `OrbitControls`:
  - `enableDamping: true, dampingFactor: 0.06`(重め、プラネタリウムの感触)
  - `rotateSpeed: 0.55`、`zoomSpeed: 0.9`
  - `minDistance: 120, maxDistance: 620`
  - `enablePan: false`(空の中心は 1 つ。パンはユーザーを迷子にする)
  - `autoRotate: true, autoRotateSpeed: 0.35`

### 6.2 アイドル時の自動回転
`pointerdown`/`wheel` があれば `autoRotate = false` にして `lastInteraction` を記録。ループ内で `now - lastInteraction > 7s` かつパネルが開いていなければ、なめらかに再開する(ガクつきを避けるため `autoRotateSpeed` を 0 → 0.35 へ約 2 秒かけて lerp)。

### 6.3 レイキャスティング
- `Raycaster` は `params.Sprite` デフォルトのまま。`interactive = [...starSprites, ...labelSprites]` を維持する。
- `pointermove` 時(rAF にスロットル): レイキャスト。最も近い星スプライト → ホバー状態、なければクリア。
- `click` 時(down/up 間のポインタ移動が 5px 未満の場合のみ — ドラッグ後には発火させない):
  - 星にヒット → `selectStar(topic)`
  - ラベルにヒット → `focusCluster(model)`
  - 何にもヒットせず → `deselect()`
- `Escape` キー → `deselect()`。空白部分のダブルクリック → `resetCamera()`。

### 6.4 カメラトゥイーン(外部ライブラリなし — 20 行のヘルパー)
```
tweenCamera(toPos, toTarget, ms=900):
  cubic ease-in-out on t∈[0,1]
  camera.position = lerpVectors(fromPos, toPos, e(t))
  controls.target  = lerpVectors(fromTarget, toTarget, e(t))
  controls disabled during tween, re-enabled after
```
- `selectStar`: target = 星の位置。カメラ位置 = 星の位置 + (カメラ→星の逆方向)を距離 = 90 になるようスケールし、y に +12 のバイアス。パネルはトゥイーン開始時に開く(両者が一緒にアニメーションする)。パネルが開いている間は自動回転オフ。
- `focusCluster`: target = クラスタ中心。距離 170。
- `resetCamera`: `(0, 60, 330)` / 原点に戻る、1100ms。

### 6.5 リサイズと DPR
`renderer.setPixelRatio(Math.min(devicePixelRatio, 2))`。`resize` ハンドラでカメラのアスペクトとレンダラーサイズを更新。レイアウトスラッシュなし — キャンバスは `position:fixed; inset:0`。

---

## 7. データ — JSON スキーマ(grok が埋める)

HTML 内に `const DATA = {...}` として埋め込む(単一ファイル制約。fetch なし)。grok はこの正確な形を opus に渡す:

```json
{
  "generatedAt": "2026-07-03T18:00:00Z",
  "source": "x.com",
  "models": [
    {
      "id": "fable-5",
      "name": "Fable 5",
      "vendor": "Anthropic",
      "color": "#E8865A",
      "topics": [
        {
          "id": "fable-5-agentic-coding",
          "label": "Agentic coding leap",
          "summary": "One or two sentences summarizing what X is saying about this topic.",
          "volume": 1250,
          "heat": 0.85,
          "posts": [
            {
              "author": "@handle",
              "text": "Post text, may be truncated to ~280 chars.",
              "url": "https://x.com/handle/status/123456789",
              "likes": 3400,
              "reposts": 900,
              "postedAt": "2026-07-03T12:34:00Z"
            }
          ]
        }
      ]
    }
  ]
}
```

**契約(grok が必ず守ること):**
- モデルはちょうど 5 つ、`id`/`name`/`vendor`/`color` はこの正確な値(§1 の表を参照)。
- モデルあたりトピック 4–8 個。トピックの `id` = モデル id + ケバブケースのスラグで、ファイル全体で一意。
- `label` は 40 文字以内(ツールチップとパネルタイトルになる)。`summary` は 1–2 文。
- `volume`: 1 以上の整数 — 相対的な言及数。全トピックで一貫した算出方法にする(サイズはグローバルに比較される)。
- `heat`: 0–1 の浮動小数 — *今まさに*どれだけ活発に議論されているか(速度 + 新しさ + エンゲージメント)。全体で少なくとも 1 トピックは ≥ 0.9、いくつかは ≤ 0.3 にして、パルスのコントラストが見えるようにする。
- `posts`: 代表的な実在ポスト 2–4 件、エンゲージメント降順でソート。`url` は実在する `https://x.com/...` のステータスリンクであること。`likes`/`reposts` は整数(0 も可)。適切にエスケープすること — これは `<script>` ブロック内に置かれる(テキスト中に生の `</script>` を入れない)。

---

## 8. three.js 実装構成(opus)

単一の `index.html`。three.js は ESM CDN の import map で(バージョンを固定):

```html
<script type="importmap">
{ "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.166.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.166.0/examples/jsm/"
} }
</script>
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
```

単一 HTML 内のファイルレイアウト: `<style>`(全 CSS)、`<div id="ui">`(タイトル、凡例、ヒント、ツールチップ、パネル)、`<canvas>` — その後に 1 つの `<script type="module">` を次のように構成する:

```
// ---------- data ----------
const DATA = {...};                        // grok's JSON verbatim
const MODEL_ORDER = ['fable-5','gpt-5-6','opus-4-8','gemini-3','glm-5-2'];

// ---------- utils ----------
fnv1a(str) -> uint32           // string hash for seeds
mulberry32(seed) -> () => f32  // seeded RNG
easeInOutCubic(t)
lerpV3(a, b, t, out)

// ---------- textures ----------
createStarTexture()            // §4.1 core + spikes
createHaloTexture()            // §4.1 soft radial
createLabelTexture(name, vendor, color)  // §5.4
createRingTexture()            // §4.4 selection ring

// ---------- layout ----------
layoutClusters(models) -> Map<modelId, {center: V3}>          // §5.1
layoutTopics(model, center) -> [{topic, pos: V3, scale}]      // §4.2 + §5.2
computeMST(points) -> [[i, j], ...]                            // §5.3 Prim's

// ---------- scene build ----------
initRenderer()                 // renderer, camera, controls, resize, fog
buildSky()                     // §3 dome + background starfield
buildConstellations(DATA)      // per model: group { stars[], halos[], lines, label }
                               //   sprite.userData = { topic, model, baseScale, phase, halo }

// ---------- interaction ----------
onPointerMove(e) / onClick(e)  // §6.3 raycast; drag-vs-click via pointer delta
selectStar(sprite) / deselect()
focusCluster(modelId)
tweenCamera(pos, target, ms)   // §6.4
renderPanel(topic, model)      // fills #panel DOM from data
renderLegend()                 // builds legend chips, wires click/dblclick

// ---------- loop ----------
function animate(tMs):
  requestAnimationFrame(animate)
  const t = tMs / 1000
  controls.update()
  updatePulses(t)              // §4.3 scale + halo flicker on every star
  updateHover()                // lerp hoverBoost toward targets
  updateLabelOpacity()         // §5.4 distance fade
  updateIdleAutorotate()       // §6.2
  spinSelectionRing(t)
  renderer.render(scene, camera)
```

**完了の定義(opus):**
1. `file://` または任意の静的サーバーからコンソールエラーなしで開き、ラップトップで 60fps。
2. 空全体が自動回転する。ドラッグ/ズームはダンピングの効いた感触。アイドルで回転が再開する。
3. データに従い、ラベル付きの 5 星座、MST の線、対数スケールでパルスする星。
4. ホバーツールチップ + カーソル。星クリック → カメラフライ + 実在ポスト入りパネル(リンクは新しいタブで X を開く)。ESC / 外側クリックで選択解除。凡例チップでクラスタフォーカス。
5. データは grok の JSON を `DATA` として貼り付けたもの — データ更新時にコード変更はゼロ。

---

## 9. 磨き込みターゲット(codex、opus の後)

優先順 — 収穫逓減が始まったら止める。60fps を維持:

1. **ブルーム**: `EffectComposer` + `UnrealBloomPass`(strength 0.55、radius 0.6、threshold 0.72)。コアが飛ばないようにハローの不透明度を下げて再調整(約 ×0.6)。hidpi で FPS が落ちる場合、コンポーザーは最大 DPR 1.5 でレンダリング。
2. **慣性の監査**: 星ホバーの lerp、パネルのイージング、カメラトゥイーン — 何もスナップしてはならない。ツールチップに 150ms の不透明度フェードを追加。
3. **線品質**: `LineBasicMaterial`(1px 制限)をより太い感触に置き換える — アドオンの `Line2`/`LineMaterial` で約 1.6px にするか、エッジごとのグラデーション不透明度(大きい星の近くほど明るく)。
4. **マイクロパララックス**: 背景星空をカメラの実効角速度の 0.35 倍で回転させる(あるいは単に一定のゆっくりした逆ドリフト、0.002 rad/s) — コストなしで奥行きを出す。
5. **星のまたたき**: 背景ポイントに ±6% のランダム不透明度シマーを、小さな ShaderMaterial のポイントごとの phase 属性で(安価な場合のみ)。
6. **パネルの気配り**: ヒートメーターはオープン時にフィルをアニメーション。ポストカードはスタガーイン(40ms/カード、translateY 8px → 0)。カスタムスクロールバー(6px、`rgba(255,255,255,0.15)`)。
7. **登場の演出**: ロード時、星がクラスタごとに合計約 1.6 秒かけてフェード/スケールイン(クラスタ間 100ms スタガー)、その後に線が描き込まれる(opacity 0 → 目標値)。第一印象でムードを決める。
8. **モバイル対応の最低限**: タッチでの回転/ズームが動くこと(OrbitControls デフォルト)。幅 640px 未満ではパネルをボトムシートにする(`bottom:0; left:0; right:0; height:55vh;` で X の代わりに `translateY`)。

---

## 10. チームフロー

```
fable (this doc) ──schema──▶ grok ──DATA json──▶ opus ──index.html──▶ codex ──▶ fable (visual review) ──▶ koichi
```

- **grok**: §7 の契約に沿って X のトピックを収集 → JSON を agmsg 経由で opus に送る(またはこのディレクトリに `data.json` を書いて通知)。
- **opus**: §1–§8 をこのディレクトリの `index.html` として実装。grok のデータが届いていなければ、正確に同じスキーマのプレースホルダーデータで構築し、到着時に差し替える。完了したら → codex に通知。
- **codex**: §9 を `index.html` に適用(その場で編集) → fable に通知。
- **fable**: ビジュアルレビュー、必要なら変更依頼、その後 koichi に納品。

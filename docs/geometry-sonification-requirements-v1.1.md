# TRANSFIGURATION — Geometry Sonification 要件定義書

**Version:** 1.1  
**Date:** 2026-08-08  
**Target:** `TRANSFIGURATION — 多面体変相` WebGL作品  
**Current source:** `index.html`（v1.0時点の `polyhedra_transfiguration (4).html` を Phase 0 でリネーム）  
**Purpose:** 多面体・数式によるリアルタイム音響生成（Sonification）の追加  
**Status:** Phase 0 完了 / Phase 1 実装前

---

## v1.1 変更履歴

v1.0 は既存HTMLを精読する前に書かれた要件定義であり、実装確認の結果、いくつかの前提が成立しないことが判明した。v1.1 はその修正版である。

**v1.1 が正史であり、以下の表で「廃止」とした v1.0 の記述は無効とする。**

| # | 変更点 | 該当章 | 理由 |
|---|---|---|---|
| 1 | `update()` に音響処理を入れず、25Hz の独立した読み取り専用 audio metrics パスを採用 | §5.2 / §16.1 / §22 / §29 | 描画ホットパスへの副作用をゼロにするため。追加コストは約 10k 演算/秒で無視できる |
| 2 | Audio Graph を `Voices → Mix → Compressor → MasterGain → Destination` に変更 | §11 / §17 | fade用Gainが Compressor の前段だと、fade in 初期に音量の膨らみが出る |
| 3 | 周波数の hard clamp を廃止。範囲外 voice は gain fade で処理 | §4.2 / §24 | clamp すると voice が境界に張り付き、幾何比率が失われ、複数 voice が同一周波数へ潰れる |
| 4 | 形態ごとの追加音域正規化を行わない | §4.2 | 正多面体は辺長が均一なため、形態別に正規化すると ico / dod / ti の区別が消え AC-05 を満たせない |
| 5 | ランタイム辺長クラスタリングを廃止し、幾何クラス A / B / C / D × 2モード方式を採用 | §4.1 / §22 / §23 | 位相が完全に静的なため分類はコンパイル時に確定できる。かつ正多面体では辺長が均一で、クラスタリングは1〜2クラスタに退化し8モードを構成できない |
| 6 | 音響クラスを `kind` index へ固定しない | §23 / §24 | 区間1→2 の base 切替（ico→dod）で kind=0 / kind=1 の役割が反転し、voice が瞬間ジャンプするため |
| 7 | L→0 時の Infinity / NaN 防御を必須化 | §18 / §24 / AC-15 | τ=0 でクラスB、τ=0.5 でクラスA が厳密に 0 になる。非有限値は AudioParam 例外、またはグラフ全体の恒久的無音化を招く |
| 8 | SOUND UI は通常の `<button>` を使用 | §13.2 | アクセシビリティと標準的な操作性を優先 |
| 9 | interactive element にフォーカスがある場合、SPACE HOLD を発火しない | §13.2 / §14 / AC-16 | button クリック後にフォーカスが残り、SPACE で HOLD と SOUND が同時にトグルされるため |
| 10 | モバイルでも SOUND 操作を可能にする | §13 / §13.2 / AC-17 | 既存 `.br` は `max-width:680px` で `display:none` のため、そのまま追加すると 680px 以下でトグルが消える |
| 11 | baseline FPS 計測を Phase 0 必須項目から外す | §28 | AC-14 の描画差分検証は visual baseline スクリーンショットで足りる。FPS は Phase 4 の性能確認で必要に応じて実施する |

---

## 0. 要約

既存の `TRANSFIGURATION — 多面体変相` に、既製BGM・録音素材・外部音源ファイルを追加するのではなく、**現在表示されている多面体の幾何形状と変相パラメータからリアルタイムに音を生成する機能**を追加する。

音は作品に付随するBGMではなく、視覚と同じ数学的状態から生成される。

```text
数式 / 幾何
   ↓
τ / σ / s / chaos / 現在の頂点・辺・面
   ├── WebGL       → 形態・色・動き
   └── Web Audio   → 共鳴・音程・倍音・定位
```

作品の中心コンセプトを以下とする。

> **形が鳴る。**
>
> 多面体を仮想的な共鳴体として扱い、形状が変化すると共鳴構造も変化する。

物理的に厳密な音響シミュレーションを目標とはしない。  
幾何学的特徴を一貫した規則で音へ写像する **sonification / modal-synthesis-like representation** とする。

---

# 1. 優先度

## P0 — 必須

1. 既存WebGL作品のビジュアル、数式、変形シーケンス、操作感を変更しない。
2. 外部BGM、録音音声、SEファイルを使用しない。
3. 音はWeb Audio APIでブラウザ内リアルタイム生成する。
4. 初期状態は無音とし、ユーザー操作なしに自動再生しない。
5. 音声機能が利用できない環境でも、既存WebGL作品は正常動作する。
6. Sound OFF時にはAudio処理負荷を可能な限り抑える。
7. 音量ピーク、クリックノイズ、急激な周波数ジャンプを防止する。
8. Sound機能の追加によって既存コードを大規模リファクタリングしない。

## P1 — 本実装の中心

1. 現在の幾何形状から共鳴周波数群を算出する。
2. 形態変化に伴い音も連続的に変化する。
3. 安定した名前付き多面体状態では音響も安定・収束する。
4. 変相途中では `chaos` に応じて微細な不協和・揺らぎが増える。
5. `σ` による星形化では高域・倍音成分が増える。
6. `s` によるアフィン変形ではスペクトルが伸縮する。
7. 各状態で音量を正規化し、形による不要な音量差を抑える。

## P2 — 発展

1. DRAG — ORBIT に合わせて音の左右定位を変化させる。
2. 面法線と視点方向から「正面を向く面ほど強く聞こえる」表現を行う。
3. 多面体を回転させる行為そのものを簡易的な演奏行為にする。
4. 将来的なヘッドフォン向け空間音響への拡張余地を残す。

---

# 2. 既存実装との関係

現行作品では以下のパラメータで形態が制御されている。

```text
τ (tau)   : truncation / 切頂
σ (sigma) : stellation / 星形化
s         : affine scale / 異方的変形
chaos     : 変相途中の一時的な崩し量
```

既存の主要式：

```text
p(v,w) = M · ( v + τ(w−v) )

apex = C + σ · R_f · N

M = diag(s^−1/2, s, s^−1/2)
det M = 1
```

既存コード上の接続点は以下（v1.1 でコード実測により確定）。

| 識別子 | スコープ | 音響側での用途 |
|---|---|---|
| `curM` | module | 現在アクティブなモデル。`curM.K`（静的な位相）と `curM.base`（静的な基底頂点）を読む |
| `curTau` / `curSig` / `curSt` | module | 現在の τ / σ / s |
| `curChaos` | **v1.1 で新規追加** | 現在の chaos。v1.0 時点では `applySequence()` のローカル変数で、外部から参照できない |
| `MODEL.ico` / `MODEL.dod` | module | モデル実体。通常は `curM` 経由で参照する |
| `frame()` | module | 音響更新の呼び出し位置（`applySequence(dt)` の直後） |
| `buildModel(base)` / `M.update(...)` | — | **読むだけ。v1.1 では改変しない**（§5.2） |

### 既存JavaScriptへの変更は次の3箇所のみとする

| # | 場所 | 変更内容 | 根拠 |
|---|---|---|---|
| 1 | `applySequence()` | `curChaos = chaos;` を追加（`curTau` / `curSig` / `curSt` と同じ位置・同じ形式） | §5.2 |
| 2 | `frame()` | `applySequence(dt)` の直後に音響更新の呼び出しを1行追加 | §12 |
| 3 | SPACE の `keydown` ハンドラ | interactive element にフォーカスがある場合は HOLD を発火しないガードを追加 | §13.2 / AC-16 |

これ以外の既存関数の書き換えを行ってはならない。特に `buildModel()` と `M.update()` は読むだけで、一切変更しない。

3 は既存の挙動を変えるように見えるが、変更前の挙動（`document.body` にフォーカスがある通常状態）では完全に同一であり、v1.0 時点では存在しなかった interactive element がフォーカスを持つ場合のみ分岐する。

---

# 3. コンセプト要件

## 3.1 音楽を再生しない

本機能はBGM生成機能ではない。

以下を避ける。

- メロディをあらかじめ定義する
- コード進行を設定する
- アルペジオを鳴らす
- BPMを設定して音楽化する
- 音声サンプルを再生する
- 「多面体Aにはこの曲」のような状態別プリセットを割り当てる

音は、現在の幾何形状から導出される。

---

## 3.2 多面体を仮想共鳴体として扱う

概念的な対応関係は以下とする。

| 幾何／パラメータ | 音響上の役割 |
|---|---|
| 辺長 | 共鳴周波数 |
| 辺長比 | 共鳴モード間の周波数比 |
| 面積 | 各モードの重み |
| 面／頂点の空間位置 | 定位・重み付け |
| 面法線 | 視点に対する音響強度 |
| τ | 主として幾何形状を介して周波数構造へ反映 |
| σ | 高域・倍音・brightness |
| s | スペクトルの伸縮／register |
| chaos | micro-detune / beating / inharmonicity |
| 安定形 | 共鳴の収束 |
| 変相中 | 共鳴構造の移動・揺らぎ |

---

# 4. 音響生成方式

## 4.1 基本方式

**8 voice の固定数**の共鳴モードを使用する。

v1.1 では、この 8 voice の内訳を次のように固定する。

```text
8 voices = 幾何クラス 4 種（§23 の A / B / C / D） × 2 モード
```

voice スロットと幾何クラスの対応は静的であり、実行中に組み替えない（§24）。

```text
8 resonant voices
+
Voice Mix
+
Compressor / Limiter相当
+
Master Gain
```

※ 接続順は §11 を参照（v1.1 で Compressor と Master Gain の順序を入れ替えた）。

AudioNodeをフレームごとに生成・破棄しない。

初期化時に必要数を生成し、その後は、

- frequency
- gain
- detune
- filter
- pan

等のパラメータのみ更新する。

---

## 4.2 周波数算出

基本原理：

```text
長い辺 → 低い共鳴
短い辺 → 高い共鳴
```

概念式：

```text
f ∝ 1 / L
```

ただし画面内の全辺を個別Oscillatorへ直接割り当てない。

### 必須処理（v1.1 改訂）

1. 現在の幾何形状から、§23 が定義する幾何クラス A / B / C / D の代表長を取得する。
2. 各クラス長の逆数比を、そのまま周波数比へ変換する。
3. 各クラスに 2 モードを割り当て、計 8 voice とする。
4. 長さが 0 へ退化したクラスは、**周波数を clamp せず gain を 0 へフェード**する。
5. 非有限値（Infinity / NaN）を AudioParam へ渡さない（§18 / AC-15）。

### 廃止した処理（v1.0 からの削除）

以下は v1.1 で**行わない**。v1.0 の該当記述は無効とする。

| 廃止項目 | 置換先 |
|---|---|
| 類似長をランタイムでクラスタリングする | §23 の静的な幾何クラス分類 |
| 上位最大8モードへまとめる | 4クラス × 2モード = 8 の固定割当（§4.1） |
| 聴覚上の極端な低域・高域を Clamp する | gain fade（上記4） |
| 全形態で基準音域が移動しないよう正規化する | **行わない**（下記） |

**形態ごとの追加音域正規化を行ってはならない。** 正多面体・準正多面体は定義上すべての辺長が等しいため、形態別に音域を正規化すると Icosahedron / Dodecahedron / Truncated Icosahedron が同一音域へ潰れ、AC-05 を満たせなくなる。

### 周波数式

```text
f = f0 · ( L_ref / L )

f0    = 110 Hz    固定
L_ref = 1.2361    固定定数
        （中半径を1に正規化した正二十面体の辺長 a = 2/φ）
```

`L_ref` は形態によらず固定する。これにより幾何学的な長さ比がそのまま音程比になる。

「音階へ量子化」しない。幾何学的な比率を維持する。

### 実測される音域

上式で §23 の各クラスを鳴らした場合、基音は約 **110〜400 Hz**、affine 変形（s）による分裂とモード倍音を含めても約 **1.7 kHz 以下**に収まる（実測値は §23 の表）。

したがって v1.0 が想定した 55〜1760 Hz の範囲に自然に収まり、clamp は不要である。

なお 55 Hz はノートPC内蔵スピーカーの多くが再生できないため、基音の実用下限は 110 Hz 前後とする。

---

# 5. 幾何データの取得

## 5.1 原則

音響用の値は、可能な限り **現在描画されている形態そのもの** から取得する。

`τ` を単純に「音程」へ直接割り当てるのではなく、

```text
τ
↓
頂点位置・辺長が変化
↓
辺長から共鳴周波数が変化
```

という流れを優先する。

これにより、視覚と音が同じ幾何学的原因を共有する。

---

## 5.2 audioMetrics（v1.1 改訂）

### 方式：独立した読み取り専用パス

**`M.update()` の内部には音響処理を一切追加しない。**

音響メトリクスは、`update()` とは完全に独立した読み取り専用関数として実装し、約 **25 Hz** で実行する。

```text
frame()
 ├─ applySequence(dt) → M.update(...)     ← 描画。音響コードを入れない
 └─ SOUND.tick()                          ← 25Hz スロットリング。読み取りのみ
       ↓
     curM.base.V / curM.K（静的）と curTau / curSt から P[60] を再計算
       ↓
     §23 の幾何クラス A / B / C / D を算出
```

#### v1.0 からの変更理由

v1.0 §5.2 は `M.update()` の計算途中で `A` / `B` / `AP` / `C` / `N` / `Rf` を流用する方式だったが、v1.1 ではこれを採用しない。

- `update()` は毎フレーム実行される最ホットパスであり、ここへ分岐や書き込みを入れると AC-14 / §16.2（描画とFPSに差を出さない）の検証コストが跳ね上がる
- 独立パスの実測コストは、25 Hz で「60頂点の再計算 + 90本の辺長 + 32面の重心・法線」＝**約 10k 演算/秒**であり、無視できる
- §30 のロールバック条件を完全に満たす。音響ブロックを削除するだけで元の作品に戻る

したがって v1.0 §5.2 の「`M.update()` の計算中に利用して更新する」という記述は**無効**とする。

### 取得するもの

```js
audioMetrics = {
  classes: [ /* §23 の A / B / C / D。各 { len, spread, weight } */ ],
  centers: [],   // P2 定位用（§15）
  normals: [],   // P2 定位用（§15）
  radius: 0      // 参考値
}
```

### 座標系（重要）

音響メトリクスは、**`inset` / `lift` / 面回転を適用する前の `P` 座標**から算出する。

`update()` 内の `Q` は `inset = 0.905 − 0.115·chaos` によって一様スケールされているため、`Q` を使うと chaos が最大 1.145 倍（約 2.3 半音）の全体ピッチ変動として音程へ漏れる。chaos は §6.4 が定義する detune / beating / inharmonicity としてのみ作用させる。

同じ理由で、`M.radius()`（= `maxR`）は `AP` と `lift` を含み chaos 依存であるため、**正規化基準には使わない**。

### 時間軸

音響は `THREE.Clock` の `elapsedTime`（コード上の `t`）を参照しない。`getDelta()` は clamp されるが `elapsedTime` は clamp されず、タブ復帰時に大きく飛ぶ既存挙動があるため。

- スケジューリング：`ctx.currentTime`
- 状態：`curTau` / `curSig` / `curSt` / `curChaos`

### 制約

- 描画結果を変更しない。
- 毎フレーム大量のObject生成を行わない。
- TypedArrayまたは再利用Arrayを使う（`P[60]` は再利用 `Float32Array`）。
- 静的テーブル（辺クラス分類・面インデックス）はモデルごとに初回1回だけ構築し、モデルへ読み取り専用プロパティとしてキャッシュしてよい。

---

# 6. パラメータ別サウンド設計

## 6.1 τ — truncation

`τ` 自体を直接「音程」にすることは避ける。

原則：

```text
τ
→ 現在の辺長構造
→ resonant frequency ratios
```

その結果として、

- 正二十面体
- 切頂二十面体
- 二十・十二面体
- 切頂十二面体
- 正十二面体

の形態差が音響構造へ反映される。

---

## 6.2 σ — stellation

`σ` が増加し、面のapexが突出するほど、

- 高域成分を増やす
- 倍音比率を増やす
- filter cutoffを上げる
- 非整数倍音をわずかに追加する

方向とする。

目標：

```text
凸多面体
→ 丸く透明な共鳴

星形化
→ より硬質で鋭いスペクトル
```

ただし耳障りな金属音にはしない。

---

## 6.3 s — affine deformation

既存式：

```text
M = diag(s^−1/2, s, s^−1/2)
det M = 1
```

に対応して、音響スペクトルにも異方的な伸縮感を与える。

### 目標

```text
s > 1
→ 高低差／スペクトル幅が広がる

s < 1
→ スペクトル幅が圧縮される
```

全体Gainは正規化し、`s` の変化だけで急激に音量が変わらないようにする。

`det M = 1` の視覚的コンセプトと対応して、

**形が伸びても全体の音響エネルギー感は大きく変えない**

という設計にする。

物理的なエネルギー保存の再現ではなく、作品上の対応関係である。

---

## 6.4 chaos — transfiguration

既存：

```js
chaos = Math.sin(Math.PI * u)
```

により変相中央で最大になる。

音響側では、

```text
chaos = 0
→ 安定・純度が高い

chaos ↑
→ micro-detune
→ beating
→ slight inharmonicity

chaos ↓
→ 新しい共鳴構造へ収束
```

とする。

### chaos は幾何長へ作用させない（v1.1 追記）

既存コードでは `chaos` が `inset` / `lift` / 面回転を駆動しているが、これらのうち音響へ影響しうるのは `inset` による**全辺の一様スケール**だけである（面回転は長さを変えない）。

この一様スケールをそのまま音程へ通すと、chaos が最大 1.145 倍（約 2.3 半音）の全体ピッチ変動になり、「構造が移動している」という表現と競合する。

したがって §5.2 のとおり `inset` 適用前の `P` 座標を使い、chaos は上記の detune / beating / inharmonicity としてのみ作用させる。

### 禁止

- 大きなランダム音
- 激しいノイズ
- グリッチSE
- sudden pitch jump
- 破裂音
- クリックノイズ

変相は「崩壊」ではなく、**連続的な相転移**として聞こえること。

---

# 7. 安定形と変相

既存シーケンスの各名前付き状態では音響も落ち着く。

```text
Stable Form
↓
Transfiguration
↓
Stable Form
```

音響：

```text
Clear resonance
↓
Detune / beating / spectral migration
↓
New clear resonance
```

安定形へ到達した瞬間に新しい音を鳴らし直すのではなく、前状態から連続的に遷移する。

---

# 8. Small Stellated Dodecahedron

`Small Stellated Dodecahedron / 小星形十二面体` は凸多面体群と異なる音響的性格を持たせてよい。

ただし、

```text
この形だから専用プリセットを再生する
```

という実装にはしない。

`σ`、星形化した幾何、辺長比、面構造から結果的に、

- 高域成分が多い
- 倍音が複雑
- わずかにinharmonic

な音になることを狙う。

---

# 9. 音色

## 9.1 方向性

目標：

```text
glass
+
metal
+
membrane
+
crystal resonance
```

の中間。

既存WebGLの

- thin film
- pearl
- Fresnel
- transparent surface

と視覚・聴覚のトーンを合わせる。

---

## 9.2 避ける音

- 8bit
- chip tune
- EDM synth
- obvious techno sequence
- piano
- strings
- drums
- cinematic pad
- preset-like ambient music

「シンセサイザーを鳴らしている」と強く感じさせない。

---

# 10. Web Audio 構成

v1.1でも外部Audio libraryを導入しない。

使用候補：

```text
AudioContext
OscillatorNode
GainNode
BiquadFilterNode
StereoPannerNode
DynamicsCompressorNode
PeriodicWave（必要な場合のみ）
```

必要に応じてブラウザ内生成NoiseBufferを使うことは可能だが、録音素材は使用しない。

---

# 11. Audio Graph

v1.1 の接続順：

```text
Voice 1 ─┐
Voice 2 ─┤
Voice 3 ─┤
Voice 4 ─┤
Voice 5 ─┤
Voice 6 ─┤→ Voice Mix → Compressor → Master Gain → Destination
Voice 7 ─┤                            (limiter設定)   (fade)
Voice 8 ─┘
```

### v1.0 からの変更理由

v1.0 は `Master Gain → Compressor` の順だったが、これを**入れ替える**。

fade 用の Master Gain が Compressor の**前段**にあると、fade in 初期の小音量に Compressor が反応せず、音量が立ち上がった瞬間にゲインリダクションが遅れて一時的な「膨らみ」が出る。これは §17 の「sudden gain jump防止」と衝突する。

Compressor をリミッターとして先に置き、その後段で Master Gain によりフェードすることで、ON/OFF が常にクリーンになる。

v1.0 §11 の接続順、および §17 の「Master段にDynamicsCompressorNode等を配置」という記述は**無効**とする。

P2では各VoiceまたはVoice Groupの前段へStereoPannerを追加できる。

---

# 12. パラメータ更新

WebGLは毎フレーム描画を継続する。

Audio制御値の重い計算は必ずしも60fpsで行わない。

v1.1 で確定：

```text
Geometry render : requestAnimationFrame（既存のまま）
Audio metrics   : 25 Hz（独立した読み取り専用パス。§5.2）
Audio smoothing : Web Audio automation
```

音響メトリクスの更新は `frame()` から呼ぶが、内部で経過時間を見て 25 Hz にスロットリングする。Sound OFF 時は関数先頭で即 return し、メトリクス計算を一切行わない（P0-6）。

Audio parameter変更には可能な限り、

```text
setTargetAtTime()
linearRampToValueAtTime()
exponentialRampToValueAtTime()
```

等を使用し、値を瞬間切替しない。

---

# 13. UI

既存右下UI：

```text
DRAG — ORBIT
SCROLL — DOLLY
SPACE — HOLD
```

へ以下を追加する。

```text
SOUND — OFF
```

Sound ON時：

```text
SOUND — ON
```

Audio初期化に失敗した場合（§18）：

```text
SOUND — N/A
```

### 表示要件（v1.1 追記）

**SOUND トグルは全画面幅で操作可能でなければならない。**

既存 `.br` は `@media (max-width:680px)` で `display:none` になっているため、`.br` 内へそのまま行を追加すると 680px 以下で SOUND トグルが消える。モバイルは AudioContext のユーザージェスチャ要件が最も厳しい環境であり、ここで操作できないことは許容しない（AC-17）。

対応方針：680px 以下では `.br` のうち **SOUND 行のみを表示**する。既存の 3 行（DRAG / SCROLL / SPACE）は従来どおり非表示のままとし、レイアウトの見え方を変えない。

---

## 13.1 初期状態

```text
SOUND — OFF
```

必須。

ページロード時にAudioContextを自動開始しない。

ユーザーがSoundをONにした操作をトリガーとしてAudioContextを生成／resumeする。

---

## 13.2 UI実装上の注意（v1.1 改訂）

### 要素種別：通常の `<button>` を使用する

Sound toggle は通常の `<button>` 要素として実装する。アクセシビリティと標準的な操作性（キーボード操作・スクリーンリーダー）を優先する。

既存 `.ui` は `pointer-events:none` のため、この button 部分のみ `pointer-events:auto` とする。

見た目は既存タイポグラフィに合わせ、`background` / `border` / `padding` をリセットして周囲のモノスペース行と同一の外観にする。派手なボタン、アイコン、スライダーは追加しない。

v1.1 でも音量スライダーは不要。

### SPACE キーとの競合を防ぐ（必須）

既存の HOLD は `window` レベルの `keydown` で `e.code === 'Space'` を捕捉している。

button はフォーカスを保持するため、**対策なしでは SOUND をクリックした後に SPACE を押すと、HOLD と SOUND が同時にトグルされる。**

対策として、**HOLD 側のハンドラに「interactive element にフォーカスがある場合は発火しない」ガードを入れる**（AC-16）。

```text
SPACE 押下
  ↓
フォーカスが button / a / input / select / textarea
または contenteditable にあるか？
  ├─ Yes → HOLD を発火しない（button 本来の動作に委ねる）
  └─ No  → 従来どおり paused をトグル
```

button 側をフォーカス不可にする方式（`tabindex` 除去など）は採用しない。アクセシビリティを損なうため。

### その他

- タップ時のテキスト選択を防ぐため `user-select:none` を指定する。
- 状態は `aria-pressed` で表現する。
- 既存レイアウト・タイポグラフィを崩さない。

---

# 14. HOLD動作

既存：

```text
SPACE — HOLD
```

でシーケンス停止時、

- 形態を停止
- 音響ターゲット値の変化も停止
- 現在の共鳴は継続

とする。

Soundそのものをmuteしない。

つまり、

```text
HOLD = freeze geometry state
```

であり、

```text
HOLD ≠ stop audio
```

とする。

### 実装上の含意（v1.1 追記）

`paused` は `phase` の進行のみを止める。したがって `curTau` / `curSig` / `curSt` / `curChaos` が自動的に凍結し、§5.2 の独立パスはそれらを読むだけなので、**音響ターゲット値の凍結に特別な処理は不要**である。

ただし既存コードでは `root.rotation` が `elapsedTime` 駆動のため、HOLD 中もオブジェクトは回転し続ける。P2 の定位（§15）は HOLD 中も追従して動くことになるが、画面上のオブジェクトが実際に回転している以上、視聴覚一致としてはそちらが正しい。Phase 5 で最終確認する。

SPACE キー押下時のフォーカスガードについては §13.2 を参照。

---

# 15. Orbit連動 — P2

DRAG操作によるカメラ回転を音響へ反映する。

概念：

```text
左に見える構造 → 左寄り
右に見える構造 → 右寄り

正面向き面 → 強い
側面 → 弱い
背面 → さらに弱い
```

使用候補：

```text
face center
face normal
camera position
camera right vector
```

ただしv1.0完成を妨げる場合はP2として後回しにする。

---

# 16. 性能要件

## 16.1 Audio

- voice数：8固定（4クラス × 2モード）
- AudioNodeを毎フレーム生成しない
- Sound OFF時は不要処理を停止（`tick()` 先頭で即 return、fade完了後に `ctx.suspend()`）
- 毎フレームのArray/Object allocationを最小化
- 幾何メトリクスは **`update()` とは独立した読み取り専用パスで 25 Hz** で算出する（§5.2）
- 静的テーブル（辺クラス分類・面インデックス）はモデルごとに初回1回のみ構築
- パラメータ更新をsmooth化

v1.0 の「幾何メトリクス抽出は既存計算を再利用」は**無効**とする（§5.2 の変更理由を参照）。独立パスの実測コストは約 10k 演算/秒であり、再利用による節約より描画側の無副作用を優先する。

## 16.2 Visual

Sound追加前後で、

- WebGL FPS
- Geometry
- Shader
- sequence timing
- camera
- orbit
- dolly
- caption
- piece
- ring text

に目視可能な差を発生させない。

---

# 17. 音量・安全性

Master Gainは低めから開始する。

必須：

- clipping防止
- sudden gain jump防止
- sudden frequency jump防止
- Sound ON/OFFに短いfadeを入れる
- **Voice Mix の直後にリミッター設定の DynamicsCompressorNode を置き、その後段の Master Gain でフェードする**（§11）
- 初回ON時に大音量を出さない
- AudioParam へ非有限値を渡さない（§18 / AC-15）

v1.0 の「Master段にDynamicsCompressorNode等を配置」は、Compressor が Master Gain の後段であることを含意していたため**無効**とする。正しい接続順は §11 を参照。

目標：

```text
Sound ON
→ 100〜300ms程度でfade in

Sound OFF
→ 100〜300ms程度でfade out
```

---

# 18. フォールバック

以下の場合でもWebGL作品を継続する。

- `AudioContext` が利用できない
- Audio initialization failure
- AudioContext resume failure（Promise の reject を必ず catch する）
- StereoPanner非対応
- その他Audio node生成エラー

エラーによって`frame()`を停止しない。

Sound機能はWebGL描画から疎結合にする。

初期化に失敗した場合は内部フラグを立て、UI を `SOUND — N/A` に戻して以後の音響処理を行わない（§13）。

## 18.1 非有限値の防御（v1.1 新設・必須）

`f = f0 · (L_ref / L)` は `L → 0` で Infinity になる。そして **`L` が厳密に 0 になる状態が実際に存在する。**

```text
τ = 0    → クラスB（頂点図形の辺）が 0
τ = 0.5  → クラスA（原辺上の切り残し）が 0
```

これは異常系ではなく、シーケンスが必ず通過する正常な状態である（Icosahedron / Dodecahedron / Icosidodecahedron）。

**AudioParam へ Infinity / NaN を渡した場合、例外が投げられるか、ノードが NaN を出力し続けてグラフ全体が恒久的に無音化する。** 一度この状態になるとコンテキストを作り直すまで復帰しない。

したがって以下を必須とする。

1. AudioParam へ値を設定する直前に `Number.isFinite()` で検査する。
2. 非有限または下限未満の場合、**周波数は直前の有効値を保持**し、更新しない。
3. 当該クラスの gain を 0 へフェードする（§4.2）。
4. gain の復帰も、長さが下限を超えた時点からフェードで行う。

長さ 0 への退化は、切頂そのものの聴覚化として意図された挙動である。τ が 0 から増えるにつれてクラスB が無音からフェードインし、τ が 0.5 へ向かうにつれてクラスA がフェードアウトする。

---

# 19. 既存Three.jsへの影響

このSound実装ではThree.jsのバージョンを変更しない。

現行の

```text
three.js r128
```

を維持する。

Sound機能追加と、

- Three.jsアップグレード
- ES Module化
- Vite導入
- TypeScript化
- 大規模ファイル分割

を同一変更に含めない。

---

# 20. 実装構造案

単一HTMLを維持する場合の概念：

```text
1. Geometry
2. Render Model
3. Shader
4. Setup
5. Forms / Equations
6. Caption
7. Ring
8. Piece
9. Sequence
10. Composite
11. Camera

12. Sonification
    - Audio state
    - Voice creation
    - Static class tables      （§23。モデルごとに初回1回）
    - Geometry metrics         （25Hz 読み取り専用パス。§5.2）
    - Class → voice mapping    （長さ降順の静的割当。§23.5）
    - Parameter mapping
    - Audio update
    - Sound toggle

13. Frame loop
```

既存番号を無理に変更する必要はない。

重要なのは、音響コードを独立セクションとして識別できること。

---

# 21. Sonification State案

```js
const SOUND = {
  enabled:    false,   // ユーザーが ON にしているか
  failed:     false,   // 初期化に失敗した（§18）。true 以降は何もしない
  ctx:        null,
  mix:        null,    // Voice Mix
  comp:       null,    // Compressor（limiter設定）
  master:     null,    // Master Gain（fade用。Compressor の後段）
  voices:     [],      // 8。各 { osc, gain, filter, pan, lastFreq }
  classes:    [],      // 4。§23 の A / B / C / D。各 { len, spread, weight }
  lastUpdate: 0        // 25Hz スロットリング用
};
```

必要以上にグローバル状態を増やさない。

`voices[i]` と幾何クラスの対応は静的とする（§24）。

```text
voice 0,1 → クラス0    voice 2,3 → クラス1
voice 4,5 → クラス2    voice 6,7 → クラス3
```

---

# 22. Geometry → Audio変換フロー

v1.1 のフロー。`M.update()` は音響へ関与せず、音響は状態変数のみを読む。

```text
applySequence(dt)
   ├─→ M.update(tau, sigma, st, chaos)   ← 描画。音響コードなし
   └─→ curTau / curSig / curSt / curChaos / curM を更新
                    │
                    │  frame() が 25Hz でスロットリングして呼ぶ
                    ↓
          SOUND.tick()（読み取り専用）
                    ↓
   curM.base.V + curM.K（静的） + curTau + curSt
                    ↓
             P[60] を再計算（inset 適用前）
                    ↓
   §23 の静的分類テーブルで幾何クラス A / B / C / D を集計
     ├─ len    代表長
     ├─ spread クラス内の min/max 幅（s ≠ 1 で拡がる）
     └─ weight 面積または本数による重み
                    ↓
        長さ降順でクラス0〜3へ整列（kind index を使わない）
                    ↓
      f = f0 · (L_ref / L)  ×  2モード = 8 voice
                    ↓
          σ / s / chaos によるモジュレーション
                    ↓
       非有限値チェック + 退化クラスの gain fade（§18.1）
                    ↓
      parameter smoothing（setTargetAtTime）
                    ↓
                Web Audio nodes
                    ↓
                  sound
```

---

# 23. 幾何クラス分類（v1.1 全面改訂）

**v1.0 の「ランタイム辺長クラスタリング」は廃止する。** 本章がその置換であり、v1.0 §23 の記述は無効とする。

## 23.1 v1.0 方式が成立しない理由

### 理由1 — 正多面体では辺長が均一

正則・準正則多面体は**定義上すべての辺長が等しい**。したがって辺長クラスタリングは、シーケンス中の名前付き安定形（＝音が最も豊かであるべき状態）で常に1〜2クラスタへ退化し、8モードを構成できない。

### 理由2 — 位相が完全に静的

`truncation()` が生成する `pairs` と `faces` は τ / σ / s に依存しない。動くのは頂点座標だけである。したがって分類はコンパイル時に確定でき、ランタイムでクラスタリングする必要がない。静的分類は定義上 deterministic であり、voice 順序の入替が原理的に起こらない。

## 23.2 静的な辺分類

`truncation()` の構造から、辺はちょうど2種類に分かれる。

```text
pairs[i]  = 有向辺 (v,w) に対応する切頂頂点   … 60本（= 2E。ico系・dod系とも）

クラスA「原辺上の切り残し」
    idx(a,b) ─ idx(b,a)  を結ぶ辺            … 30本（= E）
    長さ L_A = |1 − 2τ| · a₀
    τ = 0.5 で 0 に退化

クラスB「頂点図形の辺」
    idx(v,w) ─ idx(v,w′)  を結ぶ辺
    （w, w′ は nbrs[v] 上で隣接）              … 60本
    長さ L_B = 2τ · a₀ · sin(θ/2)
    θ = 頂点における稜線間角度
        ico 基底: θ = 60°    dod 基底: θ = 108°
    τ = 0 で 0 に退化

合計 90本 = 切頂多面体の辺数（V60 − E90 + F32 = 2）に一致
```

`a₀` は基底多面体の辺長。中半径を1に正規化しているため `a₀(ico) = 2/φ = 1.23607`、`a₀(dod) = 0.76393`。

## 23.3 静的な面分類

```text
クラスC : kind = 0 の面（元の面由来の 2m 角形）
クラスD : kind = 1 の面（頂点図形）
```

各クラスの代表長は、**面中心から apex を経由した頂点までの距離**とする。

```text
L_C = Rf₀ · √(1 + σ²)
L_D = Rf₁ · √(1 + σ²)

Rf = 面中心から面頂点までの最大距離（inset 適用前）
```

σ = 0 では `√(1+σ²) = 1` となり単なる面の外接半径に一致するため、σ に関して連続である。σ > 0 では角錐の側稜そのものになる。これは実際に描画されている辺であり、σ の効果が幾何量として音へ入る。

## 23.4 s ≠ 1 の扱い

`M = diag(s^{−1/2}, s, s^{−1/2})` は等長変換ではないため、s ≠ 1 では同一クラス内の合同性が崩れ、長さが分布する。

```text
方向ごとのスケール係数の範囲 : [ s^{−1/2} , s ]
    s = 2.00 → [0.70711, 2.00000]   比 2.828 (= s^{3/2})
    s = 0.46 → [0.46000, 1.47442]   比 3.205 (= 1/s^{3/2})
```

各クラスについて次を保持する。

| 値 | 定義 | 用途 |
|---|---|---|
| `len` | クラス内の長さの（面積または本数で重み付けした）**幾何平均** | 代表周波数 |
| `spread` | クラス内の max/min 比 | detune 幅・スペクトル幅（§6.3） |
| `weight` | 面積合計または本数 | voice gain |

`det M = 1` により方向スケール係数の対数和が 0 になるため、**幾何平均長は s の変化でほとんど動かない。** これにより §6.3 が要求する「音域中心を保ったままスペクトル幅だけが伸縮する」が自動的に成立する。

### prolate と oblate の区別

s = 2.00（col）と s = 0.46（obl）は max/min 比が近い（2.828 と 3.205）ため、幅だけでは区別が弱い。両者は**分布の偏り**で区別される。

```text
prolate (s>1)  : xz 方向へ 0.707 倍。大半の辺が縮む
                 → 代表音域が上がり、少数の低い成分が伸びる

oblate  (s<1)  : xz 方向へ 1.474 倍。大半の辺が伸びる
                 → 代表音域が下がり、少数の高い成分が現れる
```

幾何平均と spread を素直に算出すればこの差は自然に出る。AC-05 の検証時はこの2状態を重点的に確認する（§26）。

## 23.5 クラス → voice スロットの割当

4クラス × 2モード = 8 voice。第2モードの比は面の多角形次数から導く非整数比とし、§9 の「ガラス／金属／膜／結晶」的な音色に寄与させる。

**クラスを `kind` index へ固定してはならない。**

区間1→2 の base 切替（`SEGS[1]` の `b:'ico'` → `SEGS[2]` の `b:'dod'`）では、幾何は同一の二十・十二面体でありながら kind の役割が反転する。

```text
ico 基底 τ=0.5 : kind=0 → 三角形 ,  kind=1 → 五角形
dod 基底 τ=0.5 : kind=0 → 五角形 ,  kind=1 → 三角形
```

`kind` index で voice スロットを固定すると、この境界で voice が瞬間的に別周波数へ飛ぶ。

**割当規則：4クラスを長さの降順に整列し、その順にクラス0〜3として voice スロットへ固定する。**

この規則が安全である理由は、クラスの順序交差が必ず「両クラスの長さが一致する点」で起きるためである。

```text
L_A = L_B となる τ  = 正則切頂の条件そのもの
    ico 基底 : τ = 1/3        （Truncated Icosahedron）
    dod 基底 : τ = 1/(2+φ)    （Truncated Dodecahedron）
```

順序が入れ替わる瞬間に両者の周波数は等しいため、可聴なジャンプが発生しない。また base 切替点（τ=0.5）では ico 基底・dod 基底のいずれも `L_A = 0`、`L_B = id の辺長` となり完全に連続である。

## 23.6 代表周波数（実測値・正規リファレンス）

`f = 110 · (1.2361 / L)` による、s = 1 の各状態の値。

| 状態 | クラスA | クラスB | クラスC | クラスD |
|---|---|---|---|---|
| Icosahedron | **110.0 Hz** | 0（無音） | 190.5 Hz | 0（無音） |
| Truncated Icosahedron | 330.0 Hz | 330.0 Hz | 330.0 Hz | 388.0 Hz |
| Icosidodecahedron | 0（無音） | 220.0 Hz | 381.1 Hz | 258.6 Hz |
| Truncated Dodecahedron | 398.0 Hz | 398.0 Hz | 246.0 Hz | 689.3 Hz |
| Dodecahedron | 178.0 Hz | 0（無音） | 209.2 Hz | 0（無音） |
| Small Stellated Dodecahedron | 178.0 Hz | 0（無音） | **110.0 Hz** | 0（無音） |

affine 変形状態（spn / col / obl）は上記の値を中心に、23.4 の係数範囲で分布する。

### この表から読み取れる設計上の含意

- **ti / td では A・B・C が同一周波数へ収束する。** これは正則切頂＝一様多面体であることの直接の帰結であり、純度の高いユニゾンとして §7「安定形では共鳴が収束」を満たす。重複 voice の除去は行わず、gain 正規化（§P1-7）で音量だけを整える。
- **τ → 0 でクラスB が、τ → 0.5 でクラスA が無音へフェードする。** 切頂そのものが聴覚化される（AC-06）。
- **Dodecahedron → Small Stellated Dodecahedron では、クラスC が 209.2 Hz → 110.0 Hz へ連続的に降下する。** σ による専用プリセットを持たずに AC-05 と §8 を満たす。なお SSD の側稜長 1.23607 は正二十面体の辺長と厳密に一致する（φ · a₀(dod) = 2/φ）。
- 個別 voice の最高周波数は、s = 0.46（obl）のクラスD 端で約 1.5 kHz、第2モードを含めて約 2.4 kHz に達する。**v1.0 が想定した 1760 Hz の上限で clamp すると、最も特徴的な状態でちょうど clamp が作動して形態差が潰れる。** これも hard clamp を廃止した理由である（§4.2）。

## 23.7 遵守事項

- deterministic であること（`Math.random()` を使わない。§25）
- 軽量であること
- 同じ形なら常に同じ結果になること
- **voice index が更新ごとに入れ替わらないこと**

Voice index が毎更新で入れ替わり、周波数が飛ぶ実装は禁止。

---

# 24. Voice tracking

形態変化中、各voiceが突然別周波数へジャンプしないようにする。音響品質上、ここは重要。

## 24.1 v1.1 の方式

**voice スロットは §23.5 の静的な割当規則で固定されるため、v1.0 が要求した nearest-frequency matching は不要である。**

```text
voice 0,1 → 長さ降順クラス0    voice 2,3 → クラス1
voice 4,5 → クラス2            voice 6,7 → クラス3
```

クラス数は常に4で変動しない。順序交差は両クラスの長さが一致する点でのみ起こる（§23.5）。したがってフレーム間で voice が担当する周波数は連続である。

v1.0 §24 の「previous modes → nearest-frequency matching → new modes」は**不要**として廃止する。

## 24.2 ジャンプを起こしうる箇所と対策

静的割当でもなお不連続が生じうる箇所は次の3つ。いずれも対策を必須とする。

| # | 箇所 | 対策 |
|---|---|---|
| 1 | base 切替（`SEGS[1]` → `SEGS[2]`、ico→dod）での kind 反転 | クラスを `kind` index へ固定せず長さ降順で整列する（§23.5） |
| 2 | クラス長が 0 へ退化（τ=0 / τ=0.5） | 周波数を更新せず直前値を保持し、gain を 0 へフェード（§18.1） |
| 3 | 非有限値（Infinity / NaN）の混入 | AudioParam へ渡す直前に `Number.isFinite()` 検査（§18.1 / AC-15） |

## 24.3 平滑化

周波数・gain とも瞬間切替を禁止し、Web Audio automation で平滑化する（§12）。

```text
frequency : setTargetAtTime(target, ctx.currentTime, 0.08)   時定数 約80ms
gain      : setTargetAtTime(target, ctx.currentTime, 0.05)   時定数 約50ms
```

1区間の遷移は約 3.84 秒（`SEG_DUR` 6.4秒 × 0.60）であり、上記の時定数で追従しつつ滑らかになる。

---

# 25. 音のランダム性

ランダム値を使用する場合でも、

- chaosに応じた微細揺らぎのみ
- 音の構造そのものをrandomにしない
- 再読込ごとに全く異なる作品にならない

こと。

可能ならdeterministicな関数を優先する。

---

# 26. 受入条件

## AC-01

ページロード時、音は鳴らない。

## AC-02

`SOUND — OFF` からユーザー操作で `SOUND — ON` にできる。

## AC-03

外部音声ファイルへの通信が存在しない。

## AC-04

Sound ON後、現在の多面体形状から音がリアルタイム生成される。

## AC-05

シーケンス上の名前付き状態すべてで、隣接状態と聴覚的な差が確認できる。

- Icosahedron
- Truncated Icosahedron
- Icosidodecahedron
- Truncated Dodecahedron
- Dodecahedron
- Small Stellated Dodecahedron
- Prolate Stellation
- Prolate Truncated Dodecahedron
- Oblate Truncated Dodecahedron

v1.1 で `Truncated Dodecahedron` と `Prolate Truncated Dodecahedron` を追加した（v1.0 では欠落）。形態ごとの追加音域正規化を廃止した（§4.2）主たる根拠がこの受入条件であるため、リストは網羅されている必要がある。

### 重点確認ペア

| ペア | 差が出る理由 | 難易度 |
|---|---|---|
| Dodecahedron ↔ Small Stellated Dodecahedron | クラスC が 209.2 → 110.0 Hz へ降下（§23.6） | 易 |
| Icosahedron ↔ Dodecahedron | 基音 110.0 Hz ↔ 178.0 Hz | 易 |
| **Prolate Truncated D. ↔ Oblate Truncated D.** | τ・σ が同一で s のみ異なる（2.00 / 0.46）。spread 比が近いため、分布の偏り（§23.4）でのみ区別される | **難。最重点** |

## AC-06

変相途中で周波数・倍音が連続変化し、安定形で収束する。

## AC-07

`chaos = 0` 近傍では大きなdetuneが残らない。

## AC-08

`σ` 増加時、星形化に対応したbrightness / harmonic complexityの増加を確認できる。

## AC-09

`s` 変化時、スペクトル幅またはregisterの変化が確認できる。

## AC-10

Sound ON/OFF時にクリックノイズを発生させない。

## AC-11

SPACE HOLD中、形と音響状態が同じ位置で保持される。

## AC-12

Sound OFFでも既存作品と同等に動作する。

## AC-13

Audio初期化失敗時でもWebGL描画は停止しない。

## AC-14

Sound追加前後で既存Shader・Geometry・Caption・Ring・Piece・Cameraの表示結果を意図的に変更していない。

照合基準は Phase 0 で保存した visual baseline スクリーンショットとする（§28）。

## AC-15

`τ = 0`（Icosahedron / Dodecahedron）および `τ = 0.5`（Icosidodecahedron）を通過しても、Infinity / NaN による無音化・例外が発生しない。

通過後も全 voice が正常に復帰する（§18.1）。

## AC-16

SOUND トグルをクリックした直後に SPACE を押しても、SOUND が再トグルされない。

HOLD は従来どおり動作する（§13.2）。

## AC-17

ビューポート幅 680px 以下でも SOUND トグルが表示され、タップで ON / OFF できる（§13）。

---

# 27. 非目標

v1.1でも以下を行わない。

- MIDI
- Web MIDI API
- microphone input
- external synth
- Tone.js
- DAW連携
- downloadable audio
- recording
- waveform visualization
- spectrum analyzer UI
- BPM
- sequencer
- melody generation
- AI music generation
- spatial audio HRTFの本格実装
- physical FEM acoustic simulation

---

# 28. 実装フェーズ

## Phase 0 — 保護 ✅ 完了

1. ✅ 現在のHTMLをgit commit。
2. ✅ Sound追加前の動作を保存。
3. ✅ visual baselineのスクリーンショットを保存。
4. ✅ Sound branchまたは明確なcommit単位で作業。

実施結果：

```text
main                 3363847   Sound追加前の visual baseline（ロールバック先）
feature/sonification 68d4c6d   polyhedra_transfiguration (4).html → index.html
```

`.backup-YYYYMMDD` コピーは作成しない。git の初期コミットがバックアップを兼ねる。

### baseline FPS 計測について（v1.1 変更）

**baseline FPS の計測は Phase 0 の必須項目としない。**

AC-14 が要求する描画差分の検証は、Phase 0 で保存した visual baseline スクリーンショットで足りる。FPS は Phase 4 の性能確認において、体感上の差が疑われた場合にのみ計測する。

計測する場合もHTMLへ計測コードを埋め込まず、ブラウザの開発者ツールで行う。

## Phase 1 — Audio skeleton

1. SOUND UI（`<button>`、`pointer-events:auto`、モバイル表示、`aria-pressed`）— §13
2. SPACE HOLD のフォーカスガード — §13.2 / AC-16
3. AudioContext（`webkitAudioContext` フォールバック、ユーザージェスチャ内で生成）
4. Voice Mix → Compressor → Master Gain → Destination — §11
5. 8 voices
6. Sound ON/OFF、fade in/out、OFF完了後の `ctx.suspend()`
7. 初期化失敗時のフォールバック — §18

この段階では固定周波数でよい。

検証：AC-01 / AC-02 / AC-10 / AC-12 / AC-13 / AC-14 / AC-16 / AC-17

## Phase 2 — Geometry metrics

1. `curChaos` の公開（既存コードへの2行の変更のうち1行）— §2
2. 25Hz の独立した読み取り専用パスを `frame()` から呼ぶ — §5.2 / §12
3. 静的分類テーブル（辺クラスA/B、面クラスC/D）をモデルごとに初回1回構築 — §23
4. クラス代表長・spread・weight の算出 — §23.4
5. 長さ降順の整列と voice スロット固定 — §23.5
6. `f = f0 · (L_ref / L)` による周波数写像 — §4.2
7. 非有限値防御と退化クラスの gain fade — §18.1
8. smoothing — §24.3

ここで「形が鳴る」状態にする。

検証：AC-04 / AC-05 / AC-06 / AC-15

## Phase 3 — Transformation mapping

1. `σ` — brightness / filter cutoff / 非整数倍音（§6.2）
2. `s` — spread → detune 幅・スペクトル幅（§6.3 / §23.4）
3. `chaos` — micro-detune / beating / inharmonicity（§6.4）

検証：AC-07 / AC-08 / AC-09

## Phase 4 — Polish

1. gain normalization（ti / td のユニゾン収束時の音量を含む）
2. filter調整
3. transition調整
4. browser確認
5. performance確認（必要に応じて baseline FPS を計測）

## Phase 5 — Optional spatialization

Orbit連動を追加（§15）。HOLD 中の挙動を §14 で確認する。

---

# 29. 変更影響範囲

## 影響範囲

主に以下。

```text
HTML UI            SOUND button の追加
CSS                #snd 用ルール、680px以下での表示制御
JavaScript         Sonification セクションの新設（独立ブロック）
Web Audio          initialization / graph / voices
既存JS 3箇所       curChaos 公開 / frame() からの呼び出し / SPACE フォーカスガード（§2）
```

v1.0 が挙げていた「buildModel / update周辺のaudioMetrics取得」は、独立した読み取り専用パスへ変更したため**影響範囲から外れる**（§5.2）。`buildModel()` と `M.update()` は無変更。

## 変更しない範囲

```text
既存Geometry生成式
既存Shader
既存FORM定義
既存SEGS
既存Camera挙動
既存Caption
既存Ring
既存Piece
既存色・タイポグラフィ
```

---

# 30. 戻し方

Sound実装は既存描画から疎結合にする。

ロールバック可能条件：

1. Sound関連コードを削除または無効化すれば元のWebGL作品へ戻れる。
2. Geometry計算の本体ロジックはSound用に書き換えない。`buildModel()` / `M.update()` は無変更（§2 / §5.2）。
3. Sound用metrics取得は読み取り専用とし、描画座標へ副作用を持たせない。
4. git上でSound導入前commitへ戻せる状態で作業する。

ロールバック先は `main` の **`3363847`**（Phase 0 の visual baseline）。

---

# 31. 実装時の判断原則

迷った場合の優先順位：

```text
1. 元の作品を壊さない
2. 「形が鳴る」という因果関係を守る
3. 音を気持ちよくする
4. 数学と音の対応を説明可能にする
5. 実装を軽量にする
6. 演出を追加する
```

音が美しくても、幾何との対応を説明できなければ採用しない。

逆に数学的に厳密でも、耳障りで作品として成立しなければ調整する。

---

# 32. 完成時の説明文

作品説明として以下が成立する状態を目標とする。

> TRANSFIGURATIONでは、多面体の形態変化と音響を別々に作っていません。
>
> τ、σ、sによって変化する幾何形状から、辺長・面・空間情報を取得し、その構造を共鳴周波数へ変換しています。
>
> そのため、画面上の形が変わると音も変わります。
>
> BGMや録音素材ではなく、見えている多面体そのものを仮想的な共鳴体として鳴らしています。

---

# 33. Claudeへの引き継ぎ事項

既存HTML全体の確認は **v1.1 時点で完了済み**。以下は確認結果であり、実装時に再確認する必要はない。

| # | 確認項目 | 結果 |
|---|---|---|
| 1 | `buildModel()` の Geometry 生成フロー | `topology` → `truncation` → 固定長 TypedArray 確保 → `update()` クロージャ。`pairs`(60) と `faces`(32) は τ/σ/s に依らず静的 |
| 2 | `update()` 内で利用可能な幾何情報 | `P[]`・面ごとの `C` / `N` / `Rf` / `Q[]` / `AP`・`A`,`B`・`maxR`。ただし v1.1 では利用しない（§5.2） |
| 3 | `applySequence(dt)` の `u / chaos / tau / sig / st` | `u` は raw 0.20→0.80 を smootherstep。両端に各20%（1.28秒）の静止区間。`chaos = sin(πu)`。`s` は対数線形補間。`chaos` のみ module スコープに未公開 |
| 4 | `frame()` の更新順 | `applySequence` → caption → ring → piece → camera → `updateRing` → RT描画 → コンポジット |
| 5 | `.ui` の `pointer-events:none` | 確認。加えて **`.br` は `max-width:680px` で `display:none`**（§13 で対応） |
| 6 | 既存の `SPACE — HOLD` | `window` レベルの `keydown`。`paused` は `phase` の進行のみ停止。`elapsedTime` と `root.rotation` は継続（§14） |
| 7 | Shader / Caption / Ring / Piece への副作用 | §5.2 の読み取り専用パスにより構造的に副作用なし |

### 実装時の必読章

v1.0 から方式が変わっている章があるため、実装前に以下を読むこと。

```text
§2     接続点と既存コードへの変更3箇所
§4.2   周波数式（clamp廃止・形態別正規化廃止）
§5.2   独立した読み取り専用パス
§11    Audio Graph の接続順
§13.2  button とフォーカスガード
§18.1  非有限値の防御
§23    幾何クラス分類（v1.0 のクラスタリングを全面置換）
§24    voice スロットの静的割当
```

### 原則

**既存ビジュアルを「改善」しない。**

Sound追加に必要な最小変更のみ実施する。

まずPhase 1〜2までを実装し、音を確認してから `σ / s / chaos` のチューニングへ進む。

---

# 34. 完成定義

v1.0完成条件：

> ユーザーがページを開き、SoundをONにすると、画面上の多面体が変相するのに同期して、同じ幾何学的状態から生成された共鳴音が連続的に変化する。音声素材・BGMは使用しておらず、SoundをOFFにすれば既存のTRANSFIGURATION作品としてそのまま動作する。

以上。

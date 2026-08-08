# TRANSFIGURATION — Geometry Sonification 要件定義書

**Version:** 1.0  
**Date:** 2026-08-08  
**Target:** `TRANSFIGURATION — 多面体変相` WebGL作品  
**Current source:** `polyhedra_transfiguration (4).html`  
**Purpose:** 多面体・数式によるリアルタイム音響生成（Sonification）の追加  
**Status:** 実装前要件定義

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

既存コード上では、

- `buildModel(base)`
- `M.update(tau, sigma, st, chaos)`
- `applySequence(dt)`
- `curTau`
- `curSig`
- `curSt`
- `chaos`
- `MODEL.ico`
- `MODEL.dod`
- `frame()`

が音響側との主要な接続点になる。

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

v1.0では **8 voice程度の固定数の共鳴モード** を使用する。

推奨：

```text
8 resonant voices
+
Master Gain
+
Dynamics Compressor / Limiter相当
```

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

### 必須処理

1. 現在の幾何形状から代表的な長さ群を抽出する。
2. 類似長をクラスタリングする。
3. 上位最大8モードへまとめる。
4. 長さの逆数比を周波数比へ変換する。
5. 聴覚上の極端な低域・高域はClampする。
6. 全形態で基準音域が極端に移動しないよう正規化する。

推奨初期範囲：

```text
55 Hz ～ 1760 Hz
```

基準周波数は固定値を使用してよい。

例：

```text
f0 = 110 Hz
```

ただし「音階へ量子化」しない。

幾何学的な比率を維持する。

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

## 5.2 audioMetrics

`buildModel()` またはその周辺へ、描画とは独立した軽量な音響メトリクスを追加する。

概念：

```js
audioMetrics = {
  edgeLengths: [],
  faceAreas: [],
  centers: [],
  normals: [],
  radius: 0
}
```

`M.update()` の計算中に、すでに算出している

- `A`
- `B`
- `AP`
- `C`
- `N`
- `Rf`

等を利用して更新する。

### 制約

- 描画結果を変更しない。
- 音響用データ取得のためだけに同一幾何を大規模再計算しない。
- 毎フレーム大量のObject生成を行わない。
- 可能ならTypedArrayまたは再利用Arrayを使う。

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

v1.0では外部Audio libraryを導入しない。

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

概念：

```text
Voice 1 ─┐
Voice 2 ─┤
Voice 3 ─┤
Voice 4 ─┤
Voice 5 ─┤
Voice 6 ─┤→ Voice Mix → Master Gain → Compressor → Destination
Voice 7 ─┤
Voice 8 ─┘
```

P2では各VoiceまたはVoice Groupの前段へStereoPannerを追加できる。

---

# 12. パラメータ更新

WebGLは毎フレーム描画を継続する。

Audio制御値の重い計算は必ずしも60fpsで行わない。

推奨：

```text
Geometry render : requestAnimationFrame
Audio metrics   : 20–30 Hz程度
Audio smoothing : Web Audio automation
```

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

---

## 13.1 初期状態

```text
SOUND — OFF
```

必須。

ページロード時にAudioContextを自動開始しない。

ユーザーがSoundをONにした操作をトリガーとしてAudioContextを生成／resumeする。

---

## 13.2 UI実装上の注意

既存 `.ui` は `pointer-events:none` を使用しているため、Sound toggleはクリック可能な独立要素または対象部分のみ `pointer-events:auto` とする。

既存レイアウト・タイポグラフィを崩さない。

派手なボタン、アイコン、スライダーは追加しない。

v1.0では音量スライダーは不要。

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

- voice数：原則8
- AudioNodeを毎フレーム生成しない
- Sound OFF時は不要処理を停止
- 毎フレームのArray/Object allocationを最小化
- 幾何メトリクス抽出は既存計算を再利用
- パラメータ更新をsmooth化

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
- Master段にDynamicsCompressorNode等を配置
- 初回ON時に大音量を出さない

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
- AudioContext resume failure
- StereoPanner非対応
- その他Audio node生成エラー

エラーによって`frame()`を停止しない。

Sound機能はWebGL描画から疎結合にする。

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
    - Geometry metrics
    - Frequency clustering
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
  enabled: false,
  ctx: null,
  master: null,
  compressor: null,
  voices: [],
  metrics: null,
  lastUpdate: 0
};
```

必要以上にグローバル状態を増やさない。

---

# 22. Geometry → Audio変換フロー

```text
M.update(tau, sigma, st, chaos)
        ↓
current geometry
        ↓
audioMetrics
  ├─ representative lengths
  ├─ areas
  ├─ centers
  └─ normals
        ↓
cluster / normalize
        ↓
8 modal frequencies
        ↓
σ / s / chaos modulation
        ↓
parameter smoothing
        ↓
Web Audio nodes
        ↓
sound
```

---

# 23. クラスタリング

辺長をそのまま8本選択するのではなく、似た長さをまとめる。

目的：

- 同じ長さの辺が大量に存在する正多面体で不要な重複voiceを作らない
- 形が崩れる途中では微妙な長さ差が徐々に分離する
- 安定形では共鳴モードが自然に収束する

実装手法はClaude側で選定可。

ただし、

- deterministic
- 軽量
- 毎回同じ形なら同じ結果
- フレームごとのvoice順序入替を抑える

こと。

Voice indexが毎更新で入れ替わり、周波数が飛ぶ実装は禁止。

---

# 24. Voice tracking

形態変化中、クラスタ数や順序が変わっても各voiceが突然別周波数へジャンプしないようにする。

推奨：

```text
previous modes
↓
nearest-frequency matching
↓
new modes
```

または同等の安定化処理。

音響品質上、ここは重要。

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

少なくとも以下の状態間で聴覚的な差が確認できる。

- Icosahedron
- Truncated Icosahedron
- Icosidodecahedron
- Dodecahedron
- Small Stellated Dodecahedron
- Prolate Stellation
- Oblate Truncated Dodecahedron

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

---

# 27. 非目標

v1.0では以下を行わない。

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

## Phase 0 — 保護

1. 現在のHTMLをgit commit。
2. Sound追加前の動作を保存。
3. visual baselineのスクリーンショットを保存。
4. Sound branchまたは明確なcommit単位で作業。

## Phase 1 — Audio skeleton

1. AudioContext
2. Master Gain
3. Compressor
4. 8 voices
5. Sound ON/OFF
6. fade in/out

この段階では固定周波数でもよい。

## Phase 2 — Geometry metrics

1. current geometryから辺長候補を取得
2. clustering
3. frequency mapping
4. voice tracking
5. smoothing

ここで「形が鳴る」状態にする。

## Phase 3 — Transformation mapping

1. `σ`
2. `s`
3. `chaos`

を音色へ反映。

## Phase 4 — Polish

1. gain normalization
2. filter調整
3. transition調整
4. browser確認
5. performance確認

## Phase 5 — Optional spatialization

Orbit連動を追加。

---

# 29. 変更影響範囲

## 影響範囲

主に以下。

```text
HTML UI
JavaScript
Web Audio initialization
buildModel / update周辺のaudioMetrics取得
frame loop内のaudio update
```

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
2. Geometry計算の本体ロジックはSound用に書き換えない。
3. Sound用metrics追加は描画座標へ副作用を持たせない。
4. git上でSound導入前commitへ戻せる状態で作業する。

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

この要件定義を実装する場合、最初に既存HTML全体を読み、以下を確認してから変更する。

1. `buildModel()` 内のGeometry生成フロー
2. `update(tau, sigma, st, chaos)` 内で利用可能な幾何情報
3. `applySequence(dt)` の `u / chaos / tau / sig / st`
4. `frame()` の更新順
5. `.ui` の `pointer-events:none`
6. 既存の `SPACE — HOLD`
7. Shader / Caption / Ring / Pieceへの副作用がないこと

**既存ビジュアルを「改善」しない。**

Sound追加に必要な最小変更のみ実施する。

まずPhase 1〜2までを実装し、音を確認してから `σ / s / chaos` のチューニングへ進む。

---

# 34. 完成定義

v1.0完成条件：

> ユーザーがページを開き、SoundをONにすると、画面上の多面体が変相するのに同期して、同じ幾何学的状態から生成された共鳴音が連続的に変化する。音声素材・BGMは使用しておらず、SoundをOFFにすれば既存のTRANSFIGURATION作品としてそのまま動作する。

以上。

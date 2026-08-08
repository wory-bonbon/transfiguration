# TRANSFIGURATION

**Canonical Truncation Sequence / 多面体変相**

多面体の数式を眺めていたら、  
形が変わるなら、音も変わるのではないかと思いました。

![TRANSFIGURATION](./assets/transfiguration-raw.webp)

頂点、辺、面から生まれる幾何学的な長さを、  
そのまま耳に聞こえる周波数へ写像しています。

音楽を付けたのではなく、  
形を少しだけ「聴けるもの」にしています。

> *What would a polyhedron sound like if its geometry were allowed to speak for itself?*

[![TRANSFIGURATION Demo](./assets/demo-thumbnail.webp)](https://youtu.be/z0JAAQVdsyY)

[▶ Watch Demo](https://youtu.be/z0JAAQVdsyY) | [◇ Try it live](https://wory-bonbon.github.io/transfiguration/)

## RAW Geometry Sonification

辺や面の長さを測り、その幾何学的な比率を周波数へ変換しています。

長い部分は低く。  
短い部分は高く。

音階やコードへ整えることはせず、  
多面体が持つ数値をなるべくそのまま音として残しています。

## Controls

- **Drag** — Orbit
- **Scroll** — Dolly
- **Space** — Hold
- **Sound** — On / Off

## Technical Note

Built with `Three.js` and `Web Audio API`.

The geometry is generated and transformed in real time.  
The sound is generated from the geometry itself — no soundtrack or audio samples.

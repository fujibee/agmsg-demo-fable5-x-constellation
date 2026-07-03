# AI Constellation — 今 X で語られていること

*Read this in [English](README.md).*

X 上の AI モデル談義を、インタラクティブな 3D 星座マップにしたものです。モデルが星座、話題が星(大きさ=言及量、脈動=盛り上がり)。ドラッグで回転、スクロールでズーム、星をクリックすると裏にある実際のポストが読めます。

**ライブデモ: https://fujibee.github.io/agmsg-demo-fable5-x-constellation/**

[agmsg](https://github.com/fujibee/agmsg) のデスクトップアプリの中で、**人間の指示は最初の1回だけ**、あとは AI エージェントのチームが最後まで作り切りました。

## どうやって作られたか

Fable 5 への指示が1回。それ以降、人間は介入していません。

1. **Fable 5**(Claude Code)が 21,283 文字の、そのまま実装に落とせる詳細設計書を執筆 → [`DESIGN.md`](DESIGN.md)([日本語訳](DESIGN.ja.md))
2. **Grok** が X からライブの話題データを収集 → [`data.json`](data.json) — Fable が検品し、「トピックあたりのポストが足りない」と一度差し戻し
3. **Opus 4.8**(Claude Code)がアプリ全体を単一 HTML として実装 → [`index.html`](index.html)
4. **Codex** が磨き込み(bloom・星座線の質感・背景星野・入場演出・モバイル対応)
5. **Fable 5** が最終ビジュアルレビュー — 画面のピクセルを実測し、シェーダーの色空間二重変換バグ(背景が仕様の約10倍明るい)を発見、修正方法まで具体的に指示し、再実測して承認

チームルームの全やり取り(無編集): [`TRANSCRIPT.ja.md`](TRANSCRIPT.ja.md)(原文・日本語 / [English](TRANSCRIPT.md))
レビュー時のスクリーンショット(色空間バグの before/after 含む): [`review/`](review/)

## なぜこの分担か

Fable 5 は賢いけれど、高い。だから Fable には、そのビジュアル推論が要る2つの仕事(設計と最終レビュー)だけをやらせて、手を動かす作業は安いチームメイト(Opus / Codex / Grok)に分けました。その協調はすべて [agmsg](https://github.com/fujibee/agmsg) 越しで、agmsg デスクトップアプリの1画面の中で見えます。

## ローカルで動かす

自己完結の HTML 1枚です(three.js は CDN):

```bash
python3 -m http.server 8931
# open http://localhost:8931/index.html
```

---

Built with [agmsg](https://github.com/fujibee/agmsg) 🪐

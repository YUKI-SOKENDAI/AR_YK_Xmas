# ARクリスマスカード制作記録 ― 3Dモデル作成からWebAR表示まで ―

## 概要

本記事では、
3Dキャラクターモデルを制作し、WebARとして安定表示するクリスマスカードを作成した際の実装手順とハマりどころをまとめる。

特に以下を重点的に扱う。

- Blenderでの キーフレームアニメーション追加

- glb化した際の AR環境でのテクスチャ透け問題

- A-Frame + AR.js（hiroマーカー） を用いた安定AR構成

- echo3D 等を使わず、自前WebARで複数アニメーションを再生する方法

## 採用した最終構成
| 項目 |	内容 |
|3Dモデル生成|Meshy.ai|
|3D編集・アニメーション|Blender|
|フォーマット|glTF Binary（.glb）|
|WebAR|A-Frame + AR.js|
|マーカー|hiroマーカー|
|配信|GitHub Pages|
	
## なぜマーカーARを選択したか

WebARには以下の選択肢がある。

方式	特徴
マーカーレスAR（WebXR）	没入感は高いが端末依存が強い
マーカーAR（AR.js）	iOS / Android 両対応で非常に安定

今回の用途は 不特定多数に配布するクリスマスカードであり、

読み込み失敗しない

カメラ許可だけで動く

印刷物と相性が良い

という理由から、hiroマーカー方式を採用した。

全体の制作フロー
① Meshy.ai で3Dモデル生成
② Blenderでモデル調整・アニメーション追加
③ glb形式でエクスポート
④ WebAR（A-Frame + AR.js）実装
⑤ GitHub Pages で公開
⑥ QRコードで配布

① 3Dモデル作成（Meshy.ai）
使用理由

人型（サンタ）＋非人型（カメ）のモデルをまとめて生成できる

歩行などの基本モーションを付与可能

作業内容

サンタ＋カメの3Dモデル生成

歩行モーションを付与

glb形式でエクスポート

② Blenderでの調整・アニメーション追加

Meshy.ai から出力したモデルを Blenderで仕上げる工程。

スケール調整

AR用途では 想定よりかなり小さめが適切

最終的には Web 側で scale="0.2" 程度を使用

キーフレームアニメーションの追加（重要）

Meshy.ai 側のモーションに加えて、以下を Blender側で追加した。

空を飛んでいるように見せるための 位置（Location）

モデルの 回転（Rotation）

円軌道を描くような動き

手順概要

タイムラインを表示

フレームを移動

Location / Rotation を変更

I → Location / Rotation を選択

複数フレームにキーフレームを設定

Dope Sheet / Action Editor でループ確認

結果として：

歩行モーション（Meshy.ai）

飛行・旋回モーション（Blender）

が 同一glb内に複数Actionとして存在する状態になる。

③ AR時に発生した「テクスチャ透け問題」（最重要）
発生した問題

Blender上では正常に見える

glbを書き出して ARで表示すると、背面から内部が透けて見える

視点角度によってモデル内部が見えてしまう

特に echo3D や AR.js 上で顕著に発生。

原因

AR環境では、

片面描画（Backface Culling）

マテリアルのアルファ設定

透過ブレンド設定

が BlenderのViewport表示と異なる挙動をする。

対応内容（実際に効果があった設定）
Shader Editor（Principled BSDF）

Principled BSDF を使用

Alpha = 1.0

透過ノード（Transparent BSDF 等）を使用しない

テクスチャは Base Color のみ接続

Material Properties

ブレンドモード：Opaque（不透明）

透過系のレンダリング設定を使わない

👉
これにより AR上での透け問題が解消された。

※ Blenderで透けていなくても、
AR環境で必ず確認が必要なポイント。

④ glbエクスポート設定（Blender）

Format：glTF Binary (.glb)

☑ Animation

☑ Apply Modifiers

☑ Always Sample Animations

⑤ WebAR実装（A-Frame + AR.js）
ポイント

glb内の 複数アニメーションを同時再生

AR.js標準では 1つのアニメーションしか再生されないため、
Three.js の AnimationMixer を使用

⑥ 最終的に使用した index.html

以下が 最終版・実際に使用したコード。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <title>WebAR Christmas Card</title>

  <!-- A-Frame -->
  <script src="https://aframe.io/releases/1.5.0/aframe.min.js"></script>

  <!-- AR.js -->
  <script src="https://cdn.jsdelivr.net/gh/AR-js-org/AR.js/aframe/build/aframe-ar.js"></script>

  <script>
    AFRAME.registerComponent('multi-animation', {
      init: function () {
        const el = this.el;

        el.addEventListener('model-loaded', () => {
          const model = el.getObject3D('mesh');
          const animations = el.components['gltf-model'].model.animations;
          this.mixer = new THREE.AnimationMixer(model);
          animations.forEach((clip) => {
            this.mixer.clipAction(clip).play();
          });
        });
      },
      tick: function (time, deltaTime) {
        if (this.mixer) {
          this.mixer.update(deltaTime / 1000);
        }
      }
    });
  </script>
</head>

<body style="margin: 0; overflow: hidden;">
  <a-scene
    embedded
    arjs="sourceType: webcam; debugUIEnabled: false;"
    renderer="colorManagement: true;"
  >
    <a-marker preset="hiro">
      <a-entity
        id="christmas-model"
        gltf-model="url(SantaKame_06_text.glb)"
        multi-animation
        position="0 0 0"
        rotation="0 0 0"
        scale="0.2 0.2 0.2">
      </a-entity>
    </a-marker>

    <a-entity camera></a-entity>
  </a-scene>
</body>
</html>

```

まとめ

WebARは 用途に合った技術選定が最重要

印刷物と組み合わせる場合、マーカーARが最も安定

Blenderでのアニメーション追加は Action管理が鍵

AR環境でのテクスチャ透け問題は必ず検証が必要

echo3D を使わなくても、
A-Frame + AR.js で 十分実用的なAR演出が可能

おわりに

今回のような イベント用途・配布前提ARでは、

「最先端」より「確実に動く」

が正解になるケースが多い。

本記事が、
これから WebARを自作したい人の助けになれば幸いです。

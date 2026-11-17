
# AR Christmas Card — Development Handoff

## 1. プロジェクト概要

紙のクリスマスカードにQRコードとARマーカーを印刷し、スマートフォンのブラウザでアニメーション付き3Dモデルを表示するWebARシステムを制作した。

最終的に、hiroマーカーを認識して、サンタとカメ、および装飾文字のアニメーションを表示する方式で完成した。

### 最終成果物

- サンタ＋カメの3Dキャラクター
- Meshy.aiで生成した歩行モーション
- Blenderで追加した飛行・旋回アニメーション
- 曲率に沿って変形した3D文字
- 文字の円軌道回転アニメーション
- 複数アニメーションを同時再生するWebARページ
- GitHub PagesによるHTTPS配信
- QRコードとhiroマーカーを利用する紙のクリスマスカード

### リポジトリ

https://github.com/YUKI-SOKENDAI/AR_YK_Xmas

主要ファイル：

- `docs/index.html`
- `docs/SantaKame_06_text.glb`

GitHub Pages想定URL：

https://yuki-sokendai.github.io/AR_YK_Xmas/

---

## 2. 技術構成

| 工程 | 使用技術 |
|---|---|
| AIによる3D生成 | Meshy.ai |
| 初期モーション生成 | Meshy.ai |
| 3D編集 | Blender（日本語UI） |
| アニメーション編集 | Blender Keyframe / Action Editor / Graph Editor |
| 3Dファイル | glTF Binary (.glb) |
| WebAR | A-Frame 1.5.0 |
| マーカー認識 | AR.js |
| アニメーション再生 | Three.js AnimationMixer |
| Webホスティング | GitHub Pages |
| ARマーカー | hiro |

---

## 3. 3Dモデルの制作

### 3.1 Meshy.ai

Meshy.aiを利用してサンタとカメが一緒になった3Dモデルを生成した。

Mixamoも検討したが、人型のリギングとモーションに適しており、カメのような四足動物を含む複合モデルには適用しにくかった。

そこで、Meshy.aiのアニメーション機能を使い、歩行モーションを追加した。

### 3.2 Blenderへの転送

Meshy.aiのDCC Bridgeを利用してBlenderにモデルを転送した。

Blender側ではMeshyプラグインを実行し、モデルの受信を行った。

インポート後、メッシュ、Armature、マテリアル、アニメーションを確認した。

---

## 4. Blenderでの編集

### 4.1 モーションの確認

Blenderのタイムラインで歩行モーションを確認した。

Action EditorとGraph Editorを使用してアニメーションを確認・編集した。

Graph EditorではCycles F-Modifierを利用し、歩行モーションをループ化した。

### 4.2 スケール調整

Blenderでモデルのスケールを調整した。

WebAR側でも追加のスケール調整を行い、最終HTMLでは以下を使用した。

```html
scale="0.2 0.2 0.2"
```

### 4.3 飛行アニメーション

Meshy.aiの歩行モーションに加え、Blenderでモデル全体の位置と姿勢をキーフレーム制御した。

制御パラメータ：

- Location X
- Location Y
- Location Z
- Rotation X
- Rotation Y
- Rotation Z

約240フレームで元の位置に戻る円軌道を作成した。

位置・回転を各キーフレームで指定し、Blenderの補間機能を利用した。

当初はEmptyを親オブジェクトにする方式を検討したが、親子設定がうまくいかなかったため、Armatureオブジェクト自体のTransformを制御する方式を採用した。

### 4.4 文字の3D化

Blenderで文字を3Dモデルとして配置した。

カーブモディファイアなどを利用し、文字列を指定した円弧に沿って変形した。

さらに、円軌道に沿って文字を回転させるアニメーションを作成した。

### 4.5 複数周期のアニメーション

サンタと文字は、それぞれ独立した周期で動くようにした。

複数のAnimationClipを含むGLBとして出力し、WebAR側で個別にループ再生する構成とした。

注意：BlenderのActionはオブジェクトやデータブロックに関連するため、異なるオブジェクトのActionを単純に1つへコピーすれば統合できるとは限らない。

---

## 5. AR表示時の透過問題

### 5.1 症状

Blender上ではテクスチャ付きモデルが正常に表示されていた。

しかし、GLBにエクスポートし、echo3DやOnirixなどのAR環境に読み込むと、モデルを背面・斜め背面から見た際に内部が透けて見える問題が発生した。

### 5.2 調査した要因

- Backface Culling
- Alpha設定
- テクスチャのAlphaチャンネル
- マテリアルの描画方式
- メッシュの形状
- Solidifyによる厚み
- Metallic / Roughness / IOR

### 5.3 実施した対処

#### メッシュ

Blenderでメッシュのクリーンアップ・最適化を実施した。

Solidify（ソリッド化）モディファイアによる厚み付けも試した。

ただし「均一な厚み」を有効にすると、AI生成モデルの微細な形状ノイズが強調されたため、設定を調整した。

法線の再構築は最終的な実施工程には含めていない。

#### シェーダー

BlenderのShader EditorでPrincipled BSDFを編集した。

当初、Image TextureのColor出力がBase Colorに、Alpha出力がPrincipled BSDFのAlphaに接続されていた。

このAlpha接続を解除し、Alphaを1.0に設定した。

さらに、キャラクター用の非金属マテリアルとしてMetallicを0に変更した。

設定例：

| 項目 | 設定 |
|---|---|
| Base Color | Image TextureのColor |
| Alpha | 1.0、画像Alpha接続なし |
| Metallic | 0 |
| Roughness | 約0.6 |
| IOR | 約1.45 |
| Transmission | 0 |

#### Blender日本語UIの描画設定

マテリアルプロパティ内の「サーフェス」で、レンダーメソッドなどの設定を確認した。

Blenderのバージョンによって設定名が異なるため、旧バージョンの「不透明」と新バージョンの「ブレンド」を単純に同一視しないこと。

重要なのは、GLBに書き出されたマテリアルが意図せず半透明扱いにならないようにすること。

### 5.4 結果

上記のマテリアル設定調整後、AR表示時の透過問題が解消された。

ただし、どの変更が単独で決定的だったかは分離検証していない。

### 5.5 再発時の確認項目

1. Image TextureのAlphaがPrincipled BSDFのAlphaに接続されていないか
2. Alphaが1.0か
3. GLBマテリアルのalphaModeがOPAQUEか
4. doubleSidedの設定
5. MetallicとTransmissionの値
6. GLBを外部ビューアで表示した際にも問題が発生するか

---

## 6. GLBエクスポート

BlenderからglTF Binary形式で出力した。

主な設定：

- Format: glTF Binary (.glb)
- Animation: 有効
- モディファイアのエクスポート反映
- マテリアルとテクスチャを含める

最終モデル：

`SantaKame_06_text.glb`

開発途中のGLBは約75MBだった。

このサイズはスマートフォンでのWebAR用途には大きいため、今後の軽量化が課題。

---

## 7. WebARの実装

### 7.1 初期検討

echo3DおよびOnirixを検討した。

複数アニメーションの同時再生を自由に制御するため、A-Frame + AR.js + Three.jsによるWebAR実装へ移行した。

### 7.2 アニメーション制御

GLBに含まれるAnimationClipを読み込み、Three.jsのAnimationMixerで全クリップを再生する。

```javascript
const model = el.getObject3D('mesh');
const animations = el.components['gltf-model'].model.animations;

this.mixer = new THREE.AnimationMixer(model);

animations.forEach((clip) => {
  this.mixer.clipAction(clip).play();
});
```

毎フレーム更新：

```javascript
tick: function (time, deltaTime) {
  if (this.mixer) {
    this.mixer.update(deltaTime / 1000);
  }
}
```

この構成により、異なる対象を制御する複数のAnimationClipを同時再生した。

同じTransformを複数Clipが制御する場合は競合・ブレンドに注意する。

### 7.3 マーカー方式

AR.jsのhiroマーカーを採用した。

```html
<a-marker preset="hiro">
```

GLBモデルはマーカーの子として配置する。

マーカーがカメラに認識されている間、3Dモデルを表示する。

マーカーが視野から外れた場合は、標準動作として表示が消える。

空間固定処理は最終版に含めていない。

### 7.4 最終HTML

最終的に使用したindex.htmlは、別途提示された以下の構成を採用した。

- A-Frame 1.5.0
- AR.js A-Frame build
- multi-animationコンポーネント
- hiroマーカー
- GLB: SantaKame_06_text.glb
- scale: 0.2 0.2 0.2
- position: 0 0 0
- rotation: 0 0 0

---

## 8. GitHub Pages

GitHubリポジトリ：

https://github.com/YUKI-SOKENDAI/AR_YK_Xmas

構成：

```text
AR_YK_Xmas/
└── docs/
    ├── index.html
    └── SantaKame_06_text.glb
```

GitHub Pagesで`main`ブランチの`/docs`を公開する。

HTTPS URLにアクセスし、カメラ権限を許可する。

hiroマーカーを映すと3Dアニメーションが表示される。

QRコードはWebARページへアクセスするためのものであり、hiroマーカーとは役割が異なる。

---

## 9. 試行したが採用しなかった方式

### 9.1 マーカーレス風表示

カメラ前に3Dモデルを配置する方式を試した。

問題：

- モデル内部にカメラが入ってしまう
- カメラ移動による自由な回り込みができない
- 空間に固定されたARにはならない

最終版では採用しなかった。

### 9.2 WebXR hit-test

WebXRによる平面検出・タップ配置も検討した。

AR開始ボタンが表示されない問題が発生し、完成には至っていない。

原因は確定していない。端末対応状況だけでなく、JavaScriptのモジュール読み込み方法も検証が必要。

### 9.3 マーカー認識後の空間固定

初回マーカー認識後にモデルを表示し続ける方式も検討した。

ただし、AR.js単体ではマーカーを見失った後の実空間トラッキングを継続できない。

最終版には含めなかった。

---

## 10. 今後の開発課題

1. GLBの軽量化
   - テクスチャ解像度の最適化
   - ポリゴン数削減
   - アニメーションデータ圧縮
2. WebARのロード時間短縮
3. 複数アニメーションの再生速度・開始タイミング制御
4. カスタム画像マーカーの導入
5. WebXR hit-testの再検証
6. iPhone / Androidでの動作比較
7. UI・音声・エフェクトの追加

---

## 11. Codexへの引き継ぎ事項

このプロジェクトでは、完成済みのhiroマーカー方式を動作基準とする。

既存のindex.htmlを変更する場合は、複数アニメーションの同時再生を維持すること。

マテリアルの透明設定を変更する場合は、過去に発生した背面透過問題の再発に注意すること。

WebXR版は完成版とは別に開発・検証すること。

未検証の機能を完成済みとして扱わないこと。

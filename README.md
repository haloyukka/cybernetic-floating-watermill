# Cybernetic Floating Watermill (空に浮かぶサイバネティック水車小屋)

Three.js と WebGL を用いて構築した、単一 HTML ファイル完結型の 3D ボクセルアート作品です。
ネオントワイライトの空（天頂の濃藍から地平線のネオンマゼンタへのグラデーション）に浮かぶ直径 80 ブロックの浮遊島と、ピンクの窓が灯る近未来サイバネティックな水車小屋を描画します。

## デモ（GitHub Pages）

- https://haloyukka.github.io/cybernetic-floating-watermill/

## 主な特徴

- **単一ファイル完結 (`index.html`)**: ビルド不要・外部アセット不要で、ブラウザで直接開くだけで動作します。
- **高効率ボクセルレンダリング**: 6近傍の表面カリング（Hidden Surface Removal）と `THREE.InstancedMesh` によるマテリアル別バッチ描画により、総ブロック数を **100,000 個以内**（約 25,000〜35,000 ブロック）に最適化しています。
- **ワールド構成**:
  - 浮遊島（直径 80 ブロック・逆円錐岩盤＋サイバー装甲＋下部反重力コア＆二重エネルギーリング）
  - 水車小屋（ダークメタル外壁＋ピンクに発光する窓＋アンテナビーコン）
  - 水車（直径 14 ブロック・約 8.0 秒で 1 周）
  - サイバーツリー（7 本配置・金属幹＋ネオン発光葉）
  - 滝パーティクル（300 個のボクセル粒子が重力に従って循環落下）
- **カメラワーク & 照明**:
  - 40.0 秒周期の自動オービット回転（マウス／タッチによる手動操作・HUD からの ON/OFF 切替対応）
  - カスタム GLSL シェーダーによるネオントワイライトの空と星空表現

## ディレクトリ構成

```text
cybernetic-floating-watermill/
├── README.md
├── index.html                  # アプリケーション本体（単一HTML）
└── docs/
    ├── 01_requirements.md      # 要件定義書
    ├── 02_basic_design.md      # 基本設計書
    ├── 03_detailed_design.md   # 詳細設計書
    └── 04_design_system.md     # デザインシステム
```

## ローカルでの実行方法

1. `index.html` を Chrome / Edge / Firefox / Safari などのモダンブラウザにドラッグ＆ドロップするか、ローカルサーバーで開きます。
2. マウスドラッグで視点回転、ホイールでズーム、右ドラッグでパン操作が可能です。
3. 左上の HUD パネルから「自動周回 (40s)」の ON/OFF 切替や視点リセットが行えます。

## 技術スタック

- **Rendering**: Three.js (WebGLRenderer / PCFSoftShadowMap / ACESFilmicToneMapping)
- **Optimization**: Voxel Surface Culling + `THREE.InstancedMesh`
- **Shader**: Custom GLSL Vertex / Fragment Shader (Twilight SkyDome)

# 詳細設計書：空に浮かぶサイバネティック水車小屋

## 1. ボクセルデータ構造とグリッド仕様

### 1.1 マテリアルID定義（`MAT` 定数）
ボクセルグリッドは 1 次元化した `Uint8Array` で管理し、メモリ効率と走査速度を最大化する。

| ID | 定数名 | 用途 | マテリアル特性 |
| :--- | :--- | :--- | :--- |
| `0` | `AIR` | 空気（描画なし） | 透過・カリング判定時の「露出」基準 |
| `1` | `ROCK_DARK` | 浮遊島・深部岩盤 | Roughness `0.85`, Metalness `0.2` |
| `2` | `ROCK_METAL` | 浮遊島・装甲外殻 | Roughness `0.45`, Metalness `0.7` |
| `3` | `GRASS_CYBER` | 地表・サイバー苔 | Roughness `0.7`, 微弱エミッシブ |
| `4` | `WATER` | 川・水路 | 半透明／光沢、Roughness `0.15` |
| `5` | `WALL_DARK` | 小屋・ダークメタル壁 | Roughness `0.4`, Metalness `0.75` |
| `6` | `ROOF_SLATE` | 小屋・屋根パネル | Roughness `0.5`, Metalness `0.6` |
| `7` | `WINDOW_YELLOW` | 小屋・発光窓 | Emissive `#FFD54F` (`intensity: 2.2`) |
| `8` | `NEON_CYAN` | サイバーライン・コア | Emissive `#00F0FF` (`intensity: 2.0`) |
| `9` | `NEON_AMBER` | 警告灯・動力ライン | Emissive `#FF9100` (`intensity: 1.8`) |
| `10` | `TREE_TRUNK` | 樹木・合金幹 | Roughness `0.5`, Metalness `0.65` |
| `11` | `TREE_LEAF` | 樹木・発光葉クラスタ | Emissive `#00E676` (`intensity: 0.8`) |

### 1.2 グリッド座標変換式
グリッドサイズを `GRID_X = 84`, `GRID_Y = 76`, `GRID_Z = 84`（オフセット `OX = 42, OY = 48, OZ = 42`）と定義する。

```javascript
// ワールド座標 (x, y, z) -> 1次元配列インデックス変換
function getIndex(x, y, z) {
  const gx = x + OX;
  const gy = y + OY;
  const gz = z + OZ;
  if (gx < 0 || gx >= GRID_X || gy < 0 || gy >= GRID_Y || gz < 0 || gz >= GRID_Z) {
    return -1;
  }
  return gx + GRID_X * (gz + GRID_Z * gy);
}
```

---

## 2. 表面カリングとインスタンス上限制御

### 2.1 6近傍露出判定アルゴリズム
全有効グリッド座標 `(x, y, z)` に対して以下を評価する。

```javascript
const DIRS = [
  [1, 0, 0], [-1, 0, 0],
  [0, 1, 0], [0, -1, 0],
  [0, 0, 1], [0, 0, -1]
];

function isExposed(x, y, z) {
  const selfMat = getVoxel(x, y, z);
  for (let i = 0; i < 6; i++) {
    const nx = x + DIRS[i][0];
    const ny = y + DIRS[i][1];
    const nz = z + DIRS[i][2];
    const neighborMat = getVoxel(nx, ny, nz);
    // 隣接が AIR(0) または半透明の水(4) であれば露出とみなす
    if (neighborMat === 0 || (neighborMat === 4 && selfMat !== 4)) {
      return true;
    }
  }
  return false;
}
```

### 2.2 総ブロック数上限（100,000 個）の保証
- 静的ボクセル抽出後、`totalBlocks = staticCount + wheelCount + waterfallCount(300) + ringCount` を算出する。
- 万一パラメータ変更等で `totalBlocks` が `100,000` に近づいた場合、上限ガード処理により余剰インスタンスをトリミングし、**厳密に `totalBlocks <= 100,000` を保証** する。

---

## 3. 主要モジュール生成詳細

### 3.1 浮遊島（直径 80 ブロック）生成ロジック
- 水平半径 `r = Math.sqrt(x * x + z * z)` を計算。
- 縁の自然な凹凸を出すため、角度 `theta = Math.atan2(z, x)` に対する低周波ノイズ `edgeNoise = Math.sin(theta * 5) * 1.2 + Math.cos(theta * 3) * 0.8` を加算し、`r <= 39.2 + edgeNoise`（最大外径 80 ブロック）を島領域とする。
- 下部深度 `depth(r)` は中心ほど深く、外周で浅くなるプロファイルを採用：
  $$\text{depth}(r) = 36 \times \left(1 - \left(\frac{r}{40}\right)^{1.65}\right) + \text{rockNoise}(x, z)$$
- 島外周の一部にはサイバー装甲プレート（`MAT.ROCK_METAL`）とシアン発光スリット（`MAT.NEON_CYAN`）を埋め込み、浮遊機関の人工物感を演出する。

### 3.2 水車（直径 14 ブロック）生成と回転仕様
- **構成**: 外輪（半径 `R = 7`、外径 `14` ブロック）、内輪（半径 `R = 4`）、中心ハブ（半径 `2`）、放射状スポーク（8本）、水受けブレード（12枚、幅 `3` ブロック）。
- **回転制御**: `THREE.Group`（`waterwheelGroup`）にまとめて配置し、毎フレーム経過時間 `dt`（秒）に応じて Z 軸回転を更新する。
  $$\omega_{\text{wheel}} = \frac{2\pi}{8.0} \approx 0.785398 \text{ [rad/s]}$$
  $$\text{waterwheelGroup.rotation.z} -= \omega_{\text{wheel}} \times \Delta t$$

### 3.3 滝パーティクル（300 個）物理仕様
- `THREE.InstancedMesh`（インスタンス数 `COUNT = 300`）で管理。
- 各粒子 `i` は状態ベクトル `{ x, y, z, vx, vy, vz, scale }` を保持する。
- **初期スポーン / リスポーン条件**: `y < -46.0` に到達した粒子を滝口（`x = 39.5 + rand(-0.5, 0.5)`, `y = -0.5`, `z = rand(-2.6, 2.6)`）へ再配置する。
- **運動更新式（オイラー積分）**:
  $$v_y \leftarrow v_y - g \cdot \Delta t \quad (g = 18.0 \text{ [block/s}^2\text{]})$$
  $$x \leftarrow x + v_x \cdot \Delta t, \quad y \leftarrow y + v_y \cdot \Delta t, \quad z \leftarrow z + v_z \cdot \Delta t$$
- 落下速度に応じて Y 方向のスケールを引き伸ばし（`scaleY = 0.5 + Math.abs(vy) * 0.04`）、水流の躍動感を表現する。

### 3.4 カメラ周回軌道（40 秒周期）仕様
- 注視点 `target = (0, -4, 0)`、基準半径 `R_cam = 96`、高さ `Y_cam = 28`。
- 自動周回フラグ `autoOrbit === true` の間、毎フレーム角度を更新する：
  $$\omega_{\text{cam}} = \frac{2\pi}{40.0} \approx 0.157080 \text{ [rad/s]}$$
  $$\theta_{\text{cam}} \leftarrow \theta_{\text{cam}} + \omega_{\text{cam}} \times \Delta t$$
- ユーザーがマウスドラッグ／タッチ操作を開始（`OrbitControls` の `start` イベント）した際は自動周回を一時停止し、HUD ボタンからいつでも再開可能とする。

---

## 4. 夕焼け空シェーダー（GLSL）仕様

```glsl
uniform vec3 uTopColor;      // #0B192C (天頂の濃い青)
uniform vec3 uMidColor;      // #2B3A67 (中間のトワイライトブルー)
uniform vec3 uHorizonColor;  // #FF6B35 (地平線のオレンジ)
uniform vec3 uBottomColor;   // #1A1128 (下部ナイトパープル)

varying vec3 vWorldPosition;

void main() {
  float h = normalize(vWorldPosition).y;
  vec3 col;
  if (h >= 0.0) {
    float t1 = smoothstep(0.0, 0.28, h);
    float t2 = smoothstep(0.22, 0.85, h);
    col = mix(uHorizonColor, uMidColor, t1);
    col = mix(col, uTopColor, t2);
  } else {
    float tb = smoothstep(0.0, -0.45, h);
    col = mix(uHorizonColor, uBottomColor, tb);
  }
  gl_FragColor = vec4(col, 1.0);
}
```

# BUG-003: カメラTOMLインポート時のZ軸まわり90°回転

| 項目 | 内容 |
|------|------|
| 文書番号 | DOC-004 |
| バグ番号 | BUG-003 |
| 発見日 | 2026-02-19 |
| 重要度 | 高 |
| ステータス | 原因特定済み |

## 概要

カメラキャリブレーションデータをTOMLファイルからインポートすると、カメラがZ軸（鉛直軸）まわりに90°回転した状態でBlenderに配置される。

## 原因

インポート処理（`setup_cams()`）において、不要なRz(-90°)回転が平行移動ベクトルとオイラー角の両方に適用されている。具体的には `t = [t[1], -t[0], t[2]]`（Rz(-90°)と同値）と `rotation_euler += [0, 0, -90°]` の2箇所。これにより `matrix_world` 全体にRz₄(-90°)が左から乗じられた状態になる。エクスポート側にも対応する逆変換 `rot_90 @`（Rz(+90°)）が存在する。

## データの流れ

### TOMLファイルのデータ形式

TOMLファイルのR（Rodrigues回転ベクトル）とTはOpenCVカメラ中心視点で格納されている:

```
Qc = R*Q + T   (ワールド点Q → カメラ点Qc)
```

ワールド座標系はPose2Simが使用するOpenSim/ISB規約: X=前方, Y=上方, Z=右方。

### インポート処理の追跡（setup_cams, 497-504行）

```
ステップ1: world_to_camera_persp(R, T)
  カメラ中心 → ワールド中心に変換
  r = R^T,  t = -R^T * T
  → r, t はOpenSim/ISB座標系でのカメラの姿勢・位置

ステップ2: t = [t[1], -t[0], t[2]]        ← 誤り
  これはZ軸まわり-90°回転: (x,y,z) → (y,-x,z)
  結果:
    Blender_X = OpenSim_Y (上方)   ← 正しくはOpenSim_Z (右方)
    Blender_Y = -OpenSim_X (-前方) ← 正しくはOpenSim_X (前方)
    Blender_Z = OpenSim_Z (右方)   ← 正しくはOpenSim_Y (上方)

ステップ3: matrix_worldを設定
  回転行列rはOpenSim/ISB座標系のまま変換されずに使用される ← 誤り

ステップ4: set_loc_rotation([180°,0,0])
  OpenCV→Blenderカメラ規約変換（+Zを見る → -Zを見る）
  この操作自体は正しい

ステップ5: rotation_euler += [0, 0, -90°]  ← 誤り
  Z軸まわり-90°を追加してステップ2と合わせて座標系変換を意図
  しかし回転行列rが未変換のまま残っているため不完全
```

### BUG-001との比較

| 項目 | BUG-001 (markers/motion/forces) | BUG-003 (cameras) |
|------|-------------------------------|-------------------|
| 誤った変換 | `(X,Y,Z)→(X,-Z,Y)` | `(X,Y,Z)→(Y,-X,Z)` |
| 正しい変換 | `(X,Y,Z)→(Z,X,Y)` | `(X,Y,Z)→(Z,X,Y)` |
| 手法 | H_zup行列 | 平行移動の成分入れ替え + Z回転 |
| 共通原因 | CG汎用Y-up変換の誤用 | CG汎用Y-up変換の誤用 |

### 数学的検証

**現行の平行移動変換:**

Z軸-90°回転行列を適用:
```
R_z(-90°) = [[ 0, 1, 0],
             [-1, 0, 0],
             [ 0, 0, 1]]

(x,y,z) → (y, -x, z)
```

**正しい平行移動変換:**

OpenSim/ISB → Blender 軸置換行列:
```
M = [[0, 0, 1],    # Blender_X = OpenSim_Z (右)
     [1, 0, 0],    # Blender_Y = OpenSim_X (前)
     [0, 1, 0]]    # Blender_Z = OpenSim_Y (上)

(x,y,z) → (z, x, y)
```

**差異: Z軸まわり+90°のずれ**（ユーザー報告と一致）

## 影響範囲

### 直接影響を受ける関数

| ファイル | 関数 | 行番号 | 影響 |
|---------|------|--------|------|
| `cameras.py` | `setup_cams()` | 488, 493 | インポート: 動的カメラ |
| `cameras.py` | `setup_cams()` | 499, 504 | インポート: 非動的カメラ |
| `cameras.py` | `retrieveCal_fromScene()` | 359, 367-368, 400-401 | エクスポート: 逆変換 |

### 呼び出しチェーン

```
__init__.py
├── importCal.execute() [line 97]
│   └── cameras.import_cameras() [line 525]
│       ├── cameras.retrieveCal_fromFile() [line 256]  # TOMLパース
│       └── cameras.setup_cams() [line 435]            ← BUG-003（インポート）
└── exportCal.execute() [line 122]
    └── cameras.export_cameras() [line 543]
        ├── cameras.retrieveCal_fromScene() [line 313]  ← BUG-003（エクスポート）
        └── cameras.write_calibration() [line 414]
```

### ラウンドトリップの整合性

インポートとエクスポートは互いの逆変換として設計されている:
- インポート: Z軸 -90° 回転 + XY入れ替え `[t[1], -t[0], t[2]]`
- エクスポート: Z軸 +90° 回転 `rot_90 @ t`

**ラウンドトリップ（エクスポート→インポート）は内部的に整合する。** しかし両方とも正しい軸置換ではないため、Pose2Simが出力したTOMLファイルを直接インポートすると90°ずれる。

## コード内の該当箇所

### インポート: setup_cams()（488-504行）

```python
# 動的カメラ（488, 493行）
t = np.array([t[1], -t[0], t[2]])          # ← 誤り: Z軸-90°回転
camera_obj.rotation_euler += np.radians([0, 0, -90])  # ← 誤り

# 非動的カメラ（499, 504行）
t = np.array([t[1], -t[0], t[2]])          # ← 誤り: Z軸-90°回転
camera_obj.rotation_euler += np.radians([0, 0, -90])  # ← 誤り
```

### エクスポート: retrieveCal_fromScene()（359, 367-368, 400-401行）

```python
rot_90 = mathutils.Euler(np.radians([0,0,90])).to_matrix()  # ← 誤り

# 非動的カメラ（367-368行）
t_new = rot_90 @ t      # ← 誤り
r_new = rot_90 @ r      # ← 誤り

# 動的カメラ（400-401行）
t_new = rot_90 @ t      # ← 誤り
r_new = rot_90 @ r      # ← 誤り
```

## 調査チェックリスト

- [x] TOMLファイル内のカメラパラメータの座標系を確認（OpenCV camera-centered, ワールドはOpenSim/ISB）
- [x] `setup_cams()` のZ軸-90°回転の意図を調査（CG系Y-up変換の誤用）
- [x] `t = np.array([t[1], -t[0], t[2]])` の意図を調査（Z軸-90°回転と同値）
- [x] エクスポート→インポートのラウンドトリップ整合性を確認（内部整合するが外部データとは不整合）
- [x] 正しい変換の特定（Rz(-90°)の除去のみ。軸置換Mは不要）
- [x] 機能設計書の作成
- [x] 修正実施（第1版: M適用 → 第2版: M除去に修正）
- [ ] テスト

---

# 第2部: 機能設計書

## 2.1 設計方針

### 2.1.1 目的

カメラキャリブレーションデータのインポート・エクスポートにおいて、不要なRz(-90°)回転を除去し、Pose2Simが出力したTOMLファイルを正しくインポートできるようにする。

### 2.1.2 基本方針

1. **Rz(-90°)の除去**: インポート時の `t = [t[1], -t[0], t[2]]`（平行移動へのRz(-90°)）と `rotation_euler += [0,0,-90°]` を削除する
2. **エクスポートの対応する逆変換も除去**: `rot_90 @` による+90° Z回転を削除する
3. **軸置換Mは適用しない**: BUG-001（markers/motion/forces）とは異なり、カメラには軸置換行列Mを適用しない（実験で検証済み）
4. **カメラ規約変換は保持**: OpenCV→Blenderカメラ規約変換（180° X軸フリップ `set_loc_rotation`）は正しいため変更しない

### 2.1.3 技術的根拠

**バグの本質:**

バグありコードの最終的な `matrix_world` を追跡すると:

```
1. r, t = world_to_camera_persp(R, T)         → OpenSim/ISB座標系
2. t ← Rz(-90°) @ t                           → [t[1], -t[0], t[2]]
3. matrix_world = [[r, Rz(-90°)@t], [0, 1]]   → rは未変換
4. rotation ← r @ Rx(180°)                     → カメラ規約変換
5. rotation ← Rz(-90°) @ r @ Rx(180°)         → euler Z成分に-90°加算

最終結果: matrix_world = Rz₄(-90°) @ [[r·Rx(180°), t], [0, 1]]
```

バグありコード全体として、正しい変換に対してRz₄(-90°)が左から乗じられている。

**実験による検証:**

バグありコードでインポートしたカメラに対して以下の補正スクリプトを適用:
```python
correction = mathutils.Matrix.Rotation(radians(+90), 4, 'Z')
obj.matrix_world = correction @ obj.matrix_world
```

結果: `Rz₄(+90°) @ Rz₄(-90°) @ H = H` → カメラが既知の正しい位置に移動した。

**正しいmatrix_world（補正後）:**
```
[[r @ Rx(180°), t], [0, 1]]
```

ここで r, t は `world_to_camera_persp()` の出力（OpenSim/ISB座標系）をそのまま使用。

**BUG-001との違い — なぜ軸置換Mを適用しないか:**

M（軸置換行列）を適用した場合、カメラが正しい位置に配置されず、上方向が水平方向に対して90°回転する（横向き）ことが実験で確認された。これは M @ (0,0,1) = (1,0,0) となり、カメラの上方向（Blender Z）が右方向（Blender X）に変換されるためである。

カメラの matrix_world はマーカー/モーション/力のような純粋な座標データとは異なり、カメラの外部パラメータ（位置・姿勢）をBlenderのカメラオブジェクトとして表現するものであり、別の変換規則が適用される。

**ラウンドトリップ検証（インポート→エクスポート）:**

```
インポート:
  1. (R_cv, T_cv) → world_to_camera_persp → (r, t)
  2. matrix_world = [[r, t], [0, 1]]
  3. set_loc_rotation([180°,0,0])  → rotation = r @ Rx(180°)

エクスポート:
  1. r = rotation.to_matrix() @ Rx(180°) = r @ Rx(180°) @ Rx(180°) = r  [Rx(180°)² = I]
  2. t = location
  3. world_to_camera_persp(r, t) → (R_cv, T_cv)

結果: R_cv, T_cv が完全に復元される ✓
```

## 2.2 修正対象の詳細設計

### 2.2.1 インポート: setup_cams() — 動的カメラ

**ファイル:** `Pose2Sim_Blender/cameras.py`
**関数:** `setup_cams()`

**現行コード:**
```python
r, t = world_to_camera_persp(R[c][n], T[c][n])
t = np.array([t[1], -t[0], t[2]])          # ← 削除: Rz(-90°)
homog_matrix = np.block([[r,t.reshape(3,1)],
                        [np.zeros(3), 1 ]])
camera_obj.matrix_world = mathutils.Matrix(homog_matrix)
set_loc_rotation(camera_obj, np.radians([180,0,0]))
camera_obj.rotation_euler += np.radians([0, 0, -90])  # ← 削除
```

**修正後コード:**
```python
r, t = world_to_camera_persp(R[c][n], T[c][n])
homog_matrix = np.block([[r,t.reshape(3,1)],
                        [np.zeros(3), 1 ]])
camera_obj.matrix_world = mathutils.Matrix(homog_matrix)
set_loc_rotation(camera_obj, np.radians([180,0,0]))
```

**変更点:**
1. `t = np.array([t[1], -t[0], t[2]])` を削除
2. `camera_obj.rotation_euler += np.radians([0, 0, -90])` を削除

### 2.2.2 インポート: setup_cams() — 非動的カメラ

変更点は 2.2.1 と同一。

### 2.2.3 エクスポート: retrieveCal_fromScene() — 非動的カメラ

**現行コード:**
```python
rot_90 = mathutils.Euler(np.radians([0,0,90])).to_matrix()  # ← 削除

r = camera_obj.rotation_euler.to_matrix() @ rot_180
t = camera_obj.location
t_new = rot_90 @ t      # ← 削除
r_new = rot_90 @ r      # ← 削除
r_loc, t_loc = world_to_camera_persp(np.array(r_new), t_new)
```

**修正後コード:**
```python
r = camera_obj.rotation_euler.to_matrix() @ rot_180
t = camera_obj.location
r_loc, t_loc = world_to_camera_persp(np.array(r), t)
```

**変更点:**
1. `rot_90` の定義を削除
2. `t_new = rot_90 @ t` / `r_new = rot_90 @ r` を削除し、r, t を直接使用

### 2.2.4 エクスポート: retrieveCal_fromScene() — 動的カメラ

変更点は 2.2.3 と同一（ループ内の各フレームに適用）。

## 2.3 変更一覧

| # | 関数 | 変更種別 | 内容 |
|---|------|----------|------|
| 1 | setup_cams（動的） | 削除 | `t = np.array([t[1], -t[0], t[2]])` |
| 2 | setup_cams（動的） | 削除 | `rotation_euler += [0, 0, -90°]` |
| 3 | setup_cams（非動的） | 削除 | `t = np.array([t[1], -t[0], t[2]])` |
| 4 | setup_cams（非動的） | 削除 | `rotation_euler += [0, 0, -90°]` |
| 5 | retrieveCal_fromScene | 削除 | `rot_90` の定義 |
| 6 | retrieveCal_fromScene（非動的） | 削除 | `rot_90 @` の適用、r/tを直接使用 |
| 7 | retrieveCal_fromScene（動的） | 削除 | `rot_90 @` の適用、r/tを直接使用 |

## 2.4 テスト計画

### 2.4.1 テストケース

| ID | テスト内容 | 期待結果 |
|----|-----------|---------|
| T01 | TOMLファイルからカメラをインポート | カメラが既知の正しい位置・方向に配置される |
| T02 | 動的カメラのインポート | 各フレームでカメラが正しい方向に配置される |
| T03 | インポート後エクスポートし、元のTOMLと比較 | R, T の値が元のTOMLファイルと一致する（ラウンドトリップ） |
| T04 | エクスポートしたTOMLを再インポート | 元のBlenderシーンと同一のカメラ配置になる |

### 2.4.2 検証基準

- T01: カメラの位置・方向が既知の正しい位置と一致すること
- T01: カメラの上方向が正しいこと（横向きにならない）
- T03: エクスポートされたR, Tが元のTOMLの値と数値的に一致すること（浮動小数点誤差以内）

## 2.5 リスク評価

| リスク | 影響度 | 発生確率 | 対策 |
|-------|--------|---------|------|
| 既存ユーザーがBlender内でエクスポートしたTOMLを再利用している場合、修正後は互換性が失われる | 中 | 低 | 修正後のコードで再エクスポートすれば正しいデータになる |

## 2.6 修正チェックリスト

- [x] cameras.py インポート修正: 動的カメラ
- [x] cameras.py インポート修正: 非動的カメラ
- [x] cameras.py エクスポート修正: 非動的カメラ
- [x] cameras.py エクスポート修正: 動的カメラ
- [ ] テスト: カメラインポートの位置・方向確認（T01）
- [ ] テスト: ラウンドトリップ（T03）
- [ ] コミット

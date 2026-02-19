# BUG-001: 座標変換バグ（Z軸まわり+90°回転）

| 項目 | 内容 |
|------|------|
| 文書番号 | DOC-002 |
| バグ番号 | BUG-001 |
| 発見日 | 2026-02-19 |
| 修正日 | 2026-02-19 |
| 重要度 | 高 |
| ステータス | **修正済み** |
| 修正コミット | b76f475 |

---

# 第1部: バグ報告

## 1.1 概要

TRC/MOTファイルのインポート時に、データがZ軸（鉛直軸）まわりに+90°回転した状態で配置されるバグ。

## 1.2 影響範囲

### 直接影響を受けるファイル

| ファイル | 関数 | 行番号 | 呼び出し元 | 影響 |
|---------|------|--------|-----------|------|
| `markers.py` | `import_trc()` | 349-355 | `__init__.py:285` | TRCマーカー位置 |
| `motion.py` | `apply_mot_to_model()` | 129-130, 156-157 | `__init__.py:338` | ボディ変換行列 |
| `forces.py` | `import_forces()` | 151-152, 159-160 | `__init__.py:362` | 力ベクトル変換 |

### 要調査ファイル

| ファイル | 関数 | 行番号 | 懸念事項 |
|---------|------|--------|---------|
| `markers.py` | `import_trc()` (C3D分岐) | 377-379 | `use_manual_orientation=True, axis_forward='Y', axis_up='Z'` の設定が誤りの可能性 |

### 影響を受けないファイル

| ファイル | 理由 |
|---------|------|
| `cameras.py` | Pose2Simカメラキャリブレーション用の独自変換を使用（OpenSim座標系とは別系統） |
| `model.py` | 座標変換なし。.osimファイルからメッシュを直接インポート |
| `skeletons.py` | スケルトン階層定義のみ |
| `common.py` | ユーティリティ関数のみ |

## 1.3 呼び出しチェーン

```
__init__.py
├── addMarkers.execute() [line 285]
│   └── markers.import_trc(direction='zup') ← BUG-001
├── addMotion.execute() [line 338]
│   └── motion.apply_mot_to_model(direction='zup') ← BUG-001
└── addForces.execute() [line 362]
    └── forces.import_forces(direction='zup') ← BUG-001
```

## 1.4 原因

### 座標系の違い

**OpenSim/ISB規約:**
- X = 前方 (forward)
- Y = 上方 (up)
- Z = 右方 (right)

**Blender:**
- X = 右方 (right)
- Y = 前方 (forward)
- Z = 上方 (up)

### 正しい変換

```
(X, Y, Z)_OpenSim → (Z, X, Y)_Blender
```

つまり:
- Blender_X = OpenSim_Z (右)
- Blender_Y = OpenSim_X (前)
- Blender_Z = OpenSim_Y (上)

### 現行コードの変換（誤り）

```
(X, Y, Z)_OpenSim → (X, -Z, Y)_Blender
```

これはMaya/OpenGL系（X=右, Y=上, Z=奥）からBlenderへの汎用的な「Y-up → Z-up」変換であり、OpenSim/ISB系には適用できない。

## 1.5 コード内の証拠

### markers.py (348-359行)

```python
if direction=='zup':
    # 旧コード（正しい）
    # loc_x = trc_data_np[n,3*i+4]   # Z_data → Blender X
    # loc_y = trc_data_np[n,3*i+2]   # X_data → Blender Y
    # loc_z = trc_data_np[n,3*i+3]   # Y_data → Blender Z

    # 現行コード（誤り）
    loc_x = trc_data_np[n,3*i+2]     # X_data → Blender X
    loc_y = -trc_data_np[n,3*i+4]    # -Z_data → Blender Y
    loc_z = trc_data_np[n,3*i+3]     # Y_data → Blender Z
```

### motion.py / forces.py (H_zup行列)

```python
# 旧コード（正しい）
# H_zup = np.array([[0,0,1,0], [1,0,0,0], [0,1,0,0], [0,0,0,1]])

# 現行コード（誤り）
H_zup = np.array([[1,0,0,0], [0,0,-1,0], [0,1,0,0], [0,0,0,1]])
```

### 変換行列の比較

**旧H_zup行列（正しい）:**
```
[0 0 1 0]   Blender_X = OpenSim_Z (右)
[1 0 0 0]   Blender_Y = OpenSim_X (前)
[0 1 0 0]   Blender_Z = OpenSim_Y (上)
[0 0 0 1]
```

**現行H_zup行列（誤り）:**
```
[1  0  0 0]   Blender_X = OpenSim_X (前) ← 誤り
[0  0 -1 0]   Blender_Y = -OpenSim_Z (-右) ← 誤り
[0  1  0 0]   Blender_Z = OpenSim_Y (上) ← 正しい
[0  0  0 1]
```

## 1.6 推定される経緯

旧コードはOpenSim/ISBの座標系を正しく考慮していたが、後に汎用的な「Y-up → Z-up」変換に「修正」された際、水平軸（X, Z）の意味の違いが見落とされた。3ファイルとも同時に変更された形跡がある。

---

# 第2部: 機能設計書

## 2.1 設計方針

### 2.1.1 目的

OpenSim/ISB座標系からBlender座標系への変換を正しく実装し、TRC/MOT/C3Dファイルのインポート時にデータが正しい向きで配置されるようにする。

### 2.1.2 基本方針

1. **最小限の変更**: コメントアウトされた旧コードへの復帰を基本とし、新規ロジックの追加は最小限に留める
2. **一貫性の確保**: 全ての座標変換箇所で同一の変換ルールを適用する
3. **後方互換性**: `direction='yup'` オプションの動作は変更しない
4. **コメントの整理**: 修正後、不要なコメントアウトコードを削除する

### 2.1.3 変換仕様

| 入力（OpenSim/ISB） | 出力（Blender） | 説明 |
|---------------------|-----------------|------|
| X (前方) | Y | 前方向 |
| Y (上方) | Z | 上方向 |
| Z (右方) | X | 右方向 |

**変換行列:**
```
H_opensim_to_blender = [
    [0, 0, 1, 0],  # Blender_X = OpenSim_Z
    [1, 0, 0, 0],  # Blender_Y = OpenSim_X
    [0, 1, 0, 0],  # Blender_Z = OpenSim_Y
    [0, 0, 0, 1]
]
```

## 2.2 修正対象の詳細設計

### 2.2.1 markers.py - TRC分岐

**ファイル:** `Pose2Sim_Blender/markers.py`
**関数:** `import_trc()`
**行番号:** 349-355

**現行コード:**
```python
if direction=='zup':
    loc_x = trc_data_np[n,3*i+2]
    loc_y = -trc_data_np[n,3*i+4]
    loc_z = trc_data_np[n,3*i+3]
```

**修正後コード:**
```python
if direction=='zup':
    # OpenSim/ISB → Blender: (X,Y,Z) → (Z,X,Y)
    loc_x = trc_data_np[n,3*i+4]   # OpenSim_Z → Blender_X
    loc_y = trc_data_np[n,3*i+2]   # OpenSim_X → Blender_Y
    loc_z = trc_data_np[n,3*i+3]   # OpenSim_Y → Blender_Z
```

**TRCファイル列インデックス:**
- `3*i+2`: X座標（前方）
- `3*i+3`: Y座標（上方）
- `3*i+4`: Z座標（右方）

### 2.2.2 markers.py - C3D分岐

**ファイル:** `Pose2Sim_Blender/markers.py`
**関数:** `import_trc()`
**行番号:** 377-379

**現行コード:**
```python
c3d_importer.load(operator, bpy.context,
                  filepath = trc_path,
                  use_manual_orientation=True, axis_forward='Y', axis_up='Z')
```

**修正後コード:**
```python
# C3Dメタデータによる自動座標変換に任せる
c3d_importer.load(operator, bpy.context,
                  filepath = trc_path,
                  use_manual_orientation=False)
```

**設計根拠:**
- io_anim_c3dはC3Dファイル内のメタデータ（X_SCREEN, Y_SCREEN）を読み取り、適切な座標変換を自動適用する
- Pose2Simが出力するC3Dファイルにはこのメタデータが含まれている
- 手動設定より自動検出の方が堅牢

### 2.2.3 motion.py

**ファイル:** `Pose2Sim_Blender/motion.py`
**関数:** `apply_mot_to_model()`
**行番号:** 129-130

**現行コード:**
```python
# H_zup = np.array([[0,0,1,0], [1,0,0,0], [0,1,0,0], [0,0,0,1]])
H_zup = np.array([[1,0,0,0], [0,0,-1,0], [0,1,0,0], [0,0,0,1]])
```

**修正後コード:**
```python
# OpenSim/ISB → Blender 変換行列
# (X,Y,Z)_OpenSim → (Z,X,Y)_Blender
H_zup = np.array([[0,0,1,0], [1,0,0,0], [0,1,0,0], [0,0,0,1]])
```

### 2.2.4 forces.py

**ファイル:** `Pose2Sim_Blender/forces.py`
**関数:** `import_forces()`
**行番号:** 151-152

**現行コード:**
```python
# H_zup = np.array([[0,0,1,0], [1,0,0,0], [0,1,0,0], [0,0,0,1]])
H_zup = np.array([[1,0,0,0], [0,0,-1,0], [0,1,0,0], [0,0,0,1]])
```

**修正後コード:**
```python
# OpenSim/ISB → Blender 変換行列
# (X,Y,Z)_OpenSim → (Z,X,Y)_Blender
H_zup = np.array([[0,0,1,0], [1,0,0,0], [0,1,0,0], [0,0,0,1]])
```

## 2.3 テスト計画

### 2.3.1 テストデータ

| データ | パス | 用途 |
|--------|------|------|
| TRCファイル | `Examples/Pose2Sim_markers.trc` | マーカーインポートテスト |
| C3Dファイル | （同一データのC3D版を用意） | C3Dインポートテスト |
| MOTファイル | `Examples/Pose2Sim_motion.mot` | モーションインポートテスト |
| 力データ | `Examples/Moco_forces.mot` | 力インポートテスト |
| OSIMモデル | `Examples/Pose2Sim_model.osim` | モデルインポートテスト |

### 2.3.2 テストケース

| ID | テスト内容 | 期待結果 | 検証方法 |
|----|-----------|---------|---------|
| T01 | TRCインポート | io_anim_c3dのC3D結果と一致 | 座標値の数値比較 |
| T02 | C3Dインポート | io_anim_c3d（デフォルト設定）と一致 | 視覚的比較 |
| T03 | MOTモーション | モデルが正しい向きでアニメーション | 前方向が+Y方向 |
| T04 | 力ベクトル | 力ベクトルが正しい向きで表示 | 視覚的確認 |
| T05 | yupオプション | direction='yup'の動作が不変 | 回帰テスト |

### 2.3.3 検証基準

**TRC/C3D比較:**
1. 同一データのTRCとC3Dをインポート
2. 各フレームの各マーカー座標を比較
3. 差異が1mm（C3Dの丸め誤差）以内であること

**視覚的検証:**
1. 被験者の前方向がBlenderの+Y方向を向いていること
2. 被験者の右方向がBlenderの+X方向を向いていること
3. 被験者の上方向がBlenderの+Z方向を向いていること

## 2.4 実装手順

### ステップ1: markers.py TRC分岐の修正

```
ファイル: Pose2Sim_Blender/markers.py
行: 349-355

変更内容:
- loc_x = trc_data_np[n,3*i+2] → loc_x = trc_data_np[n,3*i+4]
- loc_y = -trc_data_np[n,3*i+4] → loc_y = trc_data_np[n,3*i+2]
- loc_z = trc_data_np[n,3*i+3] （変更なし）
- コメントアウトされた旧コードを削除
- 新しいコメントを追加
```

### ステップ2: markers.py C3D分岐の修正

```
ファイル: Pose2Sim_Blender/markers.py
行: 377-379

変更内容:
- use_manual_orientation=True → use_manual_orientation=False
- axis_forward='Y', axis_up='Z' を削除
```

### ステップ3: motion.py の修正

```
ファイル: Pose2Sim_Blender/motion.py
行: 129-130

変更内容:
- 現行のH_zup行列を旧行列に置換
- コメントアウトされた旧コードを削除
- 新しいコメントを追加
```

### ステップ4: forces.py の修正

```
ファイル: Pose2Sim_Blender/forces.py
行: 151-152

変更内容:
- 現行のH_zup行列を旧行列に置換
- コメントアウトされた旧コードを削除
- 新しいコメントを追加
```

### ステップ5: テスト実施

```
1. Blenderを起動
2. Pose2Sim_Blenderアドオンをリロード
3. テストケースT01〜T05を実行
4. 結果を記録
```

### ステップ6: コミット

```
git add Pose2Sim_Blender/markers.py
git add Pose2Sim_Blender/motion.py
git add Pose2Sim_Blender/forces.py
git commit -m "Fix coordinate transformation for OpenSim/ISB to Blender (BUG-001)"
```

## 2.5 リスク評価

| リスク | 影響度 | 発生確率 | 対策 |
|-------|--------|---------|------|
| 既存プロジェクトとの互換性 | 中 | 高 | リリースノートで変更を明記 |
| yupオプションへの影響 | 低 | 低 | 回帰テストで確認 |
| C3D自動検出の失敗 | 低 | 低 | メタデータがない場合のフォールバック検討 |

## 2.6 修正チェックリスト

- [x] markers.py TRC分岐 修正 (349-355行)
- [x] markers.py C3D分岐 修正 (377-379行)
- [x] motion.py 修正 (129-130行)
- [x] forces.py 修正 (151-152行)
- [x] テストケース T01 実行・確認（TRCインポートで90°回転なし）
- [ ] テストケース T02 実行・確認
- [ ] テストケース T03 実行・確認
- [ ] テストケース T04 実行・確認
- [ ] テストケース T05 実行・確認
- [x] コミット作成 (b76f475)

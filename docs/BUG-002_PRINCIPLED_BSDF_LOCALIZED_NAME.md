# BUG-002: Principled BSDFノード名のローカライズによるKeyError

| 項目 | 内容 |
|------|------|
| 文書番号 | DOC-003 |
| バグ番号 | BUG-002 |
| 発見日 | 2026-02-19 |
| 修正日 | 2026-02-19 |
| 重要度 | 中 |
| ステータス | **修正済み** |
| 修正コミット | e320c23 |

---

# 第1部: バグ報告

## 1.1 概要

BlenderのUIが英語以外（日本語等）に設定されている場合、マテリアル作成時に `KeyError: 'bpy_prop_collection[key]: key "Principled BSDF" not found'` が発生し、マーカー・力・カメラのインポートが失敗する。

## 1.2 発生条件

- Blender 3.4以降
- UIの言語設定が英語以外（日本語、中国語等）
- Preferences > Interface > Translation > "New Data" が有効

## 1.3 エラーメッセージ

```
Python: Traceback (most recent call last):
  File ".../__init__.py", line 285, in execute
    markers.import_trc(...)
  File ".../markers.py", line 340, in import_trc
    matg = createMaterial(color=COLOR, metallic = 0.5, roughness = 0.5)
  File ".../common.py", line 55, in createMaterial
    bsdf = nodes["Principled BSDF"]
KeyError: 'bpy_prop_collection[key]: key "Principled BSDF" not found'
```

## 1.4 原因

Blender 3.4以降、`material.use_nodes = True` で作成されるデフォルトノードの名前がUIの言語設定に従ってローカライズされる。日本語環境では `"Principled BSDF"` ではなく `"プリンシプルBSDF"` 等になるため、英語名でのキー検索が失敗する。

Blenderの既知バグ: [#104145 - Invalid translation on keys in python API](https://projects.blender.org/blender/blender/issues/104145)

## 1.5 影響範囲

| ファイル | 関数 | 行番号 | 影響を受ける機能 |
|---------|------|--------|----------------|
| `common.py` | `createMaterial()` | 55 | マーカー・モデルのマテリアル作成 |
| `forces.py` | `addForce()` | 94 | 力ベクトルのマテリアル作成 |
| `cameras.py` | `add_bezier()` | 207 | レイ（線）のマテリアル作成 |

### 呼び出しチェーン

```
__init__.py
├── addMarkers.execute()
│   └── markers.import_trc()
│       └── common.createMaterial()        ← BUG-002 (common.py:55)
├── addModel.execute()
│   └── model.import_model()
│       └── common.createMaterial()        ← BUG-002 (common.py:55)
├── addForces.execute()
│   └── forces.import_forces()
│       └── forces.addForce()              ← BUG-002 (forces.py:94)
└── raysFrom3Dpoint.execute()
    └── cameras.reproject_3D_points()
        └── cameras.add_bezier()           ← BUG-002 (cameras.py:207)
```

## 1.6 コード内の該当箇所

3ファイルに同一パターンのコードが存在する。

### common.py (50-56行)

```python
matg = bpy.data.materials.new(str(color))
matg.use_nodes = True
tree = matg.node_tree
nodes = tree.nodes
bsdf = nodes["Principled BSDF"]           # ← KeyError
bsdf.inputs["Base Color"].default_value = color
```

### forces.py (90-96行)

```python
matg = bpy.data.materials.new("Green")
matg.use_nodes = True
tree = matg.node_tree
nodes = tree.nodes
bsdf = nodes["Principled BSDF"]           # ← KeyError
bsdf.inputs["Base Color"].default_value = color
matg.diffuse_color = color
```

### cameras.py (203-209行)

```python
matg = bpy.data.materials.new("Orange")
matg.use_nodes = True
tree = matg.node_tree
nodes = tree.nodes
bsdf = nodes["Principled BSDF"]           # ← KeyError
bsdf.inputs["Base Color"].default_value = color
matg.diffuse_color = color
```

## 1.7 参考

- [Blender #104145 - Invalid translation on keys in python API](https://projects.blender.org/blender/blender/issues/104145)
- [Blender Artists - key Principled BSDF not found](https://blenderartists.org/t/blender-python-key-principled-bsdf-not-found/1486800)

---

# 第2部: 機能設計書

## 2.1 設計方針

### 2.1.1 目的

Principled BSDFノードの取得をUI言語に依存しない方法に変更し、全言語環境でマテリアル作成が正常に動作するようにする。

### 2.1.2 基本方針

1. **ノードタイプによる検索**: ノード名（ローカライズされる）ではなくノードタイプ `'BSDF_PRINCIPLED'`（言語非依存）で検索する
2. **最小限の変更**: ノード取得の1行のみを修正し、周囲のロジックは変更しない
3. **全箇所の統一**: 3ファイルの同一パターンを全て同じ方法で修正する

### 2.1.3 技術的根拠

Blenderの `bpy.types.ShaderNode` には `type` 属性があり、Principled BSDFの場合は常に `'BSDF_PRINCIPLED'` である。この値はUI言語やBlenderバージョンに依存しない。

## 2.2 修正対象の詳細設計

### 2.2.1 common.py

**ファイル:** `Pose2Sim_Blender/common.py`
**関数:** `createMaterial()`
**行番号:** 55

**現行コード:**
```python
bsdf = nodes["Principled BSDF"]
```

**修正後コード:**
```python
bsdf = next(n for n in nodes if n.type == 'BSDF_PRINCIPLED')
```

### 2.2.2 forces.py

**ファイル:** `Pose2Sim_Blender/forces.py`
**関数:** `addForce()`
**行番号:** 94

**現行コード:**
```python
bsdf = nodes["Principled BSDF"]
```

**修正後コード:**
```python
bsdf = next(n for n in nodes if n.type == 'BSDF_PRINCIPLED')
```

### 2.2.3 cameras.py

**ファイル:** `Pose2Sim_Blender/cameras.py`
**関数:** `add_bezier()`
**行番号:** 207

**現行コード:**
```python
bsdf = nodes["Principled BSDF"]
```

**修正後コード:**
```python
bsdf = next(n for n in nodes if n.type == 'BSDF_PRINCIPLED')
```

## 2.3 テスト計画

### 2.3.1 テストケース

| ID | テスト内容 | 環境 | 期待結果 |
|----|-----------|------|---------|
| T01 | TRCマーカーインポート | 日本語UI | KeyErrorなしでマーカー表示 |
| T02 | OSIMモデルインポート | 日本語UI | KeyErrorなしでモデル表示 |
| T03 | 力データインポート | 日本語UI | KeyErrorなしで力ベクトル表示 |
| T04 | レイ投影 | 日本語UI | KeyErrorなしでレイ表示 |
| T05 | TRCマーカーインポート | 英語UI | 回帰テスト: 動作不変 |
| T06 | OSIMモデルインポート | 英語UI | 回帰テスト: 動作不変 |

### 2.3.2 検証基準

- 日本語環境: `KeyError` が発生しないこと
- 英語環境: 修正前と同一の動作であること
- マテリアルの色・メタリック・ラフネスが正しく設定されること

## 2.4 実装手順

### ステップ1: common.py の修正

```
ファイル: Pose2Sim_Blender/common.py
行: 55

変更内容:
- nodes["Principled BSDF"] → next(n for n in nodes if n.type == 'BSDF_PRINCIPLED')
```

### ステップ2: forces.py の修正

```
ファイル: Pose2Sim_Blender/forces.py
行: 94

変更内容:
- nodes["Principled BSDF"] → next(n for n in nodes if n.type == 'BSDF_PRINCIPLED')
```

### ステップ3: cameras.py の修正

```
ファイル: Pose2Sim_Blender/cameras.py
行: 207

変更内容:
- nodes["Principled BSDF"] → next(n for n in nodes if n.type == 'BSDF_PRINCIPLED')
```

### ステップ4: テスト実施

```
1. Blenderを日本語UIで起動
2. テストケースT01〜T04を実行
3. Blenderを英語UIに切り替え
4. テストケースT05〜T06を実行
5. 結果を記録
```

### ステップ5: コミット

```
git add Pose2Sim_Blender/common.py
git add Pose2Sim_Blender/forces.py
git add Pose2Sim_Blender/cameras.py
git commit -m "Fix Principled BSDF node lookup for non-English Blender UI (BUG-002)"
```

## 2.5 リスク評価

| リスク | 影響度 | 発生確率 | 対策 |
|-------|--------|---------|------|
| `BSDF_PRINCIPLED` ノードが存在しない場合の `StopIteration` | 低 | 極低 | `use_nodes=True` 直後なのでデフォルトノードは必ず存在する |
| 将来のBlenderでタイプ名が変更される | 低 | 極低 | `BSDF_PRINCIPLED` はBlender内部定数であり変更の可能性は低い |

## 2.6 修正チェックリスト

- [x] common.py 修正 (55行)
- [x] forces.py 修正 (94行)
- [x] cameras.py 修正 (207行)
- [x] 日本語環境でテスト（KeyErrorの解消を確認）
- [ ] 英語環境で回帰テスト (T05〜T06)
- [x] コミット作成 (e320c23)

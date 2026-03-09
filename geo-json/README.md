# 日本地図 GeoJSON 変換プロジェクト

## 概要

国土地理院の行政区域データ（市区町村ポリゴン）を、D3.jsによる都道府県別人口地図に適した軽量GeoJSONへ変換したプロジェクトの記録。

- **元データ**: 国土地理院「行政区域データ」2025年1月1日版（`N03-20250101_XX.geojson`）
- **出力データ**: 都道府県別 `pref_XX.geojson`（47ファイル）
- **用途**: GitHub Pages公開の人口可視化地図（D3.js）

---

## 変換処理の内容

### やったこと

1. **Dissolve（統合）**: 市区町村ポリゴンを行政区域コード（`N03_007`）単位で統合
2. **Simplify（簡略化）**: RDPアルゴリズムでポリゴンの座標点数を削減
3. **プロパティ整理**: 元データの `N03_XXX` フィールドを以下3項目に絞り込み

```json
{ "code": "01101", "pref": "北海道", "city": "札幌市" }
```

| フィールド | 元フィールド | 内容 |
|---|---|---|
| `code` | `N03_007` | 行政区域コード（5桁） |
| `pref` | `N03_001` | 都道府県名 |
| `city` | `N03_004` | 市区町村名 |

---

## 処理方法（2種類）

### 方法A: Claude dissolve.py（44県で使用）

Claudeが処理したPythonスクリプト。外部ライブラリ不要。

```bash
python3 dissolve.py N03-20250101_XX.geojson pref_XX.geojson 0.0001
```

| パラメータ | 値 | 内容 |
|---|---|---|
| epsilon | `0.0001` | RDP簡略化の閾値（約10m精度） |
| keep-shapes相当 | スクリプト内で対応 | 3点未満になったリングは元を保持（離島対策） |

### 方法B: mapshaper オンライン（3県で使用）

元データが大容量（30MB超）でClaude/ChatGPTにアップロードできない場合に使用。
https://mapshaper.org/ のコンソール画面で以下を順番に実行。

```
-dissolve N03_007 copy-fields=N03_001,N03_004
-simplify dp 2% keep-shapes
-rename-fields code=N03_007,pref=N03_001,city=N03_004
```

その後 Export → GeoJSON → `pref_XX.geojson` で保存。

> **注意**: `$` のみ表示されるコマンドは正常（エラーではない）。

---

## 処理方法の使い分け基準

| 元ファイルサイズ | 対応方法 |
|---|---|
| 〜31MB | Claude（claude.ai）に直接アップロードして処理 |
| 31MB超 | mapshaperオンラインで処理（上記コマンドB参照） |

---

## 処理結果サマリー

処理した47都道府県の詳細は `geojson_summary.xlsx` を参照。

| 処理方法 | 件数 |
|---|---|
| Claude dissolve.py | 44県 |
| mapshaper 2% | 3県（北海道・岩手県・長崎県） |

---

## 出力ファイル構成

```
geo-json/
  pref_01.geojson   ← 北海道
  pref_02.geojson   ← 青森県
  ...
  pref_47.geojson   ← 沖縄県
```

---

## HTMLとの連携（D3.js側）

HTMLのコードはファイル名の先頭2桁（都道府県コード）で市区町村を都道府県にマッピングしています。

```javascript
const codeStr = f.properties.code || "";
const prefCodeStr = codeStr.slice(0, 2); // 先頭2桁 = 都道府県コード
const id = parseInt(prefCodeStr, 10);
```

そのため、政令市の各区が複数フィーチャーに分かれていても（例：札幌市10区）、都道府県単位の色分けは正常に動作します。

---

## dissolve.py スクリプト全文

```python
"""
GeoJSONのポリゴンを行政区域コード（N03_007）単位で統合する。
ポリゴンの結合はMultiPolygonとして単純にまとめる方式（外部ライブラリ不要）。
簡略化（RDP）も同時に実施。
"""
import json
import math
import sys
from collections import defaultdict

def point_line_distance(point, start, end):
    if start[0] == end[0] and start[1] == end[1]:
        return math.sqrt((point[0]-start[0])**2 + (point[1]-start[1])**2)
    n = abs((end[1]-start[1])*point[0] - (end[0]-start[0])*point[1] + end[0]*start[1] - end[1]*start[0])
    d = math.sqrt((end[1]-start[1])**2 + (end[0]-start[0])**2)
    return n / d

def simplify_rdp(points, epsilon):
    if len(points) < 3:
        return points
    dmax = 0
    index = 0
    end = len(points) - 1
    for i in range(1, end):
        d = point_line_distance(points[i], points[0], points[end])
        if d > dmax:
            index = i
            dmax = d
    if dmax > epsilon:
        left = simplify_rdp(points[:index+1], epsilon)
        right = simplify_rdp(points[index:], epsilon)
        return left[:-1] + right
    else:
        return [points[0], points[end]]

def simplify_ring(ring, epsilon):
    simplified = simplify_rdp(ring[:-1], epsilon)
    if len(simplified) < 3:
        return ring  # 小さいポリゴンはそのまま保持（離島対策）
    simplified.append(simplified[0])
    return simplified

def simplify_polygon(coords, epsilon):
    return [simplify_ring(ring, epsilon) for ring in coords]

def dissolve_and_simplify(input_path, output_path, epsilon):
    with open(input_path, 'r', encoding='utf-8') as f:
        data = json.load(f)

    groups = defaultdict(lambda: {'properties': None, 'polygons': []})

    for feature in data['features']:
        geom = feature.get('geometry')
        if geom is None:
            continue
        props = feature['properties']
        code = props['N03_007']
        if groups[code]['properties'] is None:
            groups[code]['properties'] = props
        if geom['type'] == 'Polygon':
            groups[code]['polygons'].append(geom['coordinates'])
        elif geom['type'] == 'MultiPolygon':
            groups[code]['polygons'].extend(geom['coordinates'])

    print(f"統合前: {len(data['features'])} → 統合後: {len(groups)} フィーチャー")

    new_features = []
    for code, group in sorted(groups.items()):
        simplified_polygons = [simplify_polygon(poly, epsilon) for poly in group['polygons']]
        if len(simplified_polygons) == 1:
            geometry = {'type': 'Polygon', 'coordinates': simplified_polygons[0]}
        else:
            geometry = {'type': 'MultiPolygon', 'coordinates': simplified_polygons}

        p = group['properties']
        new_props = {
            'code': p['N03_007'],
            'pref': p['N03_001'],
            'city': p['N03_004'],
        }

        new_features.append({
            'type': 'Feature',
            'properties': new_props,
            'geometry': geometry
        })

    new_data = {'type': 'FeatureCollection', 'features': new_features}
    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump(new_data, f, ensure_ascii=False, separators=(',', ':'))

    size_kb = len(json.dumps(new_data, ensure_ascii=False).encode()) / 1024
    print(f"出力サイズ: {size_kb:.0f}KB → {output_path}")

dissolve_and_simplify(sys.argv[1], sys.argv[2], float(sys.argv[3]))
```

---

## 更新履歴

| 日付 | 内容 |
|---|---|
| 2025年 | 47都道府県すべての変換完了 |
| 2025年 | dissolve.py作成・福岡県（pref_40）で動作確認 |

---

*元データ出典: 国土地理院「国土数値情報 行政区域データ」2025年1月1日版*

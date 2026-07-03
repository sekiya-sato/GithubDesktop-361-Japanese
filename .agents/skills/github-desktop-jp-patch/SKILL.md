---
name: github-desktop-jp-patch
description: GitHub Desktop の新バージョンに対応した日本語化 CSS パッチ（renderer-{VERSION}-patch.css）を作成する。ユーザーが「新しいバージョンのパッチを作りたい」「org に最新ファイルを置いたのでパッチ更新して」などと言った場合に使用する。
---

# GitHub Desktop 日本語化パッチ作成スキル

GitHub Desktop の Windows 版リソース（主に `renderer.js` / `renderer.css`）に対し、CSS で英語テキストを隠し、日本語テキストを `::after` 疑似要素で表示するパッチファイルを作成・更新するワークフロー。

## 発動タイミング

ユーザーが以下のような発言をした場合にこのスキルを使用する：

- 「新しいバージョンのパッチを作りたい」
- 「org に最新ファイルを置いたのでパッチ更新して」
- 「renderer-X.Y.Z-patch.css を作成して」
- 「GitHub Desktop の日本語パッチを更新して」

## 命名規則

- パッチファイル名: `renderer-{VERSION}-patch.css`
- `{VERSION}` は **ドットなし** のセマンティックバージョンとする
  - 例: v3.6.2 → `renderer-362-patch.css`
  - 例: v3.7.0 → `renderer-370-patch.css`
  - 例: v3.10.1 → `renderer-3101-patch.css`
- 既存の最新パッチファイルをベースにする
  - 例: `renderer-362-patch.css` があれば、それをベースに `renderer-363-patch.css` を作る

## フォルダ構成の約束

```text
{repo-root}/
├── org/                          # 最新版の GitHub Desktop オリジナルリソース
│   ├── renderer.js
│   ├── renderer.css
│   ├── main.js
│   ├── package.json              # version フィールドでターゲットバージョンを確認
│   └── ...
├── renderer-361-patch.css        # 過去のパッチ
├── renderer-362-patch.css        # 現在のパッチ
├── renderer-363-patch.css        # これから作る新パッチ
└── README.md                     # 適用手順書（あれば更新）
```

- `org/` には **必ず最新版** のファイルを置く
- 作業前に `org/package.json` の `version` を確認し、ターゲットバージョンを認識する

## 作業手順

### Step 1: ターゲットバージョンを確認

1. `org/package.json` を読み、 `version` フィールドを確認
2. ファイル名を決定: `renderer-{VERSION_WITHOUT_DOTS}-patch.css`
3. 既存の最新パッチファイルを特定する

### Step 2: UI 構造の変更有無を調べる

GitHub Desktop のリリースはパッチバージョン（3.6.1 → 3.6.2 など）では UI 構造が変わらないことが多いが、必ず確認する。

```bash
# 前バージョンと今バージョンのソース差分を取得（タグ名は実際のリリースタグに合わせる）
curl -L "https://github.com/desktop/desktop/compare/release-{OLD}...release-{NEW}.diff" | head -n 200
```

または Web ブラウザ / API で確認：

```
https://api.github.com/repos/desktop/desktop/compare/release-{OLD}...release-{NEW}
```

確認ポイント：

- `app/src/ui/` 以下のコンポーネントに変更がないか
- `app/src/main-process/menu/` 以下のメニュー構造に変更がないか
- 設定ダイアログ、タブ、差分ビュー、ブランチボタン、空状態など本パッチのセレクタ対象コンポーネントに影響する変更がないか

**メニュー文言は `main.js` に含まれる**ため、`renderer.js` 内に `"New Repository"` などの文字列が無くても正常。CSS パッチはレンダラー側の DOM 構造を対象とする。

### Step 3: セレクタの整合性を確認

既存パッチに含まれる主要な ID / クラスが、新しい `org/renderer.css` にも存在するか確認する：

```python
python3 -c "
with open('org/renderer.css') as f:
    css = f.read()
selectors = [
    '#app-menu-bar',
    '#changes-tab',
    '#history-tab',
    '#repository-sidebar',
    '.branch-button',
    '.changes-interstitial',
    '#diff',
    '#preferences',
    '#__Dialog_preferences_Options',
]
for s in selectors:
    print(s, 'OK' if s in css else 'MISSING')
"
```

もし `MISSING` が出た場合、新しい `renderer.js` / `renderer.css` を解析し、セレクタを更新する必要がある。

### Step 4: 新パッチファイルを作成

UI 構造に変更が無ければ：

1. 最新の既存パッチをコピー
2. 先頭のバージョン注釈コメントを新バージョンに更新
3. 必要に応じてリージョンコメント（`#region 日本語化追加 ...` など）も最新に更新

```bash
# 手動で行う場合の例
cp renderer-362-patch.css renderer-363-patch.css
# 先頭コメントを v3.6.3 向けに書き換える
```

UI 構造に変更があれば：

1. 変更されたコンポーネントの英語文字列を `org/renderer.js` / `org/main.js` から検索
2. 新しい DOM 構造に合わせて CSS セレクタを修正
3. 日本語訳は既存パッチのものを流用し、必要に応じて修正

### Step 5: 品質確認

1. 新パッチファイルの CSS 構文が壊れていないか目視確認
2. 前パッチとの差分を確認し、意図しない変更がないかチェック

```bash
diff -u renderer-{OLD}-patch.css renderer-{NEW}-patch.css
```

3. `org/renderer.css` の末尾に新パッチを貼り付けた想定で、セレクタが衝突・重複していないか確認

### Step 6: README の更新（オプション）

ユーザーが求めた場合、または新バージョンの適用手順が変わった場合は `README.md` を更新する。

更新例：

```markdown
## v3.6.3 への適用方法

`%LOCALAPPDATA%\GitHubDesktop\app-3.6.3\resources\app\renderer.css` の末尾に `renderer-363-patch.css` の内容を貼り付けます。
```

## 注意事項

- **型安全性の制約はこのスキルには関係ない**が、CSS ファイルの構文エラーは絶対に残さない
- 新バージョンの `org/renderer.js` を解析する際は、必ず `org/package.json` の `version` と一致しているか確認する
- ユーザーが `org/` にファイルを置いたと言っている場合、まず `git status` や `ls -la org/` で存在を確認してから作業を開始する
- パッチ作成後は `git diff` / `git status` で意図したファイルだけが変更されているか確認する

## トラブルシューティング

### パッチを貼り付けても日本語化されない

- `renderer.css` の末尾に正しく貼り付けられたか確認
- セレクタ対象の DOM が新バージョンで変わっていないか確認（Step 2, 3）
- アプリを再起動してキャッシュを破棄したか確認

### `org/package.json` のバージョンとファイル名が不一致

- ユーザーに確認: `org/` に置いたファイルは本当に目標バージョンか？
- 不一致の場合、正しいファイルが配置されるまで新パッチ作成を保留する

### 新バージョンで大幅な UI 変更があった

- CSS パッチのアプローチが通用しなくなる可能性がある
- その場合はユーザーに「セレクタの大幅な見直しが必要」と報告し、対象コンポーネントごとに修正方針を確認する

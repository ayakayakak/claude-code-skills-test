---
name: create-release-pr
description: main ブランチから release ブランチへの本番リリース用 PR を作成する。「本番リリース PR を作って」「リリース PR を作成」「main を release にマージ」などのフレーズが出たら必ずこのスキルを使うこと。差分に含まれる各 PR のタイトルと Tested セクションを自動収集し、ステージング確認用チェックリスト付きのドラフト PR を自動生成する。
---

# 本番リリース PR 作成

`main` ブランチから `release` ブランチへの本番リリース用 PR を作成する。

## 使用方法

このリポジトリで Claude Code を開き、以下のいずれかの方法で起動する:

```
/create-release-pr
```

または自然言語で:

```
本番リリース PR を作って
```

Claude が自動で以下を行う:

1. `main` と `release` の差分から対象 PR を特定
2. 各 PR のタイトルと Tested 項目を収集
3. PR のタイトル・本文のドラフトを提示
4. 確認後、ドラフト状態で PR を作成

作成された PR はドラフト状態なので、内容を確認してから手動で **Open** に変更すること。

## 手順

### 1. 差分に含まれるマージコミットを特定する

```bash
git fetch origin
git log origin/release..origin/main --merges --oneline
```

出力例:

```
def5678 Merge pull request #142 from ayakayakak/feature-branch
abc1234 Merge pull request #141 from ayakayakak/another-branch
```

コミットメッセージの `#数字` パターンから PR 番号を抽出する。番号は古い順（小さい順）に並べる。

差分に PR が存在しない場合は、「main と release の差分に PR が見つかりませんでした」とユーザーに伝えて中断する。

### 2. 各 PR の情報を取得する

各 PR 番号に対して以下を実行し、タイトル・URL・本文を取得する:

```bash
gh pr view <number> --repo ayakayakak/claude-code-skills-test --json title,body,url
```

### 3. Tested セクションを抽出・変換する

各 PR の本文から `## Tested` 見出しから次の `##` 見出し（または文末）までの内容を抽出する。

抽出後、以下の変換を行う:

**チェックボックスの変換**（行の形式を変えず、チェック状態のみ変更する）:

- `- [x]` → `- [ ]`（チェック済みを未チェックに戻す）
- `- [ ]` → `- [ ]`（未チェックはそのまま）
- チェックボックスのない箇条書き（`- テキスト`）はそのまま維持する
- 平文のテキスト行（箇条書きでない行）もそのまま維持する

**URL の置換**:
- `http://localhost:3000/` → `https://staging.ayakayakak.co.jp/`（チェックボックスの有無にかかわらず全行に適用）

`## Tested` セクションが存在しない PR については、Staging Tested セクションにそのPRの項目を省略し、ユーザーに警告する。

### 4. PR タイトルを組み立てる

```
(本番リリース) タイトル141、タイトル142
```

- 先頭は `(本番リリース) `（末尾に半角スペース1つ）
- 各タイトルは `、`（読点）区切り
- 古い順（PR番号が小さい順）に並べる

### 5. PR 本文を組み立てる

```markdown
## PR
- https://github.com/ayakayakak/claude-code-skills-test/pull/141
- https://github.com/ayakayakak/claude-code-skills-test/pull/142

## Staging Tested
### https://github.com/ayakayakak/claude-code-skills-test/pull/141
- [ ] https://staging.ayakayakak.co.jp/article/1 の表示が崩れていないこと
- [ ] https://staging.ayakayakak.co.jp/article/2 の表示が崩れていないこと
### https://github.com/ayakayakak/claude-code-skills-test/pull/142
- [ ] https://staging.ayakayakak.co.jp/comic の表示が崩れていないこと
```

- `## PR` セクション: 古い順に PR の URL を箇条書き
- `## Staging Tested` セクション:
  - 差分に含まれる PR が **2つ以上**の場合: 各 PR の URL を `###` 見出しにし、その下に変換済みの Tested 項目を列挙
  - 差分に含まれる PR が **1つだけ**の場合: `###` 見出しは付けず、変換済みの Tested 項目を直接列挙する

### 6. ユーザーに確認してから PR を作成する

承認を得たら本文を一時ファイルに書き出し、`--body-file` で渡して**ドラフト PR** を作成する。レビュー準備ができたら手動で Ready for review に変更する:

```bash
cat > /tmp/release-pr-body.md << 'EOF'
<本文>
EOF

gh pr create \
  --repo ayakayakak/claude-code-skills-test \
  --base release \
  --head master \
  --draft \
  --title "<タイトル>" \
  --body-file /tmp/release-pr-body.md

rm /tmp/release-pr-body.md
```

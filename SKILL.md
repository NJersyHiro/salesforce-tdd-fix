---
name: salesforce-tdd-fix
description: Use when fixing a Salesforce bug (Flow, Apex, LWC) using TDD — covers requirements gathering, bug reproduction, unit test scenario creation, fix planning with user approval, implementation, pre-deploy backup, ST execution, screenshot archival, and cleanup, with agent-based review gates between steps.
---

# Salesforce TDD バグ修正ワークフロー

## このスキルでやること

Salesforce（Flow / Apex / LWC）のバグを、TDDの手順（RED→GREEN）で安全に修正する。

**11ステップ構成。ユーザー承認なしに実装を開始してはならない。**

---

## ステップ一覧と進め方

| # | ステップ | ゲート |
|---|---------|--------|
| 1 | 要件把握 | GATE-1（code-reviewer） |
| 2 | 不具合再現 | GATE-2（code-reviewer） |
| 3 | テストシナリオ作成（RED） | GATE-3（code-reviewer） |
| 4 | テストデータ確認・作成 | GATE-4（code-reviewer） |
| 5 | 修正方針の確認 | GATE-5（code-reviewer） |
| **6** | **ユーザー承認** ← **HARD STOP** | ユーザー返答待ち |
| 7 | 実装（GREEN）— コード修正のみ | GATE-7（code-reviewer） |
| 8 | **バックアップ → デプロイ → UI確認**（この順で必ず実行） | なし |
| 9 | STシナリオ実行 | GATE-9（test-validator） |
| 10 | スクショ保管 | なし |
| 11 | クリーンアップ | なし |

**フロー:** Step N → 作業 → GATE → ✅ならStep N+1 / ❌なら同じStepを修正して再提出

---

## ⚡ 最初に必ずやること（MANDATORY）

スキルを呼び出したら、**作業開始前に必ず**以下の11タスクをTaskCreateで登録する。

```
TaskCreate × 11（上記ステップ一覧と同じ内容）
```

- 各ステップ開始時: `TaskUpdate → in_progress`
- 各ステップ完了時: `TaskUpdate → completed`

**なぜ必須か:** タスクリストなしだと、GATEやHARD STOPを無意識にスキップする。特にStep 6（ユーザー承認）を飛ばして実装を始めるリスクがある。

---

## GATEの出し方（共通）

ゲートがあるステップが終わったら、以下を実行する:

```
Agent({
  subagent_type: "superpowers:code-reviewer",
  prompt: "以下のステップ完了内容を評価してください。\n\nステップ: [ステップ名]\n完了内容: [実施した内容・成果物]\n評価基準:\n[各ステップのGATE評価基準をそのまま貼る]\n\n問題なければ '✅ OK' を、修正が必要なら '❌ [具体的な修正指示]' を返してください。"
})
```

**❌が返ってきたら:** 指摘内容を修正して同じステップを再実行し、再度GATEに出す。

---

## Step 1: 要件把握

**目的:** バグの内容・影響範囲・API名を正確に把握する。

**作業:**
1. BacklogチケットとコメントをMCPで取得する
2. 現在の動作 vs 期待動作を表形式で整理する
3. 対象オブジェクト・フィールドのAPI名を特定する
4. 関連するFlow / Apex / LWCを特定する

```
mcp__backlog__get_issue(issueKey: "COMPASS-XXXX")
mcp__backlog__get_issue_comments(issueKey: "COMPASS-XXXX")
```

**GATE-1 評価基準:**
- [ ] 現状動作と期待動作が明確に分離されているか
- [ ] 対象オブジェクト・フィールドのAPI名が正確か
- [ ] 影響範囲（関連Flow / Apex / LWC）が特定されているか

---

## Step 2: 不具合再現

**目的:** vpartialで実際にバグを再現し、スクショで証跡を残す。

**作業:**
1. vpartialにPlaywright MCPでログインする（システム管理者でよい）
2. 対象レコードを特定または作成する
3. バグを引き起こす操作を実施する
4. 結果をスクショで記録する

**ログイン方法:**
```bash
sf org open --target-org vpartial --url-only
# 出力されたfrontdoor URLをPlaywright MCPに渡す
```

スクショ保存先: `.claude/screenshots/repro-{概要}.png`

**GATE-2 評価基準:**
- [ ] バグが画面またはログで確認できるか
- [ ] 再現手順が明確に記録されているか
- [ ] スクショに証跡が含まれているか

---

## Step 3: テストシナリオ作成（RED）

**目的:** 修正後に必ず通らなければならないシナリオを、実装前に定義する（TDDのREDフェーズ）。

**作業:** 以下の形式でシナリオ表を作成する。

| # | Given（前提条件・データ状態） | When（操作） | Then（期待結果） |
|---|----------------------------|------------|----------------|
| 1 | ... | ... | ... |

シナリオに含めること:
- バグが修正されたことを確認するメインシナリオ
- エッジケース（ガード条件が機能するか）
- 既存動作が壊れていないことを確認する回帰シナリオ

**GATE-3 評価基準:**
- [ ] 修正すべき動作がすべてテストシナリオに含まれているか
- [ ] エッジケースが含まれているか
- [ ] 既存動作の回帰シナリオが含まれているか

---

## Step 4: テストデータ確認・作成

**目的:** Step 3のシナリオを実行するためのデータがvpartialに存在するか確認する。

> ⚠️ **このステップでコードを修正しない。** コード修正はStep 8で行う。シナリオの説明で「修正後の確認」と書いても、それは「Step 8修正後にStep 10で確認する計画」を意味する。

**作業:**
1. 各シナリオに必要なレコードをSOQLで検索する
2. 存在しないデータはPlaywright MCPで作成する（作成したものはクリーンアップリストに記録）
3. STでスキップするシナリオは、その理由を明記する

**SOQLの例:**
```bash
sf data query \
  --query "SELECT Id, Name FROM [オブジェクト] WHERE [条件] LIMIT 5" \
  --target-org vpartial
```

**スキップ理由の書き方:**
- データ作成が困難な場合: 理由と代替確認手段（Apex UTでカバー等）を記載する
- 実装済みのガード処理でカバーされる場合: コードのどの行でカバーされているかを記載する

**GATE-4 評価基準:**
- [ ] 全シナリオを実行できるデータが揃っているか（またはスキップ理由が適切か）
- [ ] テスト後に削除・復元が可能な状態か
- [ ] クリーンアップリストが作成されているか

---

## Step 5: 修正方針の確認

**目的:** 修正案を技術的に整理し、実装前にレビューを受ける。

**作業:** 以下のテンプレートで修正方針を作成する。

```
## 対象コンポーネント
- [Flow名 / Apexクラス名 / LWC名]

## 現状の問題
- [問題のある行・条件・ロジック]

## 修正内容
- [追加・変更・削除するロジック]

## 影響範囲
- [変更により影響を受ける他のコンポーネント]

## リスク
- [デグレの可能性・注意点]
```

**GATE-5 評価基準:**
- [ ] 修正内容がStep 3のシナリオをすべてカバーしているか
- [ ] 既存動作への影響が分析されているか
- [ ] デグレリスクが洗い出されているか

---

## Step 6: ユーザー承認（HARD STOP）

> ⚠️ **AIはユーザーの返答があるまで次に進んではならない。**

以下の形式でチャットに報告し、返答を待つ。**初心者が読んでも理解できる説明にすること。**

```
## 修正方針の確認お願いします（COMPASS-XXXX）

### 今回のバグについて（初心者向け説明）
[技術用語を避け、「何が起きていたか」「なぜ問題か」「どう直すか」を
 平易な日本語で3〜5行で説明する。

 例:
 「現在、○○画面で△△を操作すると□□が起きてしまいます。
  本来は××になるべきところ、◇◇という条件が抜けていたために
  正しく動いていませんでした。
  今回はその条件を追加することで正しく動くようにします。」]

### 修正するファイルと変更内容（初心者向け説明）
| ファイル名 | 何をするファイルか（一言） | 今回の変更内容 |
|-----------|----------------------|--------------|
| [ファイル名] | [例: 期間按分の計算を担当するApexクラス] | [例: 子見積生成時に承認フラグをTrueにセットする処理を追加] |

### テストシナリオ（全N件）
[各シナリオに「なぜこのテストが必要か」を一言添える]

| # | Given（前提条件） | When（操作） | Then（期待結果） | なぜ必要か |
|---|----------------|------------|----------------|----------|
（Step 3の表 + なぜ必要か列を追加）

### 懸念点・確認事項
（あれば記載。ない場合は「特になし」と明記する）

問題なければ実装を開始します。よろしいですか？
```

**書き方のポイント:**
- Salesforceの専門用語（FlexiPage、FLS、SOQL等）はそのまま使わず、括弧書きで補足する
- 「何のために」「なぜ」を省略しない
- ファイル名だけでなく「そのファイルが何をするか」を必ず説明する

---

## Step 7: 実装（GREEN）— コード修正のみ

**目的:** Step 5の修正方針に従ってコードをローカルで修正する。**デプロイはStep 9で行う。**

### デプロイ前の差分確認（必須）

> ⚠️ コード修正前に必ず対象環境の最新版をリトリーブして差分を確認すること。

```bash
sf project retrieve start --source-dir [対象ファイルパス] --target-org vpartial
git diff [対象ファイルパス]
```

### 実装（コード修正のみ）

ローカルファイルを修正する。デプロイはまだしない。

**GATE-7 評価基準:**
- [ ] 実装がStep 3のシナリオをすべてカバーしているか
- [ ] コンパイルエラーがない（ローカルで構文確認）
- [ ] デプロイ前バックアップ（Step 8）をまだ実施していないことを確認

---

## Step 8: バックアップ → デプロイ → UI確認

> ⚠️ この3つは必ずこの順番で実行すること。バックアップなしにデプロイしてはならない。UIで確認するまで「完了」と言ってはならない。

### 8-1: バックアップ（デプロイ前に必ず実施）

```bash
mkdir -p .claude/backup/$(date +%Y-%m-%d)

# vpartial環境の現在の状態をバックアップ
cp [対象ファイル] .claude/backup/$(date +%Y-%m-%d)/[ファイル名].vpartial-before
```

### 8-2: デプロイ

```bash
sf project deploy start \
  --source-dir [対象ファイルパス] \
  --target-org vpartial
```

### 8-3: デプロイ後のUI確認（必須）

> ⚠️ Apexテストが通っても、UIで確認するまで「動いた」と言ってはならない。
> デプロイ済みコードとローカルの乖離・FLSの欠落・画面キャッシュ等を検知するために必ずUIで目視確認する。

```
1. sf org open --target-org vpartial --url-only でfrontdoor URLを取得
2. Playwright MCP でvpartialにログイン
3. 修正した機能を実際に操作し、Step 3 のシナリオ#1（最重要シナリオ）を1件だけ画面で確認
4. スクリーンショットを撮って証跡を保存: .claude/screenshots/deploy-verify-{概要}.png
```

**UI確認でNGだった場合:** Apexテストが通っていてもNGとして扱い、原因を調査すること。
- FLSが設定されていない → プロファイルに権限を追加してデプロイ
- フィールドが画面に表示されていない → レイアウト・FlexiPageを確認
- 値がセットされていない → デプロイが正常に完了したか再確認

---

## Step 9: STシナリオ実行

**目的:** Step 3のシナリオを全件vpartialで実行し、スクショで証跡を残す。

### ユーザー切り替え（必須）

> ⚠️ STは必ず営業担当ユーザーで実施する。システム管理者は権限が広すぎて本番と条件が異なる。

```
設定 → ユーザー → 対象ユーザーの「このユーザーとしてログイン」
URL: https://vectorinc--vpartial.sandbox.my.salesforce-setup.com/lightning/setup/ManageUsers/page?address=%2F005fR000004Lqan%3Fnoredirect%3D1%26isUserEntityOverride%3D1
```

### 実行

各シナリオ実行後にスクショを撮る。スクショ保存先: `.claude/screenshots/st-case{N}-{概要}.png`

### GATE-9（test-validatorで実施）

```
Agent({
  subagent_type: "test-validator",
  prompt: "以下のSTテスト結果を検証してください。\n\nチケット: COMPASS-XXXX\nテストケース:\n[シナリオ表]\nスクショ:\n[ファイルパス一覧]\n\n確認事項:\n1. スクショで期待値が確認できるか\n2. エラーケースでエラーメッセージが表示されているか\n3. 回帰シナリオが変更されていないか"
})
```

**GATE-9 評価基準:**
- [ ] 全シナリオがスクショで確認できるか
- [ ] エラーケースでエラーメッセージが表示されているか
- [ ] 回帰シナリオ（既存動作）が変更されていないか

---

## Step 10: スクショ保管

> ⚠️ クリーンアップ（Step 11）の前に必ず実施すること。

```bash
FOLDER=~/screenshots/COMPASS-XXXX-{修正概要}
mkdir -p "$FOLDER"
mv .claude/screenshots/st-case*.png "$FOLDER/"
mv .claude/screenshots/repro-*.png "$FOLDER/" 2>/dev/null || true
```

---

## Step 11: クリーンアップ

> ⚠️ Step 10のスクショ保管が完了してから実施すること。

| 対象 | 方法 |
|------|------|
| STのために作成したレコード | `sf data delete record --sobject ... --target-org vpartial` |
| 変更したフィールド値 | 元の値に戻す（SOQL update または Playwright） |
| リトリーブしたメタデータ（コミット不要分） | `git checkout force-app/` |
| ローカルスクショ | `rm -rf .claude/screenshots/st-case*.png .claude/screenshots/repro-*.png` |
| ブラウザ | `mcp__plugin_playwright_playwright__browser_close` |
| Backlogチケット | `mcp__backlog__update_issue(issueKey: "COMPASS-XXXX", statusId: 3)` |

---

## Step 12: コミット・PR作成・マージ

> ⚠️ クリーンアップ（Step 11）完了後に実施すること。

### ブランチルール

- **作業ブランチ:** 常に `Sprint-FY26-2_YAMAMOTO` を使用する（新規ブランチは作らない）
- **マージ先:** `develop`
- **ブランチ削除:** マージ後もブランチは削除しない

### 手順

**1. ブランチ切り替え**

```bash
git checkout Sprint-FY26-2_YAMAMOTO
```

**2. このセッションで変更したファイルのみをステージング**

```bash
# 実装ファイル・メタデータのみを指定してadd（無関係のファイルは含めない）
git add [実装したファイルを個別に指定]
```

> ⚠️ `git add .` や `git add -A` は使わない。リトリーブで混入したメタデータ同期差分（`<externalId>false</externalId>` 削除等）を誤って含めないよう、ファイルを個別に指定する。

**3. コミット**

```bash
git commit -m "feat/fix: [変更内容の概要]

[詳細説明]"
```

**4. push**

```bash
git push -u origin Sprint-FY26-2_YAMAMOTO
```

**5. PR作成**

```bash
gh pr create \
  --base develop \
  --head Sprint-FY26-2_YAMAMOTO \
  --title "[コミットメッセージと同じタイトル]" \
  --body "## Summary
- [変更内容の箇条書き]

## 変更ファイル
| ファイル | 変更内容 |
|---------|--------|
| ... | ... |

## Test plan
- [ ] vpartialで動作確認済み
- [ ] Apexテストパス済み"
```

**6. マージ（ユーザーの承認後）**

> ⚠️ マージはユーザーの「マージしていいよ」の確認を得てから実行する。

```bash
# --delete-branch は付けない（ブランチを残す）
gh pr merge [PR番号] --squash
```

---

## よくあるミス

| ミス | 対策 |
|------|------|
| Step 6のHARD STOPをスキップして実装を開始した | Step 6に戻り、ユーザーの承認を得てから再開する |
| Step 4でコードを修正してしまった | コード修正はStep 8のみ。Step 4は「どのデータでテストするか」を確認するだけ |
| Step 11の前にスクショを削除した | Step 10で `~/screenshots/COMPASS-XXXX-{概要}/` に移動してから削除する |
| sysadminでSTを実施した | Step 9の手順で営業担当ユーザーに切り替える |
| Apexテストが通ったのでUIを確認しなかった | Apexテストは「コードの正しさ」を証明するが「実際にデプロイされたUIで動くか」は別物。Step 9-3のUI確認は省略不可 |
| バックアップなしでデプロイした | Step 9-1を必ず先に実施する。バックアップ→デプロイの順番は絶対 |
| デプロイ前に差分確認を省略した | 環境差異で意図しない上書きが発生するリスクがある |
| バックアップなしでデプロイした | Step 9を必ずデプロイ前に実施する |
| GATEをスキップした | 問題が次ステップに持ち越されると手戻りが大きくなる |
| developブランチに直接コミットした | 必ず `Sprint-FY26-2_YAMAMOTO` に切り替えてからコミットする |
| `git add .` でメタデータ同期差分を含めた | ステージングはファイルを個別に指定する |
| マージ時に `--delete-branch` を付けた | ブランチは残すため付けない |
| ユーザー確認前にマージした | 「マージしていいよ」の返答を待ってから `gh pr merge` を実行する |

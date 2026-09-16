# PseudoLab セキュリティ対応メモ

作成日: 2026-09-04
対象: `index.html`（GitHub Pages でホスティング / Firebase Auth + Firestore）

---

## ⚠️ まず最初にやること（未着手）

コード側の対策は完了済み。**以下はコンソール作業なので手動でやる必要がある。**
特に 1 番は他の全対策を合わせたより重要。

### 1. Firestore セキュリティルールの確認と反映 ← 最優先

Firebase Console → Firestore Database → **ルール** を開く。

`allow read, write: if true;` や `request.time < timestamp.date(...)` があれば
**テストモード＝全世界に公開状態**。この場合、URL を知る誰でも全生徒のノートを
読み・書き換え・削除できる。

対応: このリポジトリの `firestore.rules` の中身を貼り付けて「公開」。

確認方法: Rules Playground で、別の uid として
`notebooks/{他人のuid}/saved/{任意ID}` への read / write が**拒否**されること。

### 2. 承認済みドメインの整理

Firebase Console → Authentication → Settings → 承認済みドメイン
→ GitHub Pages のドメインと `localhost` **以外を削除**。

### 3. API キーに HTTP リファラー制限

Google Cloud Console → 認証情報 → 該当 API キー → アプリケーションの制限
→ 「HTTP リファラー」→ `https://<username>.github.io/*` を登録。

※ API キーが HTML に見えていること自体は Firebase では正常。制限をかけることで
他サイトからの流用を防ぐ。

### 4. App Check（reCAPTCHA v3）

Firebase Console → App Check → reCAPTCHA v3 を登録 → Firestore で「強制」。
これが Firebase 側の最終防壁で、**ノート作成数の無制限問題**（ルールでは件数を
制限できない）への実質的な唯一の対策でもある。

有効化する場合は `index.html` に初期化コード 5 行の追加が必要（サイトキー取得後）。

### 5. 予算アラート

Google Cloud → お支払い → 予算とアラート。課金枯渇型の攻撃対策。

---

## 対応済み（2026-09-04）

### 削除
- `index_source.html`（旧版）を `git rm`。※ git 履歴には残る。

### XSS（保存型）の修正 — 3 箇所

いずれも `innerHTML` に外部由来の文字列を連結していた。DOM API に置換。

| 箇所 | 内容 |
|---|---|
| `loadNotebookList()` | ノートタイトルを `createElement` + `textContent` に。`onclick="fbOpen('...')"` の文字列連結も `addEventListener` のクロージャに変更 |
| `refreshStatePanel()` | 変数名・型・値を `insertCell()` + `textContent` で描画 |
| `promptInput()` | 変数名をテキストとして注入 |

**攻撃シナリオ（塞いだもの）**: ノート名を `<img src=x onerror="...">` にして保存
→ 一覧を開くたび任意 JS が実行 → Firebase の ID トークンや全ノートを窃取。
Firestore ルールがテストモードだと他人のアカウントに書き込めるため、
**連鎖して全員に広がるワーム**になり得た。

### Firestore から読んだデータの検証
- `fbOpen()` で `title` / `cells` / `code` / `output` の型をチェック
- `sanitizeSharedState()` を新設 — `__proto__` / `constructor` / `prototype` を除去し、
  `{type, value, cellId}` の形以外を破棄（プロトタイプ汚染対策）

### CSP（Content Security Policy）の追加
`<head>` に `<meta http-equiv="Content-Security-Policy">` を追加。
**要は `connect-src` で外部送信を遮断する**のが目的で、万一注入されても
攻撃者のサーバーへデータを持ち出せない。

- `'unsafe-inline'` は単一 HTML 構成（インライン script + onclick 属性）のため必須
- `frame-ancestors` は `<meta>` では効かない → clickjacking 対策は GitHub Pages では不可

`<meta name="referrer" content="strict-origin-when-cross-origin">` も追加。

### CDN バージョンの完全固定（供給網対策）
esm.sh の CodeMirror 6 本を patch まで固定:

```
@codemirror/view@6.43.11   @codemirror/state@6.7.2
@codemirror/commands@6.11.0  @codemirror/language@6.12.4
@lezer/highlight@1.2.3      @codemirror/autocomplete@6.20.3
```

Firebase SDK は元から `10.12.0` 固定。

### 保存サイズの上限
`doSave()` にタイトル 200 文字 / 300 セル / 40 万文字のガード。
`firestore.rules` の制限と一致させ、拒否時に理由が表示されるように。

### localStorage 復元の検証
`restoreLocalNotebook()` に型チェックと上限を追加。

> **注意**: GitHub Pages では `<username>.github.io` の**全リポジトリが同一オリジン**。
> 自分の他プロジェクトのページから PseudoLab の localStorage を読み書きできる
> （外部からは不可）。

### インタプリタの予約識別子ガード
`DECLARE __proto__ : INTEGER` が `\w+` にマッチして `this.env['__proto__']` に
到達していた。`checkName()` を追加し `__proto__` / `constructor` / `prototype` を拒否。

### `fbDelete()` のサインイン未チェック
サインアウト直後に押すと `user.uid` で例外 → ガード追加。

### バグ修正: 保存したノートの変数ステートが復元されない
`let notebookState = window._notebookState;` という**エイリアス変数**が原因。
`fbOpen()` は `window._notebookState` に新オブジェクトを丸ごと代入するのに、
エイリアスは古いオブジェクトを指したまま。そこへ `refreshStatePanel()` の
`window._notebookState = notebookState;` が古い方を書き戻していた。

→ エイリアスを削除し、全アクセスを `window._notebookState` に統一（7 箇所）。

---

## 変更していないもの（問題なしと判断）

- **疑似コードインタプリタ**: JS の `eval` を一切使わず自前評価。安全。
- **DoS ガード**: ループ 10000 回・呼び出し深さ 200 の上限が既にある。
- **実行結果の描画**: `textContent` を使用。
- 残る `innerHTML` は全て静的文字列、または数値カウンタのみを埋め込むもの。

---

## 検証結果（2026-09-04 時点）

- `node --check` — script ブロック 4 つすべて通過
- `sanitizeSharedState()` 単体テスト — プロトタイプ汚染なし / 不正型の除去 / `undefined` 入力 OK
- インタプリタ回帰テスト 13 パターン全通過
  （ARRAY / WHILE / REPEAT / IF-ELSE / CASE / PROCEDURE / CLASS / FILE /
  文字列関数 / INPUT / TYPEOF / 無限ループ停止 / 無限再帰停止）
- ローカル配信 HTTP 200、CSP ヘッダー確認済み

### 未実施 — ブラウザでの目視確認（要対応）

1. `python3 -m http.server 8765` → `http://localhost:8765/index.html`
2. DevTools コンソールに **CSP 違反が 0 件**（エディタに色が付いていれば CodeMirror は読めている）
3. サインイン → セル実行 → ☁️保存 → 📂一覧 → 開く → 削除 が全て通ること
4. **XSS 回帰テスト**: ノート名を `<img src=x onerror=alert(1)>` にして保存 →
   一覧を開いて**アラートが出ない**こと
5. **ステート復元テスト**: 変数を作る → 保存 → 開き直す → 変数パネルに値が復元されること

---

## 検討事項: Firebase Hosting への移行

急ぎではないが、移すと少し堅くなる。差は 1 点だけ。

| | GitHub Pages | Firebase Hosting |
|---|---|---|
| HTTPS | ✅ | ✅ |
| CSP | `<meta>`（実装済み・ほぼ同等） | HTTP ヘッダー（完全版） |
| **clickjacking 対策** | ❌ 不可能 | ✅ 可能 |
| HSTS | ❌ | ✅ |
| localStorage のオリジン分離 | ❌ 他リポジトリと共有 | ✅ 独立 |

既に Firebase を使っているので移行コストは `firebase deploy` のみ。
承認済みドメインの管理も 1 箇所にまとまる。

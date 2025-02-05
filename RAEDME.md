# Firebase Hosting Local Server Issue: TypeError: Cannot read properties of undefined (reading 'getTime')

この README では、Firebase CLI を使用してローカルサーバーを起動 (`firebase serve`) した際に発生するエラー  
```
TypeError: Cannot read properties of undefined (reading 'getTime')
```  
に関する問題の概要、再現手順、回避策、および今後の対応についてまとめています。

---

## 問題の概要

Firebase CLI の内部で使用されている [superstatic](https://github.com/firebase/superstatic) ライブラリにおいて、ファイルの最終更新日時（mtime）を取得する際、`stat.mtime.getTime()` が `undefined` となる場合があり、その結果エラーが発生します。  
このエラーは、以下のような状況で発生します。

- ローカルサーバー起動後、ブラウザで `http://localhost:5000/` にアクセスすると、500 エラーおよび「Unexpected error occurred.」が表示される。
- 特に、ブラウザが自動的に `/favicon.ico` をリクエストする際に発生する場合が多い。

GitHub の Issue [#7173](https://github.com/firebase/firebase-tools/issues/7173) や [#7941](https://github.com/firebase/firebase-tools/issues/7941) でも同様の現象が報告されています。

---

## 環境情報

- **Firebase CLI のバージョン:** 13.8.3 〜 最新版（現時点で）
- **Node.js のバージョン:**  
  - Node.js v22 で発生  
  - Node.js v20 では正常動作（回避策として推奨）
- **プラットフォーム:**  
  - Ubuntu / macOS / Windows 等

---

## 再現手順

1. Firebase CLI のインストール（例）:
   ```bash
   npm install -g firebase-tools
   ```

2. プロジェクトの初期化:
   ```bash
   firebase init
   ```
   - 「Hosting」を選択し、指示に従ってプロジェクトを設定します。

3. ローカルサーバーの起動:
   ```bash
   firebase serve --only hosting
   ```

4. ブラウザで `http://localhost:5000/` にアクセスすると、エラーが発生し、以下のようなログが表示されます:
   ```
   TypeError: Cannot read properties of undefined (reading 'getTime')
       at Responder.provider (.../superstatic/lib/providers/fs.js:89:38)
   ```

---

## 回避策および解決方法

### 1. Node.js のバージョンを v22 から v20 にダウングレード

最も簡単な回避策は、Node.js のバージョンを v22 から v20 に切り替えることです。  
nvm を利用している場合の例:

```bash
# Node.js v20 をインストールして使用する
nvm install 20
nvm use 20
node -v  # v20.x.x と表示されることを確認

# Firebase CLI の再インストール（必要な場合）
npm install -g firebase-tools

# ローカルサーバーの起動
firebase serve --only hosting
```

### 2. 一時的な手動修正（ワークアラウンド）

急ぎの場合、Firebase CLI に同梱されている superstatic のソースコードを一時的に修正する方法もあります。  
対象ファイルは通常、以下のパスにあります。

```
/usr/local/lib/node_modules/firebase-tools/node_modules/superstatic/lib/providers/fs.js
```

**修正箇所:**

対象の 89 行付近にある以下のコードを

```js
modified: stat.mtime.getTime(),
```

下記のように変更します。

```js
modified: stat.mtimeMs,
```

> **注意:** この修正は一時的なものであり、Firebase CLI のアップデート時に上書きされる可能性があるため、あくまで緊急措置としてご利用ください。

### 3. Firebase CLI（superstatic）のアップデートを待つ

- この問題は [superstatic v9.1.0](https://github.com/firebase/superstatic/releases/tag/v9.1.0) で修正済みです。  
- Firebase CLI 側がこの修正を取り込んだアップデートをリリースするまで、Node.js のバージョンダウングレードや一時的な手動修正での回避が必要です。

---

## 関連リンク・参考情報

- [GitHub Issue #7173: firebase serve: TypeError: Cannot read properties of undefined (reading 'getTime')](https://github.com/firebase/firebase-tools/issues/7173)
- [GitHub Issue #7941: Upgrade superstatic to 9.1.0](https://github.com/firebase/firebase-tools/issues/7941)
- [Node.js リリース情報](https://nodejs.org/en/about/previous-releases)

---

## まとめ

- **問題:** Firebase CLI を使用してローカルサーバーを起動した際、Node.js v22 でファイルの更新日時取得時にエラーが発生する。
- **回避策:**  
  - **推奨:** Node.js v20 にダウングレードする  
  - **一時対応:** 該当箇所を手動で修正する（`stat.mtime.getTime()` → `stat.mtimeMs`）
- **今後:** Firebase CLI のアップデート（superstatic v9.1.0 の取り込み）を待つ


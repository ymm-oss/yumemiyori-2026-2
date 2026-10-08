---
class: content
author: ゆき
profile: Flutterエンジニアです。
---

<div class="doc-header">
<div class="doc-title">WebView を実装するときの注意点</div>
  <div class="doc-author">ゆき</div>
</div>

# WebView を実装するときの注意点

Flutter エンジニアのゆきと申します。本稿では、アプリエンジニアやフロントエンドエンジニアに向けて、WebView を実装するときの注意点をまとめます。

コード例は Dart で書いていますが、Flutter の知識を前提にせず読める内容です。

レイアウトが崩れないようにするためのヒントや、Web からアプリの機能を呼ぶ仕組み「JavaScript bridge」のセキュリティ上の注意を紹介します。

## ひとつの画面にふたつの仕組み

WebView は、アプリの画面内に Web ページを表示する部品です。アプリが WebView の大きさや位置を決め、Web ページがその中の文字や画像、フォームを描きます。余白の取り方やリンクの行き先には、両方の実装が関わります。

<hr class="page-break"/>

## WebViewとブラウザーを使い分ける

Web ページをアプリ固有のボタンと並べたり、Web とアプリで操作をつないだりするなら、WebView が候補になります。そのとき、アプリ内に表示するページと外部ブラウザーへ渡すページも一緒に考えます。外部サイトを読むだけなら、Android の Custom Tabs など、ブラウザーで開く方法も候補です。

OAuth 2.0 や OpenID Connect を使うログインでは、認可画面を通常の WebView に埋め込まず、OS の認証セッションや外部ブラウザーで開きます。アプリ内の WebView ではアプリが画面や入力内容を読めるため、認証事業者が埋め込み画面を拒否することもあります。

<hr class="page-break"/>

## 画面端の余白を調整する

Safe Area（セーフエリア）は、ステータスバーやホーム操作の領域、画面の切り欠きに重ならず、文字やボタンを置ける範囲です。WebView を画面いっぱいに広げると、ページの要素がこれらの領域の下に隠れたり、押しづらくなったりします。画面の端まで使うデザインでも、重要な内容や操作ボタンは Safe Area 内に置きます。

アプリ側で WebView を Safe Area 内に収める方法と、Web 側で CSS の`env(safe-area-inset-*)`を使って余白を取る方法があります。画面全体の構成に合わせて、どちらが画面端の余白を受けもつかを決めます。

Web 側で下端の余白を取る例です。`env()`はブラウザーがページに知らせる環境値を読みます。ここでは、本文の下余白を最低 12px とし、下端の inset がそれより大きい場合はその値を使います。

```css
body {
  padding-bottom: max(12px, env(safe-area-inset-bottom, 0px));
}
```

Web ページを画面端まで広げる構成では、viewport の設定に`viewport-fit=cover`を指定します。`env()`の値は OS や WebView の配置によって変わります。アプリ側で Safe Area を確保しても、CSS の inset が 0 になるとは限りません。アプリと Web の両方で同じ端の余白を取ると、余白を二重に確保する場合があります。

実装後は、WebView の表示範囲と CSS の値を一緒に確かめます。アプリ側で Safe Area を確保した場合も、Web 側の設定が重なっていないか、ボタンが画面端に隠れていないかを確認します。

<hr class="page-break"/>

## 表示するページと呼べる機能を絞る

アプリ内に表示するページは、URL の許可リストで制限します。外部サイトが WebView に読み込まれると、そのページから JavaScript bridge の公開機能を呼べる場合があります。読み込みを許可する URL と、bridge で公開する機能の両方を絞ります。

たとえば`https://shop.example.jp.attacker.test/`は`https://shop.example.jp`で始まりますが、ホストは別です。文字列の`startsWith`だけで許可すると、この違いを見落とします。URL を解析してスキーム・ホスト・ポートを比べる Dart の例です。

```dart
bool isAllowed(String value) {
  final uri = Uri.tryParse(value);
  return uri != null &&
      uri.scheme == 'https' &&
      uri.host == 'shop.example.jp' &&
      uri.port == 443;
}
```

origin はスキーム・ホスト・ポートの組み合わせです。この例では`https://shop.example.jp`と同じ origin を許可し、パスは問いません。ホストが異なる URL、`http`、別ポートは WebView 内の表示を許可しません。この判定を、初回の読み込みと、その後に WebView 内で表示する遷移先に適用します。

遷移先は、たとえば次のように扱いを分けます。

| 遷移先               | 扱い                                     |
| -------------------- | ---------------------------------------- |
| 自社の許可済みページ | アプリ内に表示                           |
| 外部サイト           | 外部ブラウザーで開く、または表示を止める |
| 電話・メールリンク   | 利用者の操作に応じて OS へ渡す           |

`https://shop.example.jp`の origin を許可する設定は、次の URL でもテストします。

- `https://shop.example.jp.attacker.test/`はホストが違うので許可しない。
- `http://shop.example.jp/`はスキームが違うので許可しない。
- `https://shop.example.jp:8443/`はポートが違うので許可しない。
- リダイレクト後も、最初と同じルールで行き先を判定する。

### Webからアプリへの窓口を絞る

JavaScript bridge（ブリッジ、Android では`addJavascriptInterface`など）は、Web ページからアプリの機能を呼ぶ窓口です。共有画面を開く、閉じるなど、必要な操作だけを用意します。汎用の「任意のアプリ処理を呼ぶ」窓口は、Web 上でスクリプトが動いたときの影響を広げます。

Web ページに XSS（意図しないスクリプト実行）があると、許可したページからでもブリッジを悪用されます。ブリッジに秘密情報を渡さず、機微な操作にはユーザー確認を入れます。

#### iframeもbridgeを呼べる

iframe は、ページ内に別のページを表示する枠です。ここでは、アプリ側で`AppBridge`という bridge を登録したものとします。次の例では、スクリプトだけを許可した iframe からメッセージを送ります。

```html
<iframe
  sandbox="allow-scripts"
  srcdoc="&lt;script&gt;AppBridge.postMessage('from-iframe')&lt;/script&gt;"
></iframe>
```

Android の`addJavascriptInterface`は、iframe を含むすべてのフレームに公開されます。アプリ側では、呼び出し元の origin を検証できません。`sandbox="allow-scripts"`は iframe 内でのスクリプト実行を許可するため、この bridge への呼び出しを防ぐ設定にはなりません。

bridge の公開範囲は、OS や登録方式によって異なります。トップページと iframe の両方から bridge を呼び、アプリが受け付ける操作を確かめます。

外部の iframe を読み込む場合、Web 側は`Content-Security-Policy`の`frame-src`で読み込み元を制限します。アプリ側は bridge で許す操作を絞ります。`addJavascriptInterface`のように呼び出し元を区別できない bridge は、信頼できない iframe を含む WebView に登録しません。

<hr class="page-break"/>

## URLの制限で電話・メールリンクを止めない

前節の許可判定は、WebView に表示するページを制限するためのものです。すべてのリンクにそのまま適用すると、`tel:`や`mailto:`も HTTPS ではないため拒否され、電話やメールのアプリを開けなくなります。お問い合わせページや店舗案内では、この制限で必要な操作まで止めてしまいがちです。

電話・メールリンクは、HTML では次のように書きます。

```html
<a href="tel:+81312345678">03-1234-5678に電話</a>
<a href="mailto:example@example.com">example@example.comにメール</a>
```

URI スキームは、URL の先頭にある`https:`や`tel:`の部分です。アプリ側でリンク先を受け取ったら、WebView 内の表示を許可する判定の前に、`tel:`と`mailto:`を分岐します。利用者がリンクを押したことを確認して OS へ渡し、WebView での読み込みは止めます。`https:`のリンクにはページの許可判定を適用し、それ以外の不要なスキームは拒否します。

<hr class="page-break"/>

## WebViewとOSの更新を確認する

設計どおりに動くかを確かめるには、アプリのバージョンに加えて、OS と WebView の更新状態も確認します。同じアプリでも、端末の更新状態によって WebView の表示や動作が変わるためです。

Google Play 対応の Android では、WebView provider は Google Play から更新されます。アプリから`getCurrentWebViewPackage()`で provider の package ID とバージョンを取得できます。Chrome アプリと WebView provider は別のパッケージである場合もあるため、実際に使われている provider のバージョンを確認します。

更新を案内する場合は、実際に使われている provider の Google Play 詳細画面へ誘導します。Google アカウント未登録の端末では、先にストアへのログインを求められることがあります。テスト端末では、ストアで更新できる状態かも確認します。

iOS の WKWebView が使う WebKit は、iOS と一緒に更新されます。動作確認では iOS のバージョンを確認し、Flutter のプラグインを使う場合はそのバージョンも記録します。

## おわりに

WebView を実装するときに見落としやすい、レイアウトや URL の扱い、JavaScript bridge の注意点を紹介しました。本稿が、これから WebView を実装する方や、既存の実装を見直す方の参考になれば幸いです。

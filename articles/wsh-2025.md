---
title: Web Speed Hackathon 2025参加記
emoji: 💨
type: tech
topics:
  - frontend
  - web
  - performance
published: false
---

CyberAgentさんが開催する[Web Speed Hackathon 2025](https://cyberagent.connpass.com/event/338797/)に参加しました。意図的に重くなるよう作られているアプリケーションを改善し、Lighthouseスコアの改善を競うイベントです。

スコア上の私の順位は15位でしたが、残念ながらデグレを発生させてしまい順位対象外となりました。同様の理由で上位16人のうち15人が順位対象外となったようで、波乱の展開でした……

## タイムライン

まずは、私が行った改善を競技開始から時系列で振り返ってみます。

### 開発環境構築

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/e33559300d45a3bc9a03ec1b9d8c15242d408763

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/aca95cc9862075f298358b3d8bc69d04a20907f5

とりあえず以下の二項目を行いました。

- Dev Container導入
- GitHub ActionsでLint/Type check/Testを実行

Dev Containerは隔離された環境でClineを使うために入れています。誤って重要なファイルを削除してしまうリスクを減らせて安心できます。

### Clineでアプリケーションの概要把握

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/5f6fedb25a9a321e77ab04693b394ace36710824

デプロイ作業をやっている間に、Clineに既存ソースコードの調査を行ってもらいました。結果を少し修正して`.clinerules`に残しています。今大会の概要を知らない方は一読をおすすめします。

### 無駄なPrefetchの削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/87c76995fed6db00e2d6444b99507c7d10422501

「推測するな計測せよ」はパフォーマンス改善における格言ですが、Web Speed Hackathonで提供されるアプリケーションはまともに計測できないほど遅いです。最初はあたりをつけて改善していく必要があります。

まずは、サーバーに存在する画像をすべてPrefetchする処理が入っていたので消しました。

### SSR修正

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/405e497357ccd069a9e866df4e215d86a8c7c63a

サーバー上でSSRの結果を捨てているコードがあったので、きちんとクライアントに送るよう修正しました。うまくいけばLCPを大きく改善できるはずです。しかし実際にはこのコミットでは正しく修正できておらず、このあと何度もHydration Errorを含むSSR関連の問題にに対処することになってしまいます。

### Webpack Bundle Analyzer導入

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/7ba469156f41d71721255b65ed6c9fd9e56bb164

今回はWebpackが使われていたのでBundle Analyzerを導入しました。フロントエンドのパフォーマンス改善には必須のツールです。

### インラインソースマップの削除、Minify実施、チャンク分割

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/d96c97bf2040e8949c92938b7fb799dc57776d78

インラインソースマップが有効になっていたので、削除しました。Minifyも実施します。また、すべてのコードが`main.js`に入っていたので、チャンク分割がされるよう設定を修正しました。

### WASM版ffmpeg削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/6752d3686a36ca53e1b2851b31c0371a91378186

シークバーに表示するサムネイルを生成する処理にWASM版ffmpegを使っていたので削除しました。バンドルの中でかなりの割合を占めていました。サムネイルは事前にffmpegで生成するようにしました。

### テストのシャーディング

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/621f93ea6f9475ac7d51271c043799792efb76e9

CI上で実行しているVRTがかなり遅かったので、シャーディングを導入しました。Playwrightでのシャーディングの方法は事前に調べておいたためスムーズに導入できました。時系列でテスト結果を確認するためのダッシュボードも事前に作っておいたので導入しました。

ただ、アプリケーション自体が重いため自動テストはまともに動かず、結局競技終了までテスト結果を確認することはほとんどありませんでした。

### Webpackのキャッシュ有効化

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/9a493fcbc1d31586ebbf27c9408f7c2c0918a6dd

ビルドを高速化するためにWebpackのキャッシュを有効化しました。実際に早くなったのかはよくわかりませんでした。

### リコメンドAPIから不要なデータを削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/0ea0dff6470baf928a55b04151ad2c22c1c26ff0

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/e010706a0d5086b6a94da6ffcfab2dda4c5f157e

Chrome DevToolsのNetworkタブを見ると、リコメンドAPIから取得しているデータが非常に大きいことがわかりました。実際には使われていないデータがかなり多かったため削除しました。

### 画像のLazy Load

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/a7aaccfbe0c069d5bfdb9cc635b49e9c06340880

すべての`<img>`タグに`loading="lazy"`をつけて周りました。本当はabove the foldの画像はlazy loadするべきではありませんが、とりあえず深く考えずにすべて遅延読み込みさせています。

### 画像のAspect Ratioを指定

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/1ef91f1562530cf8e068c4599819e8ddd0414efd

CLSを改善するために画像のAspect Ratioを指定しました。この改善は目に見えてレイアウトシフトが減るので楽しくなります。

### `Cache-Control: no-store`の削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/4127a7f2475c8e158151cd5fc180ee8d2775d9f0

バックエンドのミドルウェアで`Cache-Control: no-store`を設定している箇所がありました。各エンドポイントを見て回ったところ、`no-store`が必要そうなものはなかったので削除しました。

この設定により、2回目以降の静的リソースへのリクエストには304 Not Modifiedが返ってくるようになります。

### `asset/inline`を`asset/resource`に変更

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/aecdaca3789d06225827b0bc79ef5ebc431881ff

PNG画像を`asset/inline`でData URLとして埋め込んでいる箇所があったため、別のURLから読み込む`asset/resource`に変更しました。ただ、Bundle Analyzerで確認したところPNGの割合は多く見えなかったので効果は少ないかもしれません。

### UnoCSSのランタイム削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/5579edee9905a330826b2bd33718c7c3e0b5d26a

UnoCSSというCSSエンジンが使われていました。今回初めて知ったのですが、TailwindのようなUtility CSSフレームワークに近く、柔軟にカスタマイズ可能できることが特徴のようです。

これがCSSファイルをビルド時ではなくランタイムで生成する方式になっており、更に巨大なアイコンデータをすべて読みこんでいたため、バンドルサイズのかなりの割合を占めていました。ネットワーク負荷の面でもCPU負荷の点でも問題があるため、ビルド時に生成するよう修正します。

ビルド時にCSSファイルを生成する際はソースコードを静的解析する必要があります。その際、Tailwindと同様にクラス名を動的に生成している箇所は検出できないため生成されるCSSファイルに含めることができません。以下のように修正する必要があります。

```js
// NG
className={`i-fa-solid:${isSignedIn ? 'sign-out-alt' : 'user'} m-[4px] size-[20px] shrink-0 grow-0`}

// OK
className={classNames(isSignedIn ? 'i-fa-solid:sign-out-alt' : 'i-fa-solid:user', 'm-[4px] size-[20px] shrink-0 grow-0')}
```

正規表現で修正が必要な箇所を検索したところそれほど多くなかったため、すべて手動で修正しました。

### BabelからSWCに移行

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/f3af2f50714c52341ac352681ae21acc707c1274

ビルド時間短縮のためにSWCに移行しました。Babelの設定に無駄なPolyfillを追加するものがありましたが、一旦そのまま移行しています。

### Polyfill削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/732e32617251b0466bce5089e46d417f990fcb5a

今回のテスト環境はChrome 133だったので、不要なPolyfillを削除しました。

### CSSのMinify

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/339003dbaca47565d407c3f31bbc91a1c627bf1e

`css-minimizer-webpack-plugin`を入れました。

### JavaScript/CSSのファイルをimmutableにする

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/41d630bf19e4ac34589b536b8258806fdec4d19c

WebpackでJS/CSSのチャンクファイル名にハッシュを付与するよう設定しました。これにより同じURLなら必ず同じコンテンツが返されるようになるため、ブラウザからサーバーにファイルが変更されていないか確認する必要がなくなります。`Cache-Control: public, max-age=2592000, immutable`を付与することで、ブラウザからサーバーにリクエストを送り、サーバーが`304 Not Modified`を返す動作が不要になります。

ただ、実際にはすべてのファイルをimmutableにすると何故かCSSが壊れたため、一旦revertしています。結局後述するRspack導入時にこの問題は解決したため、ツールチェインのどこかにバグがあったのだと思います。

### 動画ファイルをimmutableにする

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/b5ef4b0320a98a9cbaabd2dd2f970ac4153da5c2

動画ファイルは競技中全く変更されないためimmutableとしました。

### ESLintの設定修正

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/8c251fae4925b14dae5762ce6f4908ee8dafc82e

ESLintが`react/jsx-sort-props`の警告を大量に発しており、エラーが鬱陶しい状況でした。しかし`react/jsx-sort-props`はautofixに対応していないため、代わりに同等の機能がある`perfectionist/sort-jsx-props`を導入しました。

### Rspack導入

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/13301ba730ec10cfc0985f5a58b1212845fbe8fe

ビルド高速化のためRspackを導入しました。Watch modeでdevelopment buildが300msで終わるようになり爆速です。

なぜか`builtin:swc-loader`が動かなかったので素の`swc-loader`をつかっていますが、それでも十分早いです。

### 無駄なSleep削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/337638cbedea9db806992db2b14c99f5fbb9c9db

DevToolsのPerformanceタブを見ると、ネットワークリクエストをしているわけでもCPU負荷がかかっているわけでもないのにレンダリングが遅れている謎の時間がありました。どこかでSleepしているのではないかと考え、勘で`1000`でソースコードを検索したところ、Dynamic importを必ず1000msかかるよう遅延させているコードがあったので削除しました。

### 画像をWebPに変換

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/39e82f24234055659a39b2890e4808acf95405a6

大量の画像がJPEGで配信されていたので、WebPに変換しました。また、見逃していましたが404ページにあるGIF画像がかなり重かったようです。WebPはアニメーションにも対応しているので、同様に変換すべきでした。

後から考えるとデグレと判断されない範囲で解像度を落としておくべきだったと思います。また、WebpではなくAVIFにすべきでした。

### 無駄なSleep削除 (2)

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/aa8b3861ac7c4aa35779ccc56203c869615b54fb

Performanceタブを再度見たところまだ謎の遅延があったので再度調べたところ、Schedule Pluginと称してリクエストを1000ms遅延させるコードがありました。こちらも削除しました。

### 無駄なSleep削除 (3)

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/51573187cee2e018f18a7b0d0bd005df04bfc81e

まだ謎の遅延があったので調べたところ、リクエストをバッチで送信する処理が入っていることがわかりました。GraphQLのDataLoaderと同様の仕組みのようです。完全に削除すると大量のリクエストが送られそうだったので、窓長を1000msから16msに変更しました。

この処理にはBatshitというライブラリが使われていました。名前が酷い。

### SSR修正 (2)

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/96673c7f524fd9e094c67cd4d21360f1edd7c2bf

今年は状態管理ライブラリとしてZustandが使われていました。このストアの中身がSSR時にサーバーからクライアントに送られていなかったので、初期HTMLに含めて送るよう修正しました。データを受け取るコードの残骸が最初から残っていたので、この修正は作問者が意図したものだったのだと思います。

しかし実際には、React Routerのloaderでfetchしてストアに入れたはずのデータが、SSRやHydrationの最初のレンダリングでは取得できず、ストアが空に見えるという問題が生じていました。そのためこの変更はこの時点では無意味でした。

暫定的に、Zustandのストアの中身ではなくloaderで取得したデータを`useLoader`から取り出すようにしています。ただ、この場合は深いコンポーネントまでデータをバケツリレーする必要があり面倒です。実際、この時点では修正漏れが多くあり、サーバー側でエラーが発生してClient Renderingに切り替わってしまうページもありました。また、ZustandとReact Routerで同じデータを二重にクライアントに送信する無駄もあります。

### Cloudflare導入

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/53f3f763b0ec21db8e348949b338aa78528f01f4

Cloudflareを導入しました。ImmutableなデータをCDN上でキャッシュしてくれる利点と、zstdによる圧縮やHTTP/3による多重化を自動で行ってくれる利点があります。

また、Cloudflareとオリジン間の帯域を節約するため、nginxを導入しgzipで圧縮しました。`@fastify/compress`で圧縮するのが作問者の想定だったようですが、エラーが発生したため諦めてnginxを入れました。GitHubのIssueで2週間前に同じ症状の報告があったのですが、原因は現時点で不明なようです。

https://github.com/fastify/fastify-compress/issues/350

### ReDoS修正

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/de17dc46e3a55215997559f0f1a14dba1457afd5

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/1ca3db5adece9a28081302cad5553211c99e7e0b

ReDoSを修正しました。Web Speed Hackathonでは毎年恒例で発生しているようなので、`eslint-plugin-regexp`を入れてみたら検出してくれました。

問題があったのはメールアドレスとパスワードのバリデーションでした。メールアドレスのほうは正規表現だけで対処するのが難しそうだったので、他の方法と組み合わせて当初と同様のバリデーションを実装しています。

### データベースにインデックス追加

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/7532104a054727a15620307855469924a816647c

APIレスポンスが遅かったため、データベースにインデックスを追加しました。今回はSQLite (LibSQL)が使われていました。本来はNode.jsのプロファイリングを使って遅い原因を調べるべきですが、時間が限られた中でコスパが悪そうだったので今回は見送っています。

### JavaScriptで行っている処理をCSSで再実装

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/4285a565d780d8ad66a65a4f11dddbc06f9050fa

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/8ad35d3e77eb29199bdb261235c331601a8581ef

いくつかCSSで実装できる機能をJavaScriptで実装していたため、CSSで再実装しました。マウスホバー時にスタイルを変更する処理と、ウィンドウサイズが変わった際に要素のアスペクト比を保つようリサイズする処理です。JavaScriptでは250msごとにポインタの位置を取得して処理を行っており、非常に筋が悪いです。

### 再レンダリング削減

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/92371aec3a08f738f1b3ffcb6b6687e93b1b25dc

`scroll`イベントが発生するたびに`setState`でスクロール位置を保存している処理がありました。一定以上スクロールした際にヘッダーを透過させるために使われています。頻繁に再レンダリングが行われるため重そうです。実際にはスクロール位置が一定の閾値を前後した際にのみ再レンダリングすればいいので、そのように修正しました。最近CSSでスクロールに応じてアニメーションできるAPIが追加されていたようですが、もしかしてこれもCSSだけで実装できたのでしょうか？

### APIレスポンスをImmutableにする

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/673c5477859dc354f153d11ce563fbd0e7183247

今回はユーザ登録以外でデータベースを変更することがなく、ユーザごとに表示を出し分けることもなかったため、ほとんどのAPIエンドポイントに`Cache-Control: public, max-age=2592000, immutable`を設定することができました。実際のプロダクトではこんなことは難しいでしょうね……

### SSRに`ReactDOMServer.renderToPipeableStream()`を使う

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/5c5bfd1db3375b0f062b56354bb564c4565da357

初期実装ではSSRに`ReactDOMServer.renderToString()`を使っていたのですが、エラーが出ていることに気づいて`ReactDOMServer.renderToPipeableStream()`を使うようにしました。`<Suspense>`を使っていたことが原因だったようです。後からわかったのですが、実際にはsuspendしている箇所はないので`<Suspense>`を消してしまってもよかったようです。

### 動画プレイヤーライブラリのチャンク分割

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/631fb5bccb1a23a08867213852bfe41cc535488a

このアプリケーションは動画再生にHLSを使っています。Safari以外のブラウザはHLSに対応していないため、再生のために何らかのライブラリを入れる必要があります。初期実装ではページによってVideo.js, HLS.js, Shaka Playerの3つのライブラリを使い分けているカオスな状況でした。これらはいずれもサイズがかなり大きく、バンドルサイズのかなりの割合を占めていました。

まずはじめの一歩として、それぞれをDynamic Importすることでチャンク分割を行いました。

### JavaScriptで行っている処理をCSSで再実装 (2)

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/cd569e1126c9e77a9deb785e869bda7b16743cf5

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/8aa227cce2c2eecffb6cbde1eb4db3bf22534647

JavaScriptで行っているがCSSで実装できる処理がまだまだ残っています。テキストが一定の行数を超える際に省略記号を表示する処理と、カルーセルのスクロールスナップがJavaScriptで実装されていたのでCSSで実装しました。

### SSR修正 (3)

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/6e1b6600ff1840e5568e74d1600e21a8bd3939db

`useSyncExternalStore`の第3引数が省略されておりSSRが失敗している箇所があったので修正しました。

### Luxonをdate-fnsに移行（失敗）

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/62e91c253e5ecc53005e10371d45990ee36508db

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/9602be203d1357d04234f9e86dea64f87d495f49

Luxonはバンドルサイズを増加させてしまうライブラリとして有名です。今回もバンドルの中である程度の割合を占めていました。

そこでtree shakingが可能なdate-fnsに移行しようとしたのですが、結局変更点が多くデグレが怖かったので途中で諦めました。特にタイムゾーン周りの処理が怖いです。実際、上位陣の中でも番組表画面が9時間ずれるデグレを起こしている方が数名いました。

### TypeScriptのProject Reference導入（失敗）

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/910060a0a24c37b433f11fa06e0c8bc123fd757f

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/e90f50edcfc0366a2a531736484595c204c89b76

型チェックに時間がかかっているのが気になったためTypeScriptのProject Reference導入を試みましたが、解決できないコンパイルエラーが発生してしまったため諦めました。後から考えると、ここに1.5時間近くの時間を割いてしまったのが悔やまれます。

### 動画をMP4に変換

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/35c6e48b4dee4bbfbb9a0a358e61860236391744

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/6f714853c1b701b65f7cf3ea7b3fb5b431d07db5

初期実装では動画をHLSで配信していましたが、先述したようにHLSの再生には外部ライブラリを使う必要があります。これはバンドルサイズを増加させてしまうため、ライブラリなしで再生できるMP4に変換して配信するようにしました。これで各ページのリコメンドセクションで使っていたShaka Playerと、見逃し視聴ページなどで使っていたHLS.jsを消すことができます。ライブ配信では引き続きHLS/Video.jsを使います。

この変更のお陰でバンドルサイズは大幅に減ったのですが、残念ながらHLS.jsを消す際にバグを埋め込んでしまい、デグレの原因となってしまいました。

### トップページのBelow the foldの動画を遅延再生

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/c9ac0c84304e14a733cefccadd849606252731d1

トップページにはリコメンドで動画を自動再生している箇所が3箇所あります。このうち2箇所はスクロールしないと表示されないため、`IntersectionObserver`をつかって遅延再生するようにしました。

### Zodを削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/c4fcfc18fe5a443a960141ecf515086d64560faf

ログイン画面のバリデーションにZodを使っていましたが、バリデーションの内容が簡単だったため、Zodを削除して手動でバリデーションするようにしました。Zodもバンドルサイズを増加させてしまうライブラリとして有名です。

### APIスキーマをクライアントJSから削除

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/846db87140ae9a3fb564487bfc7270b2fe73409d

バンドルの中でAPIスキーマの割合が大きかったため、削除しました。クライアントサイドでAPIリクエストやレスポンスを検証する必要性は低いです。

スキーマによりリクエストやレスポンスの型が付与されるメリットもありますが、これはファイルをうまく分割することで最終的なバンドルサイズを増やすことなく実現できます。

### lodashからlodash-esに移行

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/761564b8f80e9e2772efea4780721fc2b31e45e1

lodashもバンドルサイズを増加させてしまうライブラリとして有名です。lodash-esはtree shakingが可能なためそちらに移行しました。

### クライアントサイドでAPIリクエストを発行しない

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/92db0023d3133cf8ea8a61d611251e10d1c5ffe6

初期実装では、サーバーサイドでのSSR時とクライアントサイドでのHydration時であわせて2回同じAPIリクエストを発行していました。しかし、サーバーサイドで実行したAPIリクエストの結果はZustandのストアに格納しており、その中身はクライアントサイドに送っているため、クライアントサイドでAPIリクエストを発行する必要はないはずです。そこで、ストアに既にデータが入っている場合はそれを利用するよう修正しました。

ただ、この時点の実装ではストアに入れたはずのデータがSSR時や初回のHydration時に取得できない問題があったため、実際には意味がありませんでした。

### APIリクエストの並列化

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/c055283679221aa704a7fc1164c4a4fc97c27329

初期実装ではAPIリクエストを順番に発行していました。これを並列化することで、リクエストの待ち時間を削減しました。

### SSR修正 (4)

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/145974be30f161d0c817df41b07b58a3da2639e3

初期実装では、React RouterのloaderでAPIからデータを取得し、その結果をZustandのストアに入れていました。各コンポーネント内でストアからデータを取り出し、画面に表示するという流れになっていました。

しかし、実際にはSSR時や初回のHydration時にはにストアの中身は空になってしまう問題が起きていました。これまで暫定的にloaderの返り値を使うという対処を行っていましたが、対処漏れでエラーが発生しているページもありました。この問題の原因は結局判然としなかったのですが、おそらくReactのライフサイクルの問題で、React Routerのloaderでストアに設定したデータがコンポーネントで確認できるようになるまでタイムラグがあるのだと思います。

解決策として、SSR時と初回Hydration時には`store.getState()`で取得したデータを使うよう修正しました。ZustandのSSRの情報はインターネット上にほとんどなく、この問題の対処には苦労しました。作問者はここまで想定していたのでしょうか？それとも私が変な遠回りをしてしまっているのでしょうか……？

また、これでReact Routerのloaderで取得したデータをクライアントに送る必要がなくなったため、送らないよう修正しました。これで初期HTMLのサイズが大幅に小さくなります。

### Zustandのセレクタ最適化

https://github.com/ciffelia/web-speed-hackathon-2025-private/commit/8ff486a1ad90c8178faffa46c660c2bc5b947da7

Zustandの`useStore`を使ってストアのデータを取得する際、セレクタで取得範囲を絞らずストア全体を取得している箇所がほとんどだったため、セレクタを使って必要なデータだけを取得するよう修正しました。これにより、ストアのデータが変更された際に再レンダリングされるコンポーネントの数を減らすことができます。

こういった単純作業はClineにやらせたかったのですが、指示が悪いのかうまく動かず、結局手動で修正することになって骨が折れました。

## やり残したこと

ここまでが競技時間中に行った変更です。時間が足りずやり残した変更として以下の2点が挙げられます。

- CSSで実装できる処理をJavaScriptで実装している箇所の修正
  - 移植に時間がかかりそうで後回しにしていた箇所がまだ数か所残っています。
  - これが原因で250msごとに再レンダリングが起きているので、CPUが制限されているLighthouseではかなり不利な気がしています。
- ライブ配信の最適化
  - HLS.jsを使っている部分はほとんど触れませんでした。m3u8ファイルの中に`X-AREMA-INTERNAL`という謎の項目があるのが気になっていたのですが、後から聞いたところによるとここに巨大な不要データが入っていたようです。

振り返ってみるとかなり多くのことをやったのですが、その割にあまりスコアは伸びなかった印象です。私のスコアが300点だったところ、上位陣には500点～700点を獲得している方がいました。何をしていたのか気になります。

## まとめ

## おまけ：Clineとの会話

最近Clineを使い始めて、ハッカソン中もかなり活用できたので履歴の一部を載せておきます。コードを書かせることはうまくできなかったのですが、既存の実装を調査するのに役立ちました。簡単な指示を出すだけで自律的に多くのファイルを読んでまとめてくれるのが便利です。モデルは3.7 Sonnetを使っており、ハッカソン期間中にかかったAPI料金は400円ほどでした。ちょっと高いですね。

使ってみた感想として、Clineは既存コードの意図を尊重して問題点の指摘はしてくれない印象です。調査して報告しろとだけ伝えると、既存実装のメリットは何点か挙げてくれるものの、問題点には全く触れません。初めから「問題点を見つけたら報告に含めてください」とプロンプトに書くべきだったかもしれません。

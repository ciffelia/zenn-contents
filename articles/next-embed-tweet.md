---
title: 'Next.jsでツイートを埋め込む'
emoji: '🐦'
type: 'tech'
topics:
  - 'nextjs'
  - 'react'
  - 'typescript'
  - 'javascript'
  - 'twitter'
published: true
---

Next.jsでTwitter上の投稿を埋め込む方法を解説します。

# 手順

ページコンポーネントで`next/script`を使って`widgets.js`を読み込み、ツイートを表示したいコンポーネントで`widgets.js`が認識してくれる埋め込みコードを挿入すればOKです。

```tsx:pages/index.tsx
import React from 'react';
import Script from 'next/script';
import { Tweet } from '../components/Tweet';

const HomePage: React.FC = () => (
  <>
    <Tweet id="20" />
    <Script
      src="https://platform.twitter.com/widgets.js"
      strategy="lazyOnload"
    />
  </>
);

export default HomePage;
```

```tsx:components/Tweet.tsx
import React, { useEffect, useRef } from 'react';

export const Tweet: React.FC<{ id: string }> = ({ id }) => {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // @ts-expect-error
    window.twttr?.widgets.load(ref.current);
  }, [id]);

  return (
    <div
      dangerouslySetInnerHTML={{ __html: generateEmbedHtml(id) }}
      ref={ref}
    />
  );
};

const generateEmbedHtml = (id: string): string => {
  if (!/^\d+$/u.test(id)) {
    throw new Error(`Invalid tweet ID: ${id}`);
  }

  return `<blockquote class="twitter-tweet"><a href="https://twitter.com/i/status/${id}"></a></blockquote>`;
};
```

# 実装上の注意点

実装にあたり注意が必要な罠が2つあります。

:::message
上記のコードではいずれの問題も回避しています。
:::

## 1. `dangerouslySetInnerHTML`を使う必要がある

`widgets.js`はDOMツリー上で`.twitter-tweet`の要素を書き換えてしまうので、それらの要素は`dangerouslySetInnerHTML`で挿入しReactの管理下から外す必要があります。

```tsx
// ⭕ OK
<div
  dangerouslySetInnerHTML={{ __html: `...` }}
  ref={ref}
/>

// ❌ NG
<div ref={ref}>
  <blockquote className="twitter-tweet">
    <a href={`https://twitter.com/i/status/${id}`} />
  </blockquote>
</div>
```

詳細は以下のブログで解説されています。こちらのブログでは`<blockquote>`を`<div>`で囲うことで解決していますが、`dangerouslySetInnerHTML`を使うのがより安全だと考えられます。

https://penpen-dev.com/blog/react-tweet-error/

## 2. `twttr.widgets.load()`を呼ぶ必要がある

`widgets.js`が読み込まれる前に`<Tweet>`がレンダリングされる場合は、何もせずとも問題なくツイートが表示されます。一方で`widgets.js`が読み込まれた後に`<Tweet>`がレンダリングされる場合は、`twttr.widgets.load()`を呼んでツイート内容を取得、描画する必要があります。

上記のコードでは`useEffect`において、`window.twttr`が存在する場合は`twttr.widgets.load()`を呼ぶようにしています。

詳細はTwitterのドキュメントに記載されています。

https://developer.twitter.com/en/docs/twitter-for-websites/javascript-api/guides/scripting-loading-and-initialization

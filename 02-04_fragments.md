# フラグメント

<https://relay.dev/docs/tutorial/fragments-1/>

- [フラグメント](#フラグメント)
  - [ステップ1 - フラグメントの定義](#ステップ1---フラグメントの定義)
  - [ステップ2 - フラグメントを展開する](#ステップ2---フラグメントを展開する)
  - [ステップ3 - useFragmentの呼び出し](#ステップ3---usefragmentの呼び出し)
  - [ステップ4 - フラグメントを参照するTypeScriptの型](#ステップ4---フラグメントを参照するtypescriptの型)
  - [演習](#演習)
  - [複数の場所でフラグメントを再利用する](#複数の場所でフラグメントを再利用する)
    - [ステップ1 - フラグメントの定義](#ステップ1---フラグメントの定義-1)
    - [ステップ2 - フラグメントを展開する](#ステップ2---フラグメントを展開する-1)
    - [ステップ3 - useFragmentの呼び出し](#ステップ3---usefragmentの呼び出し-1)
    - [ステップ4 - 一度修正して、どこでも楽しむ](#ステップ4---一度修正してどこでも楽しむ)
  - [フラグメント引数とフィールド引数](#フラグメント引数とフィールド引数)
    - [Deep Dive: GraphQLディレクティブ](#deep-dive-graphqlディレクティブ)
    - [ステップ2](#ステップ2)
  - [まとめ](#まとめ)

> フラグメントは、日本語で「断片」を意味する。

フラグメントは、Relayの機能を区別する1つです。
フラグメントは、それぞれのコンポーネントが独立してそれ自身が必要とするデータを定義できる一方で、単一のクエリで効率的に保持します。
このセクションでは、クエリを分割してフラグメントにする方法を紹介します。

---

始めるために、Storyコンポーネントにストーリーーをポストした日を表示したいと考えているとします。
それをするために、サーバーからいくつのデータが必要なため、クエリにフィールドを追加しなければなりません。

`Newsfeed.tsx`を開き、新しいフィールドを追加するために`NewsfeedQuery`を見つけます。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    topStory {
      title
      summary
      createdAt // この行を追加
      poster {
        name
        profilePicture {
          url
        }
      }
      image {
        url
      }
    }
  }
`;
```

`Story.tsx`を開き、日付を表示するようにそれを修正します。

```tsx
import Timestamp from './Timestamp';  // この行を追加

type Props = {
  story: {
    createdAt: string; // この行を追加
    ...
  };
};

export default function Story({story}: Props) {
  return (
    <Card>
      <PosterByline poster={story.poster} />
      <Heading>{story.title}</Heading>
      <Timestamp time={story.createdAt} /> // この行を追加
      <Image image={story.image} />
      <StorySummary summary={story.summary} />
    </Card>
  );
}
```

現在、日付が表示されるべきです。
そして、GraphQLに感謝して、新しいサーバーコードを記述またはデプロイする必要はありません。

しかし、それについて考える場合、なぜ`Newsfeed.tsx`を修正しなければならなかったのでしょうか？
Reactコンポーネントは、自己完結型であるべきではないでしょうか？
なぜ、NewsfeedはStoryによって要求される特定のデータについて注意しなければならないのでしょうか？
データがStoryの階層の下のいくつかの子コンポーネントに要求される場合、何が起こるのでしょうか？
それが、多くの異なる場所で使用されるコンポーネントである場合、何が起こるのでしょうか？
そして、そのデータ要求が変更されたらいつでも、多くのコンポーネントを修正しなければなりません。

これらと他の多くの問題を避けるために、Storyコンポーネントのデータ要求を、`Story.tsx`に移動できます。

`Story`のデータ要求を`Story.tsx`内の*フラグメント*に定義して分割することで、これを行います。
フラグメントは、Relayコンパイラが完全なクエリにまとめ上げる、GraphQLの分離された部分です。
それらは、ランタイムでそれぞれのコンポーネントがそれ自身のクエリを実行するランタイムコストを払うことなしに、それぞれのコンポーネントをそれ自身のデータ要求を定義するようにします。

![fragments](https://relay.dev/assets/images/fragments-newsfeed-story-compilation-5988239417a9739a88f25bfcad3a7ab7.png)

先に進んで、`Story`のデータ要求をフラグメントに分けましょう。

---

## ステップ1 - フラグメントの定義

`src/components`にある`Story.tsx`の`Story`コンポーネントの上に、次を追加してください。

```tsx
import { graphql } from 'relay-runtime';

const StoryFragment = graphql`
  fragment StoryFragment on Story {
    title
    summary
    createdAt
    poster {
      name
      profilePicture {
        url
      }
    }
    thumbnail {
      url
    }
  }
`;
```

クエリ内の`topStory`内からすべての選択を取得して、この新しいフラグメントの宣言にそれらをコピーしたことに注意してください。
クエリのように、フラグメントは`StoryFragment`のような名前を持ち、これをすぐに使用しますが、それらは"on"しているため、それらはGraphQL型の`Story`も持っています。
これは、グラフにStoryノードを持っているときはいつでも、このフラグメントを使用できることを意味します。

## ステップ2 - フラグメントを展開する

`Newsfeed.tsx`を開き、次の通り`NewsfeedQuery`を修正します。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    topStory {
      ...StoryFragment
    }
  }
`;
```

`topStory`内部の選択を`StoryFragment`に置き換えます。
Relayコンパイラは、`Newsfeed`の変更なしで、現時点からすべてのStoryのデータを属できることを保証します。

## ステップ3 - useFragmentの呼び出し

現在、Storyが空のカードをレンダリングすることに気付きます。
すべてのデータがありません。
Relayは、`useLazyLoadQuery()`から取得した`story`オブジェクト内のフラグメントによって選択されたフィールドを含めることをサポートしていないのでしょうか？

この理由は、Relayがそれらを隠すためです。
コンポーネントが、あるフラグメントのデータを特に問い合わせしない限り、データはコンポーネントに見えません。
これは*データマスキング*と呼ばれ、コンポーネントが別のコンポーネントのデータ依存に暗黙的に依存せず、すべての依存関係を独自のフラグメント内で宣言するように強制します。
これはコンポーネントを自己完結型にして保守性を高め続けます。

データマスキングなしで、どこか他のコンポーネントがそのフィールドを使用していないか検証することは困難になるため、フラグメントからは決してフィールドを削除できません。

フラグメントによって選択されたデータへのアクセスは、`useFragment`と呼ばれるフックを使用します。
次のように`Story`を修正してください。

```tsx
import { useFragment } from 'react-relay';

export default function Story({story}: Props) {
  const data = useFragment(
    StoryFragment,
    story,
  );
  return (
    <Card>
      <Heading>{data.title}</Heading>
      <PosterByline poster={data.poster} />
      <Timestamp time={data.createdAt} />
      <Image image={data.image} />
      <StorySummary summary={data.summary} />
    </Card>
  );
}
```

`useFragment`は2つの引数を受け取ります。

- 読み込みたいフラグメントのGraphQLタグ付けリテラル
- 前に使用した同じストーリーオブジェクトで、フラグメントを展開したGraphQLクエリ内の場所から取得されます。これを*フラグメントキー*と呼びます。

それはそのフラグメントによって選択されたデータを返します。

> **💡 TIP**
> ここのJSXのすべての`story`を`useFragment`によって返された`data`に再書き込みしました。コンポーネントで同じことをしたか確認してください。
> そうでなければ機能しません。

フラグメントキーは、GraphQLクエリのレスポンス内でフラグメントが広げた場所です。
例えば、Newsfeedクエリを与えると・・・

```graphql
query NewsfeedQuery {
  topStory {
    ...StoryFragment
  }
}
```

そして、もし`queryResult`が`useLazyLoadQuery`によって返されたオブジェクトの場合、`queryResult.topStory`は｀StoryFragment`のフラグメントキーになります。

技術的に、`queryResult.topStory`は、それが必要とするデータを探す場所をRelayの`useFragment`に伝えるいくつかの隠されたフィールドを含んだオブジェクトです。
フラグメントキーはどのノードから読み込むか（ここではちょうど1つのストーリーしかありませんが、すぐに複数のストーリーを持つようになります）と、どのフィールドを読み出すことができるか（特定のフラグメントによって選択されたフィールド）の両方を指定しています。
`useFragment`フックは、Relayのローカルデータストアから特定な情報を読み込みます。

> **ℹ 注意事項**
> 後の方の例で確認するように、クエリ内の同じ場所に複数のフラグメントを広げたり、フラグメントが広げたフィールドを直接選択したフィールドと混ぜることもできます。

## ステップ4 - フラグメントを参照するTypeScriptの型

フラグメンテーションを完了するために、TypeScriptは、このコンポーネントが生データの代わりにフラグメントキーを受け取ることを期待しているため、`Props`の型定義を変更する必要もあります。

フラグメントをクエリまたは他のフラグメント内に広げたとき、クエリ結果の部分は、そのフラグメントの*フラグメントキー*になるフラグメントを広げた場所に対応することを思い出してください。
これは、フラグメントから読み取るためにグラフ内の特定の場所をコンポーネントに与えるために、propsでコンポーネントに渡すオブジェクトです。

これを型安全にするために、Relayはその特定のフラグメントのために、そのフラグメントキーを表現する型を生成します。
これにより、クエリ内でフラグメントを展開することなしに、コンポーネントの使用を試みる場合、型システムを満たすフラグメントキーを提供できません。
ここに、する必要のある変更を示します。

```tsx
import type {StoryFragment$key} from './__generated__/StoryFragment.graphql';

type Props = {
  story: StoryFragment$key;
};
```

これをすることで、`Story`が要求するデータが何かを心配する必要がなくなり、それ自身のクエリ内で事前にそのデータを取得できる`Newsfeed`を持ちます。

---

## 演習

`PosterByline`コンポーネントは、ポスターの名前とプロファイル写真をレンダリングするために`Story`によって使用されるコンポーネントです。
これらとオン剤ステップを使用して、`PosterByline`をフラグメント化してください。
次をする必要があります。

- `Actor`から派生した`PosterBylineFragment`を宣言して、それに必要なフィールド(`name`, `profilePicture`)を指定してください。
  `Actor`型はストーリーをポストできる人または組織を表現します。
- `StoryFragment`内の`poster`内に、そのフラグメントを広げてください。
- データを取得するために`useFragment`を呼び出してください。
- `poster`プロップとして`PosterBylineFragment$key`を受け取るためにプロップスを更新してください。

これらのステップを2回通してすることは、指を動かしてフラグメントを使用したメカニズムを把握するために価値があります。
ここには、正しい方法で一緒にはめ込むために必要な多くの部品が多くあります。

一度、それができたら、フラグメントがアプリをスケールするために支援する方法の基本的な例を確認してください。

---

## 複数の場所でフラグメントを再利用する

フラグメントは、特定の型のグラフのノードを与えれば、そのノードからどのデータを読むかを示しています。
フラグメントキーは、データがグラフ内のどのノードから選択されたかを指定します。
フラグメントを指定する再利用可能なコンポーネントは、異なるフラグメントキーを渡すことで、異なるコンテキスト内の異なる部分からデータを取得できます。

例えば、`Image`コンポーネントは2つの場所で使用されていることに注意してください。
ストーリーのサムネイルイメージは直接`Story`内にあり、またポスターのプロファイルピクチャーもまた`PosterByline`内にあります。
`Image`をフラグメント化して、そのフラグメントが使用される場所に従って、そのフラグメントがグラフ内の異なる場所から必要とするデータを選択する方法を確認しましょう。

![reuse fragments](https://relay.dev/assets/images/fragments-image-two-places-compiled-088126acb35aa6029bae65621118fc69.png)

### ステップ1 - フラグメントの定義

`Image.tsx`を開いて、フラグメントの定義を追加します。

```tsx
import { graphql } from 'relay-runtime';

const ImageFragment = graphql`
  fragment ImageFragment on Image {
    url
  }
`;
```

### ステップ2 - フラグメントを展開する

`StoryFragment`と`PosterBylineFragment`に戻り、`ImageFragment`を`Image`コンポーネントがデータを使用しているそれぞれの場所に展開します。

- `Story.tsx`

```tsx
const StoryFragment = graphql`
  fragment StoryFragment on Story {
    title
    summary
    postedAt
    poster {
      ...PosterBylineFragment
    }
    thumbnail {
      ...ImageFragment
    }
  }
`;
```

- `PosterByline.tsx`

```tsx
const PosterBylineFragment = graphql`
  fragment PosterBylineFragment on Actor {
    name
    profilePicture {
      ...ImageFragment
    }
  }
`;
```

### ステップ3 - useFragmentの呼び出し

イメージのフラグメントを使用して、そのフィールドを読み込むために、`Image`コンポーネントを修正して、フラグメントキーを受け付けるためにそのプロップスも修正してください。

- `Image.tsx`

```tsx
import * as React from "react";
import { graphql } from "relay-runtime";
import type { ImageFragment$key } from "./__generated__/ImageFragment.graphql";
import { useFragment } from "react-relay";

const ImageFragment = graphql`
  fragment ImageFragment on Image {
    url
  }
`;

type Props = {
  image: ImageFragment$key;
  width?: number;
  height?: number;
  className?: string;
};

export default function Image({
  image,
  width,
  height,
  className,
}: Props): React.ReactElement {
  const data = useFragment(ImageFragment, image);
  return (
    <img
      key={data.url}
      src={data.url}
      width={width}
      height={height}
      className={className}
    />
  );
}
```

### ステップ4 - 一度修正して、どこでも楽しむ

イメージのデータ要求をフラグメント化して、コンポーネント内で同じ場所に配置したため、それを使用するコンポーネントの何かを修正することなく、イメージに依存するデータを追加できます。

例えば、アクセシビリティのために、`Image`コンポーネントに`altText`ラベルを追加しましょう。

次の通り`ImageFragment`を編集してください。

```tsx
const ImageFragment = graphql`
  fragment ImageFragment on Image {
    url
    altText
  }
`;
```

現在、`Story`、`Newsfeed`または他のコンポーネントを編集することなく、クエリ内にあるすべてのイメージが、それらのために取得されたaltテキストを持ちます。
よって、新しいフィールドを使用するために、単に`Image`を修正する必要があります。

```tsx
function Image({image}) {
  // ...
  <img
    alt={data.altText}
  //...
}
```

現在、ストーリーのサムネイル画像とポスターのプロファイル画像の両方が、altテキストを持ちます。
これを確認するために、ブラウザの要素検査ツールを使用できます。

コードベースがより大きくなった場合、これがどれほど有益か想像できます。
それぞれのコンポーネントは自己完結型で、どれほどそれが多くの場所で使用されていても問題ありません。
もし、コンポーネントが100箇所で使用されたとしても、そのデータ依存のフィールドを追加または削除できます。
これは、Relayがアプリのサイズをスケールすることを支援する方法の1つです。

![fragment in multiple places](https://relay.dev/assets/images/fragment-image-add-once-compiled-addfb548d0a7422c83d492321e189d59.png)

フラグメントは、Relayアプリの構成要素です。
そのため、Relay機能の多くがフラグメントを基礎にしています。
次のセクションで、それらのいくつかを確認します。

---

## フラグメント引数とフィールド引数

現在、より小さなサイズで表示されるにも関わらず、`Image`コンポーネントはそれらのフルサイズのイメージを取得します。
これは非効率です。
`Image`コンポーネントはイメージを表示するサイズを示すプロップスを受け取るため、`Image`を使用するコンポーネントによって、それを制御されます。
同様な方法で、`Image`を使用するコンポーネントを、そのフラグメント内で取得するイメージのサイズを指定したいと考えています。

GraphQLフィールドは、リクエストを満たすために追加的な情報をサーバーに与える*引数*を受け取れます。
例えば、`Image`型の`url`フィールドは、サーバーがURLに組み込む`height`と`width`引数を受け取ります。
もし、次のフラグメントを持っている場合・・・

```graphql
fragment Example1 on Image {
  url
}
```

`images/abcde.jpeg`のようなURLをGETリクエストするかもしれません。

一方、次のようなフラグメントを持っている場合・・・

```graphql
fragment Example2 on Image {
  url(height: 100, width: 100)
}
```

`images/abcd.jpeg?height=100&width=100`のようなURLをGETリクエストするかもしれません。

もちろん、単に`ImageFragment`内に特定なサイズをハードコードしたくないと考えているため、`Image`コンポーネントが異なるコンテンツ内で異なるサイズを取得したいと考えています。
それをするために、`ImageFragment`が、親コンポーネントがどの大きさのイメージが取得されるべきかを指定できるように、*フラグメント引数*を受け付けるようにできます。
これら*フラグメント引数*は、この場合は`url`ですが、*フィールド引数*として、特定のフィールドに渡すことができます。

それをするために、次の通り`ImageFragment`を編集します。

```tsx
const ImageFragment = graphql`
  fragment ImageFragment on Image
    @argumentDefinitions(
      // `width`はフラグメント引数
      width: {              // ここから
        type: "Int",
        defaultValue: null
      }                     // ここまで
      height: {
        type: "Int",
        defaultValue: null
      }
    )
  {
    // `width`は`url`フィールドのフィールド引数
    // `$width`は上記`width`フラグメント引数によって作成された変数
    url(
      width: $width,      // ここを追加
      height: $height     // 個々を追加
    )
    altText
  }
`;
```

これを詳細に確認しましょう。

- フラグメントを宣言したところに、`@argumentDefinitions`ディレクティブを追加しました。
  これは、どの奥な引数をフラグメントが受け付けるか指定します。
  それぞれの引数について、次を与えます。
  - 引数の名前
  - [GraphQLのスカラー型](https://graphql.org/learn/schema/#scalar-types)になり得るその型
  - オプションのデフォルト値で、この場合、デフォルト値はnullで、画像を固有のサイズで取得します。
    もし、デフォルト値が与えられていない場合、そのフラグメントが使用されるすべての場所でその引数は必須です。
- 変数としてフラグメント引数を使用することで、GraphQLフィールドへの引数を追加します。
  ここで、フィールド引数とフラグメント引数は同じ名前を持ちますが、`width`はフィールド引数で、`$width`はフラグメント引数によって作成された変数であることに注意してください。

これで、フラグメントは選択するフィールドの1つを介して、サーバーに渡す引数を受け取ります。

### Deep Dive: GraphQLディレクティブ

フラグメント引数のための構文は、むしろぎこちなく見えるかもしれません。
これは、GraphQL言語を拡張するためのシステムである*ディレクティブ*に基づいているからです。
GraphQLにおいて、`@`で始まる任意のシンボルはディレクティブです。
それらはの意味はGraphQL仕様によって定義されてませんが、特定のクライアントまたはサーバーの実装で一致します。

Relayはその機能をサポートするために[様々なディレクティブ](https://relay.dev/docs/api-reference/graphql-and-directives/)を定義していて、フラグメント引数はその1つです。
これらのディレクティブはサーバーに送信されませんが、ビルド時にRelayコンパイラに命令を与えます。

GraphQL仕様は、実際に3つのディレクティブの意味を定義しています。

- `@deprecated`はスキーマ定義で使用され、廃止されたフィールドとしてマークします。
- `@include`と`@skip`は条件付きでフィールドを含めるマークとして使用されます。

これら以外に、GraphQLサーバーはスキーマの一部として追加的なディレクティブを指定できます。
そしてRelayはそれ独自のビルド時のディレクティブを持ち、それはその文法を変更することなしに、少し言語を拡張させます。

### ステップ2

これで、`Image`を使用した異なるフラグメントは、適切なサイズをそれぞれのイメージに渡すことができます。

- `Story.tsx`

```tsx
const StoryFragment = graphql`
  fragment StoryFragment on Story {
    title
    summary
    postedAt
    poster {
      ...PosterBylineFragment
    }
    image {
      ...ImageFragment @arguments(width: 400) // ここを変更
    }
  }
`;
```

- `PosterByline.tsx`

```tsx
const PosterBylineFragment = graphql`
  fragment PosterBylineFragment on Actor {
    name
    profilePicture {
      ...ImageFragment @arguments(width: 60, height: 60) // ここを変更
    }
  }
`;
```

これで、アプリがダウンロードしたイメージを確認した場合、より小さなサイズで、ネットワークバンド幅を節約することを確認できます。
フラグメント引数の値に整数リテラルを使用しているにも関わらず、ランタイムで提供された変数を使用することもできるため、後のセクションで確認します。

例えば`url(height: 100)`のようなフィールド引数は、GraphQLそれ自身の機能である一方で、`@argumentDefinitions`と`@arguments`のようなフラグメント引数はRelay特有の機能です。
Relayコンパイラは、フラグメントをクエリを結合するときに、これらのフラグメント引数を処理します。

---

## まとめ

フラグメントは、RelayがGraphQLを使用する最も特徴的な側面です。
単なるタイポグラフィや書式設定コンポーネントでなく、データを表示してデータの意味に注意を払うすべてのコンポーネントが、そのデータ依存を宣言するGraphQLフラグメントを使用することを推奨します。

- フラグメントはスケールを支援します。
  どれくらい多くの場所でコンポーネントが使用されているかに関係なく、一箇所でそのデータ依存を更新できます。
- フラグメントのデータは`useFragment`似よって読み出されなければなりません。
- `useFragment`は、読み込むグラフ内の場所を指定する*フラグメントキー*を受け取ります。
- フラグメントキーは、フラグメントが展開されたGraphQLレスポンス内から取得されます。
- フラグメントは、それらが展開する場所で使用される引数を定義できます。
  これは、フラグメントが使用されるそれぞれの状況でそれらを適合させます。

クエリ全体で再取得することなしで、単一のフラグメントのコンテンツを再取得する方法など、フラグメントの多くの他の機能を再訪問します。
ただし、最初に、配列について学ぶことで、ニュースフィードアプリをよりニュースフィードらしくしましょう。

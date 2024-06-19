# コネクションとページ分け

<https://relay.dev/docs/tutorial/connections-pagination/>

- [コネクションとページ分け](#コネクションとページ分け)
  - ["さらにコメントを読み込み"を実装](#さらにコメントを読み込みを実装)
    - [フラグメントに引数を付ける](#フラグメントに引数を付ける)
    - [usePaginationFragmentフック](#usepaginationfragmentフック)
    - [useTransitionを使用した読み込み体験の改善](#usetransitionを使用した読み込み体験の改善)
  - [ニュースフィードストーリーの無限スクロール](#ニュースフィードストーリーの無限スクロール)
    - [ステップ1 - クエリのコネクションフィールドの選択](#ステップ1---クエリのコネクションフィールドの選択)
    - [ステップ2 - コネクションのエッジをマッピングする](#ステップ2---コネクションのエッジをマッピングする)
    - [ステップ3 - 低次元なニュースフィードをフラグメントにいれる](#ステップ3---低次元なニュースフィードをフラグメントにいれる)
    - [ステップ4 - ページ分けのためのフラグメント引数](#ステップ4---ページ分けのためのフラグメント引数)
    - [ステップ5 - usePaginationFragmentの呼び出し](#ステップ5---usepaginationfragmentの呼び出し)
    - [ステップ6 - スクロールをトリガーに使用したページ分け](#ステップ6---スクロールをトリガーに使用したページ分け)
  - [まとめ](#まとめ)

このセクションでは、リストのページ分けと無限スクロールを含む、多くのアイテムのコネクションを扱う方法を確認します。
Relayにおいて、ページ分けと無限するコールリストは、*コネクション*として知られる抽象化を使用して扱われます。

---

Relayはアイテムのコレクションをページ分けを扱うときに、多くの作業をします。
しかし、それをするためには、スキーマでそれらのコレクションをモデル化する特定の慣習に依存しています。
この慣習は、強力で柔軟で、アイテムのコレクションを使用した多くの製品を構築するときの経験から生まれました。
この方法で機能する理由を理解するために、このスキーマの慣例のために、設計プロセスを介して進みましょう。

理解するために3つの重要な点があります。

- エッジ自身は属性を持ちます。
  例えば、友人リストにおいて、その人を友人にした日付は、その人自体の属性ではなく、あなたとその人の間のエッジの属性です。
  エッジを表現するノードを作成することから、これを処理します。
- 次のページが存在するかどうかなど、そのリスト自身が属性を持ちます。
  リスト自身を表現するノードと同様に現在のページを表現するノードを使用して処理します。
- オフセットではなく、次の結果のページを指し示す不透明なシンボルである、*カーソル*によってページ分けします。

ユーザーの友人のリストを表示したい場面を想像してください。
高いレベルでは、ビューアーとカララの友人がそれぞれノードになっているグラフを想像します。
ビューアーからそれぞれの友人のノードはエッジで、エッジ自身が属性を持ちます。

> 友人になった日は、そのエッジの属性である。

![the friend conceptual graph](https://relay.dev/assets/images/connections-conceptual-graph-a88c591f79c81af5d67b28122e17c581.png)

ここで、GraphQLを使用してこの場っ今日をモデルすることを試してみましょう。

GraphQLにおいて、ノードのみが属性を持つことができ、エッジは持てません。
よって、最初にすることは、まさにそれ自身のノードを使用して、あなたからあなたの友人に向かう概念的なエッジを表現することです。

![representing conceptual edges](https://relay.dev/assets/images/connections-edge-nodes-63a89a1f08749046137894e4b24774cc.png)

> ビューアーは「あなた」を示す。
> ビューアーから友人に向かう概念的なエッジを`FriendEdge`として表現する。
> `FriendEdge`は、友人に向かう`node`エッジを持つ。

ここで、エッジの属性は`FriendEdge`と呼ばれるノードの新しい型によって表現されます。

GraphQLはこれを次のようにこれをクエリします。

```graphql
// 単なる例であり、最終的なコードではありません
fragment FriendsFragment1 on Viewer {
  friends {
    since   // エッジの属性
    node {
      name  // 友人自身の属性
    }
  }
}
```

これにより、GraphQL内に、その人と友人になった日付を示す、エッジが作成された日付のようなエッジ特有の情報を配置する良い場所を持てます。

---

ここで、ページ分けと無限スクロールをサポートするために、スキーマ内でモデルする必要があることは何か考えてください。

- クライアントは、クライアントが望むページの大きさ（1ページに含まれるリストの要素数）を指定できなくてはなりません。
- クライアントは、クライアントが"次のページ"ボタンを有効または無効にするか、または、無限スクロールのために、将来リクエストする音を停止できるかどうかを確認するために、次のページが有るかどうか、情報を与えられなければなりません。
- クライアントは、すでに持っているページの次のページを問い合わせできなくてはなりません。

これらのことをするために、どのようにGraphQLの機能を使用できるのでしょうか？
ページサイズを指定することは、フィールド引数でできます。
言い換えれば、単なるに`friends`の代わりに、クエリは`friends(first: 3)`を伝えて、ページサイズ引数を`friends`フィールドに渡します。

> `first: 3`は最初から3つまでを示し、3つはページサイズである。
> したがって、`friends(first: 3)は最初の友人3人をクエリである。

サーバーが次のページが存在するか存在しないかを伝えるために、ちょうど、エッジ自身の情報を蓄積するそれぞれのエッジのためのノードを導入したように、*友人のリスト自身*についての情報を持つノードをグラフ導入する必要があります。
この新しいノードが*コネクション*と呼ばれます。

コネクションノードは、あなたとあなたの友人間の接続自体を表現します。
コネクションについてのメタデータは、そこで蓄積されます。
例えば、コネクションは、あなたがどれだけの人数の友人を持っているか伝える`totalCount`フィールドを持つ可能性があります。
加えて、現在のページを表現する2つのフィールドが常に存在します。
`pageInfo`フィールドには、別のページが利用可能かどうかなど、現在のページに関するメタデータが含まれます。
また、`edges`フィールドには、前に確認したエッジを示すフィールドがあります。

![friends connection](https://relay.dev/assets/images/connections-full-model-13e2ea59a6271992070d77ae4ae5e9b6.png)

最後に、次のページの結果をリクエストする方法が必要です。
`PageInfo`ノードは`lastCursor`と呼ばれるフィールドを持っています。
これは、与えられたリストの最後のエッジの位置（友人の"Charmaine"）を表現する、サーバーによって提供された不透明なトークンです。
そして、次のページを取得するために、サーバーにこのカーソルを渡すことができます。

`friends`フィールドの引数として、`lastCursor`値をサーバーに渡すことにより、すでに取得した友人の後の友人をサーバーに問い合わせできます。

![asking for the next page](https://relay.dev/assets/images/connections-full-model-next-page-88f27142830f48b2f0184ebcd0e9ad8d.png)

ページ分けされたリストのモデリングをしたこの全体のスキーマは、[GraphQLカーソルコネクション仕様](https://relay.dev/graphql/connections.htm)に詳細に示してあります。

それは多くの様々なアプリケーションにとって柔軟性があますが、Relayは自動的にページ分けを処理するために、この慣例に依存していて、この方法で設計されたスキーマは、Relayを使用している、使用していないにかかわらず良い考えです。

これで、コネクションのモデルの基礎を説明してきました。
次は、実際にそれを使用してニュースフィードストーリのコメントを実装することの注目しましょう。

---

## "さらにコメントを読み込み"を実装

一度、`Story`コンポーネントにより注意を払ってください。
インポートそして`Story`の下部に追加できる`StoryCommentsSection`コンポーネントがあります。

```tsx
import StoryCommentsSection from './StoryCommentsSection';  // 追加

function Story({story}) {
  const data = useFragment(StoryFragment, story);
  return (
    <Card>
      <Heading>{data.title}</Heading>
      <PosterByline person={data.poster} />
      <Timestamp time={data.posted_at} />
      <Image image={data.image} />
      <StorySummary summary={data.summary} />
      <StoryCommentsSection story={data} />   /{* 追加 *}/
    </Card>
  );
}
```

そして、`Story`フラグメントに`StoryCommentsSection`のフラグメントを追加してください。

```tsx
const StoryFragment = graphql`
  fragment StoryFragment on Story {
    // ... as before
    ...StoryCommentsSectionFragment
  }
`;
```

ここで、それぞれのストーリーの3つのコメントを確認できるはずです。
いくつかのストーリーは3つ以上のコメントがあり、そしてこれらは"さらに読み込み"ボタンを表示する予定ですが、まだそれは組み合わせていません。

![load more comments](https://relay.dev/assets/images/connections-comments-initial-screenshot-14ed0e7a93720ef62d79f7528a0f5ea8.png)

ここで、`StoryCommentsSection`に移動して、注意深く確認してください。

```tsx
const StoryCommentsSectionFragment = graphql`
 fragment StoryCommentsSectionFragment on Story {
  comments(first: 3) {
    edges {
      node {
        ...CommentFragment
      }
    }
    pageInfo {
      hasNextPage
    }
  }
 }
`;

function StoryCommentsSection({story}) {
  const data = useFragment(StoryCommentsSectionFragment, story);
  const onLoadMore = () => {/* TODO */};
  return (
    <>
      {data.comments.edges.map(commentEdge =>
        <Comment comment={commentEdge.node} />
      )}
      {data.comments.pageInfo.hasNextPage && (
        <LoadMoreCommentsButton onClick={onLoadMore} />
      )}
    </>
  );
}
```

ここで、コネクションスキーマの慣例を使用して、それぞれのストーリーについて最初の3つのコメントを選択した、`StoryCommentsSelection`を確認します。
`comments`フィールドは、引数としてページサイズを受け取り、それぞれのコメントごとに`edge`があり、その`edge`の中に実際のコメントデータを含んだ`node`があります。
`Comment`コンポーネント内で個々のコメントを表示するために必要なデータを取得するために、ここに`CommentFragment`を拡散しています。
また、それは、どれに"さらに読み込み"ボタンを表示するか決定するために、コネクションの`pageInfo`フィールドを使用します。

そして、タスクは、コメントの追加ページを実際に読み込む"さらに読み込み"ボタンを作成することです。
Relayはザラザラとした詳細を処理しますが、それを準備するためにいくつかのステップを提供しなくては成りません。

### フラグメントに引数を付ける

コンポーネントを修正する前に、フラグメントそれ自身が3つの追加情報を必要とします。
まず、フラグメントの引数として、ハードコードするよりも、ページサイズとカーソルを受け取る必要があります。

```tsx
const StoryCommentsSectionFragment = graphql`
  fragment StoryCommentsSectionFragment on Story
    @argumentDefinitions(
      cursor: { type: "String" }
      count: { type: "Int", defaultValue: 3 }
    )
  {
    comments(after: $cursor, first: $count) {
      edges {
        node {
          ...CommentFragment
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
`;
```

次に、Relayが引数の新しい値で、再度それを取得できるようにするために、フラグメントを[再取得可能](https://relay.dev/docs/tutorial/refetchable-fragments/)にする必要があります。
つまり、`$cursor`引数の新しいカーソルです。

```tsx
const StoryCommentsSectionFragment = graphql`
  fragment StoryCommentsSectionFragment on Story
    @refetchable(queryName: "StoryCommentsSectionPaginationQuery")  // 追加
    @argumentDefinitions(
    ... as before
`;
```

これにより、フラグメントにする必要がある、たったもう1つの変更があります。
Relayはページ分けをするコネクションを表現するフラグメント内の*どのフィールド*がどれか知る必要があります。
それをするために、それを`@connection`ディレクティブでマークします。

```tsx
const StoryCommentsSectionFragment = graphql`
  fragment StoryCommentsSectionFragment on Story
    @refetchable(queryName: "StoryCommentsSectionPaginationQuery")
    @argumentDefinitions(
      cursor: { type: "String" }
      count: { type: "Int", defaultValue: 3 }
    )
  {
    comments(after: $cursor, first: $count)
      @connection(key: "StoryCommentsSectionFragment_comments") // 追加
    {
      edges {
        node {
          ...CommentFragment
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
`;
```

`@connection`ディレクティブは、一意な文字列でなくてはならない`key`引数を要求します。
ここでは、フラグメントの名前とフィールドの名前で構成されています。
このキーは、ミューテーションの間にコネクションのコンテンツが編集されたときに使用され、次のチャプターで確認します。

### usePaginationFragmentフック

これで、すべてのフラグメントが強化されたため、さらに読み込みボタンを実装するためにコンポーネントを修正できます。

`StoryCommentsSection`コンポーネントの上部あるに次の2行を取り出してください。

```tsx
const data = useFragment(StoryCommentsSectionFragment, story);
const onLoadMore = () => {/* TODO */};
```

そして、それらを次で置き換えます。

```tsx
const {data, loadNext} = usePaginationFragment(StoryCommentsSectionFragment, story);
const onLoadMore = () => loadNext(3);
```

これにより、さらに読み込みボタンは、ロードされた他の3つのコメントを引き起こすはずです。

> Now the Load More button should cause another three comments to be loaded.

### useTransitionを使用した読み込み体験の改善

現状では、"さらに読み込み"ボタンをクリックしたとき、新しいコメントの読み込みが完了して表示されるまで、ユーザーのフィードバックはありません。
すべてのユーザーの行動は、迅速なフィードバックとして得られるべきであるため、新しいデータが読み込まれるまで、既存のUIを隠すことなく、スピナーを表示しましょう。

それをするために、Reactのトランジションの内部に`loadNext`呼び出しをラップする必要があります。
ここに、する必要がある変更を示します。

```tsx
function StoryCommentsSection({story}) {
  const [isPending, startTransition] = useTransition();
  const {data, loadNext} = usePaginationFragment(StoryCommentsSectionFragment, story);
  const onLoadMore = () => startTransition(() => {
    loadNext(3);
  });
  return (
    <>
      {data.comments.edges.map(commentEdge =>
        <Comment comment={commentEdge.node} />
      )}
      {data.comments.pageInfo.hasNextPage && (
        <LoadMoreCommentsButton
          onClick={onLoadMore}
          disabled={isPending}
        />
      )}
      {isPending && <CommentsLoadingSpinner />}
    </>
  );
}
```

> `CommentsLoadingSpinner`コンポーネントが存在せず、またスピナーも正常に表示されないため、次のように実装する。
> <https://github.com/facebook/relay/issues/4531#issuecomment-1867831999>

```tsx
export default function StoryCommentsSection({ story }: Props) {
  //const [isPending, startTransition] = useTransition();
  const { data, loadNext, isLoadingNext } = usePaginationFragment(
    StoryCommentsSectionFragment,
    story
  );
  //const onLoadMore = () =>
  //  startTransition(() => {
  //    loadNext(3);
  //  });
  const onLoadMore = () => loadNext(3);

  return (
    <div>
      {data.comments.edges.map((edge) => (
        <Comment key={edge.node.id} comment={edge.node} />
      ))}
      {/*
      {data.comments.pageInfo.hasNextPage && (
        <LoadMoreCommentsButton onClick={onLoadMore} disabled={isPending} />
      )}
      {isPending && <SmallSpinner />}
      */}
      {data.comments.pageInfo.hasNextPage && (
        <LoadMoreCommentsButton onClick={onLoadMore} disabled={isLoadingNext} />
      )}
      {isLoadingNext && <SmallSpinner />}
    </div>
  );
}
```

迅速でない結果を持つすべてのユーザーの行動は、Reactのトランジションの内部にラップされるべきです。
これは、Reactに様々な更新の優先順位を付けさせます。
例えば、データが利用可能になり、Reactが新しいコンポーネントをレンダリングしているとき、ユーザーが異なるページに遷移する他のタブをクリックした場合、Reactはユーザが望んだ新しいページをレンダリングするために、コメントのレンダリングを割り込みできます。

---

## ニュースフィードストーリーの無限スクロール

ニュースフィードの無限スクロールを作成するために、ページネーションで学んだことを使用しましょう。
ニュースフィードは、ボタンを押すことではなく、ユーザーがページの下の方へスクロールしたときに、自動的に`loadNext`が発行されることを除いて、"さらにコメントを読み込み"とかなり同じです。

### ステップ1 - クエリのコネクションフィールドの選択

今すぐ、アプリは、上位3つのストーリーの単純な配列を取得するために、`topStories`ルートフィールドを使用します。
また、このスキーマは、コネクションである`Viewer`にある`newsfeedStories`フィールドも提供します。
この新しいフィールドを使用するために`Newsfeed`コンポーネントを修正しましょう。
一度、`Newsfeed.tsx`により注意を向けてください。
最上位のGraphQLクエリは次のようになっているはずです。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    topStories {
      id
      ...StoryFragment
    }
  }
`;
```

先に進んで、それを次で置き換えてください。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    viewer {
      newsfeedStories(first: 3) {
        edges {
          node {
            id
            ...StoryFragment
          }
        }
      }
    }
  }
`;
```

ここで、初期で最初の3つのストーリーを取得するために、`first`引数を追加して、`topStories`を`viewer`の`newsfeedStories`で置き換えました。
その中で`edge`を選択して、次に`node`を選択しました。
それは、前と同じ`StoryFragment`を拡散できる`Story`ノードです。
また、Reactの`key`属性としてそれを使用するために`id`も選択しました。

> **💡 TIP**
> 単純にするために、`Query`の最上位に`topStory`と`topStories`フィールドを置きましたが、ページまたはアプリを見ている人に関連するフィールドを、`viewer`と呼ばれるフィールドの下に置くことが慣例です。
> 現在、実際のアプリと同じようにフィールドを使用するため、その慣例に切り替えました。

### ステップ2 - コネクションのエッジをマッピングする

エッジのマッピングして、それぞれのノードをレンダリングするために、`Newsfeed`コンポーネントを修正する必要があります。

```tsx
function Newsfeed() {
  const data = useLazyLoadQuery(NewsfeedQuery, {});
  const storyEdges = data.viewer.newsfeedStories.edges;
  return (
    <>
      {storyEdges.map(storyEdge =>
        <Story key={storyEdge.node.id} story={storyEdge.node} />
      )}
    </>
  );
}
```

### ステップ3 - 低次元なニュースフィードをフラグメントにいれる

Relayのページ分け機能は、クエリ全体ではなく、フラグメントとのみ機能します。
これは、この単純なアプリ例において、このコンポーネント内で直接クエリを発行していますが、実際のアプリケーションにおいて、通常、クエリは高次元のルーティングコンポーネントによって発行され、それがページ分けされたリストを表示するコンポーネンと同じになることは、滅多にないことが理由です。

機能するようにこれをするために、単純に`NewsfeedQuery`ｗフラグメントに分離する必要があり、それを`NewsfeedContentsFragment`と呼びます。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    ...NewsfeedContentsFragment
  }
`;

const NewsfeedContentsFragment = graphql`
  fragment NewsfeedContentsFragment on Query {
    viewer {
      newsfeedStories {
        edges {
          node {
            id
            ...StoryFragment
          }
        }
      }
    }
  }
`;
```

すべてのGraphQLスキーマが、クエリで利用できる最上位フィールドを表現する`Query`と呼ばれる方を含んでいることを言及する良いときです。
`on Query`でフラグメントを定義することで、最上位にそれを直接拡散できます。

`Newsfeed`内で、`useLazyLoadQuery`と`useFragment`を両方呼び出すことができますが、現実世界では、一般的にこれらは異なるコンポーネントです。

```tsx
export default function Newsfeed() {
  const queryData = useLazyLoadQuery(NewsfeedQuery, {});
  const data = useFragment(NewsfeedContentsFragment, queryData);
  const storyEdges = data.newsfeedStories.edges;
  ...
}
```

> 上記は、型エラーが発生するため、次のように実装する。
> <https://github.com/facebook/relay/issues/4183#issue-1529721645>

```tsx
import * as React from "react";
import Story from "./Story";
import { graphql } from "relay-runtime";
import { useFragment, useLazyLoadQuery } from "react-relay";
import type { NewsfeedQuery as NewsfeedQueryType } from "./__generated__/NewsfeedQuery.graphql";
import type { NewsfeedContentsFragment$key } from "./__generated__/NewsfeedContentsFragment.graphql";

const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    ...NewsfeedContentsFragment
  }
`;

const NewsfeedContentsFragment = graphql`
  fragment NewsfeedContentsFragment on Query {
    viewer {
      newsfeedStories {
        edges {
          node {
            id
            ...StoryFragment
          }
        }
      }
    }
  }
`;

export default function Newsfeed() {
  const queryData = useLazyLoadQuery<NewsfeedQueryType>(NewsfeedQuery, {});
  const data = useFragment<NewsfeedContentsFragment$key>(
    NewsfeedContentsFragment,
    queryData
  );
  const storyEdges = data.viewer.newsfeedStories.edges;
  return (
    <div className="newsfeed">
      {storyEdges.map((storyEdge) => (
        <Story key={storyEdge.node.id} story={storyEdge.node} />
      ))}
    </div>
  );
}
```

### ステップ4 - ページ分けのためのフラグメント引数

ここで、ストーリーのためにコネクションフィールドを使用して、フラグメント自身で持ったため、ページ分けをサポートするために必要な変更をフラグメントにすることができます。
これらは最後のサンプルと同じです。
次をする必要があります。

- ページサイズとカーソルのtめにフラグメント引数`first`と`after`を追加します。
- フィールド引数として`newsfeedStories`フィールドにそれらの引数を渡します。
- `@refetchable`としてフラグメントをマークします。
- `@connection`で`newsfeedStories`をマークします。

次のように仕上げてください。

```tsx
const NewsfeedContentsFragment = graphql`
  fragment NewsfeedContentsFragment on Query
  @argumentDefinitions(
    cursor: { type: "String" }
    count: { type: "Int", defaultValue: 3 }
  )
  @refetchable(queryName: "NewsfeedContentsRefetchQuery") {
    viewer {
      newsfeedStories(after: $cursor, first: $count)
        @connection(key: "NewsfeedContentsFragment_newsfeedStories") {
        edges {
          node {
            id
            ...StoryFragment
          }
        }
      }
    }
  }
`;
```

### ステップ5 - usePaginationFragmentの呼び出し

これにより、`usePaginationFragment`を呼び出すために`NewsfeedContents`を修正する必要があります。

> 上記の`NewsfeedContents`は、`Newsfeed`の間違いであると考えられる。
> <https://github.com/facebook/relay/pull/4478#issue-1940539233>

```tsx
export default function Newsfeed() {
  const queryData = useLazyLoadQuery<NewsfeedQueryType>(NewsfeedQuery, {});
  const { data, loadNext } = usePaginationFragment<
    OperationType,
    NewsfeedContentsFragment$key
  >(NewsfeedContentsFragment, queryData);
  const storyEdges = data.viewer.newsfeedStories.edges;
  return (
    <div className="newsfeed">
      {storyEdges.map((storyEdge) => (
        <Story key={storyEdge.node.id} story={storyEdge.node} />
      ))}
    </div>
  );
}
```

### ステップ6 - スクロールをトリガーに使用したページ分け

サイボのページに到達したときを検出する`InfiniteScrollTrigger`と呼ばれるコンポーネントを準備しました。
適切なときに`loadNext`を呼び出すためにこれを使用できます。
それは、さらにページが存在するかどうか、現在次のページを読み込んでいるかを知る必要があります。
`userPaginationFragment`の戻り値からこれらを取得することができます。

> 次の実装は、オリジナルを修正した実装である。

```tsx
export default function Newsfeed() {
  const queryData = useLazyLoadQuery<NewsfeedQueryType>(NewsfeedQuery, {});
  const { data, loadNext, hasNext, isLoadingNext } = usePaginationFragment<
    OperationType,
    NewsfeedContentsFragment$key
  >(NewsfeedContentsFragment, queryData);
  function onEndReached() {
    loadNext(3);
  }
  const storyEdges = data.viewer.newsfeedStories.edges;
  return (
    <div className="newsfeed">
      {storyEdges.map((storyEdge) => (
        <Story key={storyEdge.node.id} story={storyEdge.node} />
      ))}
      <InfiniteScrollTrigger
        onEndReached={onEndReached}
        hasNext={hasNext}
        isLoadingNext={isLoadingNext}
      />
    </div>
  );
}
```

これにより、ページの下部までスクロールして、さらにストーリーを読み込むことができるようになったはずです。
本当のニュースフィードアプリのような感覚です。

---

## まとめ

- コネクションは、ページ分け可能なリストの振る舞いをモデルするために、Relayが依存したスキーマの慣例です。
- 一般的に、単純なリストよりもスキーマでコネクションをしたほうが良い考えです。
  これは、必要な場合にページ分けを柔軟にします。

次は、最後にサーバーにあるデータを更新する方法を確認します。
新しく作成されたノードを既存のコネクションに追加する方法と同様に、コネクションがその役目を同様に果たします。

# ミューテーションと更新

<https://relay.dev/docs/tutorial/mutations-updates/>

- [ミューテーションと更新](#ミューテーションと更新)
  - [ライクボタンの実装](#ライクボタンの実装)
  - [ミューテーションの解剖](#ミューテーションの解剖)
  - [ライクミューテーション](#ライクミューテーション)
  - [Relayが自動的にミューテーションのレスポンスを処理する方法](#relayが自動的にミューテーションのレスポンスを処理する方法)
  - [ミューテーションのレスポンス内でフラグメントを使用する](#ミューテーションのレスポンス内でフラグメントを使用する)
  - [楽観的アップデーターを使用したUXの改善](#楽観的アップデーターを使用したuxの改善)
    - [ステップ1 - commitMutationにoptimisticUpdaterを追加する](#ステップ1---commitmutationにoptimisticupdaterを追加する)
    - [ステップ2 - 更新可能フラグメントを作成する](#ステップ2---更新可能フラグメントを作成する)
    - [ステップ3 - readUpdatableFragmentを呼び出す](#ステップ3---readupdatablefragmentを呼び出す)
  - [コメントの追加 - コネクション上のミューテーション](#コメントの追加---コネクション上のミューテーション)
    - [ステップ1 - コメントを投稿するミューテーションを定義する](#ステップ1---コメントを投稿するミューテーションを定義する)
    - [ステップ2 - 投稿するためにcommitMutationを呼び出す](#ステップ2---投稿するためにcommitmutationを呼び出す)
    - [ステップ3 - 宣言的コネクションハンドラを追加する](#ステップ3---宣言的コネクションハンドラを追加する)
    - [ステップ4 - ミューテーションの変数としてコネクションのIDを渡す](#ステップ4---ミューテーションの変数としてコネクションのidを渡す)
  - [まとめ](#まとめ)

このチャプターでは、サーバーとクライアントにあるデータを更新する方法を学びます。
それらは主に2つの例を通じて説明します。

- ニュースフィードストーリーのライクボタンの実装
- ニュースフィードストーリーにコメントを投稿する機能の実装

データを更新することは、ドメインの複雑な問題です。
アプリは、多くの手動の制御を与えて発生するケースを処理することで、可能な限り強固にする一方で、Relayは多くの側面を自動的に処理します。

まず、2つの項目を区別しましょう。

- *ミューテーション*は、サーバーにあるデータを修正する任意のアクションを実行するために、サーバーに問い合わせすることです。
  これはHTTPのPOSTと類似したGraphQLの機能です。
- Relayのローカルなクライアント側のデータストアを更新することです。

クライアントは、サーバー側にある個々のデータの一部を直接操作する能力を持っていません。
むしろ、ミューテーションは不透明で、ユーザーの意図を表現した高水準のリクエストです。
例えば、ユーザーは、誰かと友人になり、グループに参加して、コメントを投稿して、特定のニュースフィードストーリーにライクして、誰かをブロックして、またはコメントを削除します。
GraphQLスキーマは、どのようなミューテーションが利用できるか、またそれぞれのミューテーションが受け付ける入力引数も同様に定義します。

ミューテーションは、グラフの状態に、遠く届かずそして予期しない作用を持つ可能性があります。
例えば、グループに参加して、多くのことを変更できます。

- あなたの名前をグループメンバーリストに追加します。
- グループのメンバーシップ数を増やします。
- グループはグループのリストにあなたを追加します。
- グループの投稿フィードにメンバーだけの投稿が表示されます。
- あなたのお気に入りのグループを変更するかもしれません。
- グループの管理者は通知を出すかもしれません。
- ログの記録、モデルの訓練またはEメールの痩身の湯王な、ユーザーに見えない作用が多くあります。
- など。

一般的に、ミューテーションのかもしれない完全に下流の効果を知ることは不可能です。
よって、ミューテーションを実行することをサーバーに問い合わせた後で、単純にクライアントはそのローカルデータストアを更新して、可能な限り一貫性と合理性を持つデータを維持することを最大限に行います。
それは、ミューテーションのレスポンスデータの一部として特定の更新されたデータをサーバーに質問すること、また*updater*と呼ばれる命令的なコードによって可能です。
*updater*は、その一貫性を維持するようにストアをまとめます。

すべてのケースをカバーする原則的な解決方法はありません。
グラフ上に発生するミューテーションの完全な影響を知ることが可能でも、すぐに更新されたデータを見せたくない状況もあるかもしれません。
例えば、誰かのプロファイルページに進んで、それらをブロックした場合、そのページに表示されているすべてが、すぐに消えてしまうことは望んでいません。
どのデータを更新するかという質問は、最終的にUI設計の決定です。

Relayは、ミューテーションに応答して、データを更新することを可能な限り容易にします。
例えば、あるコンポーネントを更新したい場合、ミューテーション内にそのフラグメントを展開して、ミューテーションはフラグメントによって選択された更新データをサーバーに送信するために問い合わせます。
他の場合、Relayのローカルデータストアを強制的に修正する*updater*を手動で記述しなければなりません。
次でこれらのケースを確認します。

---

## ライクボタンの実装

ニュースストーリーのためにライクボタンを実装することで、水につま先を入れましょう。
幸いにも、すでにライクボタンを準備しているため、`Story.tsx`を開き、それを`Story`コンポーネントの中に落として、忘れないようにストーリーフラグメントの中にそのフラグメントを展開します。

```tsx
import StoryLikeButton from './StoryLikeButton';

...

const StoryFragment = graphql`
  fragment StoryFragment on Story {
    title
    summary
    // ... etc
    ...StoryLikeButtonFragment
  }
`;

...

export default function Story({story}: Props) {
  const data = useFragment(StoryFragment, story);
  return (
    <Card>
      <PosterByline person={data.poster} />
      <Heading>{data.title}</Heading>
      <Timestamp time={data.posterAt} />
      <Image image={story.thumbnail} width={400} height={400} />
      <StorySummary summary={data.summary} />
      <StoryLikeButton story={data} />
      <StoryCommentsSection story={data} />
    </Card>
  );
}
```

ここで、`StoryLikeButton.js`を確認してください。
現在、それは、ライク数を表示する、何もしないボタンでです。

![initial like button](https://relay.dev/assets/images/mutations-like-button-e1ff48a46424ab56ae8926749b065a8f.png)

そのフラグメントは、ライクカウントを取得する`likeCount`フィールドと、どのライクボタンをハイライトされるかを決定する`doesViewerLike`フィールドを確認してください。
例えば、`doesViewerLike`が`true`で、ビューアーがそのストーリーを好きな場合にハイライトさせます。

```tsx
const StoryLikeButtonFragment = graphql`
  fragment StoryLikeButtonFragment on Story {
    id
    likeCount
    doesViewerLike
  }
`;
```

ライクボタンを押したときに次をしたいと考えています。

1. ストーリーがサーバーにある"Liked"を取得します。
2. ローカルクライアントコピーの`likeCount`と`doesViewerLike`が更新されます。

それをするために、GraphQLのミューテーションを記述することが必要ですが、最初に・・・

## ミューテーションの解剖

GraphQLのミューテーションの構文は、最初に次の理解なしでは混乱するでしょう。

GraphQLは、クエリとニューテーションの2つの異なるリクエスト型を持っています。
そして、それらは正確に同じ方法で機能します。
これは、HTTPがGETとPOSTを行う方法と類似しています。
技術的に、唯一の違いは、POSTリクエストは影響を引き起こすことを意図されており、GETリクエストはそうではありません。
同様に、ミューテーションは、ミューテーションがモノを引き起こすことを期待されていることを除いて、クエリと正確に同じです。
これは次を意味します。

- ミューテーションはクライアントコードの一部です。
- ミューテーションは、クライアントがサーバーにデータを渡すようにする*変数*を宣言します。
- サーバーは*個々のフィールド*を実装します。
  与えられたミューテーションは、それらのフィールドを一緒に構成して、フィールド引数としてその変数を渡します。
- それぞれのフィールドは、スカラーまたは他のグラフノードへのエッジのいずれかになる、特定の型のデータを生成します。
  それは、そのフィールドが選択された場合、クライアントに返されます。
  エッジの場合、リンクされているノードから、さらにフィールドが選択されます。

唯一の違いは、ミューテーションにおいて、フィールドを選択することは何かを発生させ、また同様にデータを返します。

## ライクミューテーション

それを心に留めて、ミューテーションは次のようになります。
先に進んで、ファイルにこの宣言を追加しましょう。

```tsx
const StoryLikeButtonLikeMutation = graphql`
  mutation StoryLikeButtonLikeMutation(
    $id: ID!,
    $doesLike: Boolean!,
  ) {
    likeStory(
      id: $id,
      doesLike: $doesLike
    ) {
      story {
        id
        likeCount
        doesViewerLike
      }
    }
  }
`;
```

これは多いため、1つずつ確認しましょう。

- モジュール名から始まり、GraphQLの操作で終了しなければならないため、ミューテーションは、`StoryLikeButton` + `Like` + `Mutation`と名付けられています。
- ミューテーションは、ミューテーションが呼び出されたときにクライアントからサーバーに渡される変数を宣言します。
  それぞれの変数は、`$id`、`$doesLike`のような名前と`ID!`、`Boolean!`のような型を持ちます。
  型の後の`!`は、それがオプションではなく、必須であることを示します。
- ミューテーションは、GraphQLスキーマによって定義されたミューテーションフィールドを選択します。
  サーバーが定義したそれぞれのミューテーションフィールドは、ストーリーをライクするような、クライアントがサーバーにリクエストできる任意のアクションに対応します。
  - ミューテーションフィールドは、任意のフィールドができるように、引数を受け取ります。
    ここで、引数として宣言としたミューテーション変数を渡します。
    例えば、`doesLike`フィールド引数は、`$doesLike`ミューテーション変数に設定されます。
- `likeStory`フィールドは、ミューテーションのレスポンスを表現するノードへのエッジです。
  ミューテーションのレスポンス内で利用されるそのフィールドは、GraphQLスキーマによって指定されます。
  - `story`フィールドを選択して、それはライクした`Story`へのエッジです。
  - 更新された`Story`内の特定のフィールドを選択します。
    これらは`Story`を選択できるクエリと同じフィールドです。
    実際に、フラグメント内で同じフィールドを選択しました。

このミューテーションをサーバーに送信したとき、クエリと似た、送信したミューテーションの形状と一致するレスポンスを得ます。
例えば、サーバーは次を送信するかもしれません。

```json
{
  "likeStory": {
    "story": {
      "id": "34a8c",
      "likeCount": 47,
      "doesViewerLike": true
    }
  }
}
```

そして、わたしたちの仕事は、この更新された情報を組み込むためにローカルデータストアを更新することです。
Relayは単純なケースでこれを処理しますが、より複雑なケースでは、賢くストアを更新する独自のコードが要求されます。

しかし、進めるために、ミューテーションを発行するボタンを作成しましょう。
ここに、コンポーネントを示します。
`StoryLikeButtonLikeMutation`を実行するために`onLikeButtonClicked`イベントを組み合わせる必要があります。

```tsx
function StoryLikeButton({story}) {
  const data = useFragment(StoryLikeButtonFragment, story);
  function onLikeButtonClicked() {
    // To be filled in
  }
  return (
    <>
      <LikeCount count={data.likeCount} />
      <LikeButton value={data.doesViewerLike} onChange={onLikeButtonClicked} />
    </>
  )
}
```

それをするために、`useMutation`の呼び出しを追加します。

```tsx
import {useMutation, useFragment} from 'react-relay';

function StoryLikeButton({story}) {
  const data = useFragment(StoryLikeButtonFragment, story);
  const [commitMutation, isMutationInFlight] = useMutation(StoryLikeButtonLikeMutation);
  function onLikeButtonClicked() {
    commitMutation({
      variables: {
        id: data.id,
        doesLike: !data.doesViewerLike,
      },
    })
  }
  return (
    <>
      <LikeCount count={data.likeCount} />
      <LikeButton value={data.doesViewerLike} onChange={onLikeButtonClicked} />
    </>
  )
}
```

`useMutation`フックは、なにかすることをサーバーに伝えるために呼び出すことができる`commitMutation`関数を返します。

ミューテーションによって定義された変数に値を与える`variables`と呼ばれるオプションを、`id`と`doesViewerLike`という名前で渡します。
これは、どのストーリーについて、ライクしようとしているのかアンライクしようとしているのかをサーバーに伝えます。
フラグメントから読み込んだストーリーの`id`は、それが好きか嫌いかは、レンダリングした現在の値を切り替えることで決まります。

そのフックは、ミューテーションが発行されたことを伝えるフラグも返します。
ミューテーションが発行されているときに、ボタンを無効にすることで、ユーザー体験を良くするために使うことができます。

```tsx
<LikeButton
  value={data.doesViewerLike}
  onChange={onLikeButtonClicked}
  disabled={isMutationInFlight}
/>
```

これを配置することで、ストーリーのようにできるはずです。

## Relayが自動的にミューテーションのレスポンスを処理する方法

しかし、どのようにRelayはクリックしたストーリーを更新することを知るのでしょうか？
サーバーは、この形式でレスポンスを送信しました。

```json
{
  "likeStory": {
    "story": {
      "id": "34a8c",
      "likeCount": 47,
      "doesViewerLike": true
    }
  }
}
```

レスポンスが`id`フィールドを持つオブジェクトを含むときはいつでも、Relayは、ストアがそのレコードの`id`フィールドと一致するIDはを持つレコードをすでに含んでいるか確認します。
もし、マッチする場合、Relayはレスポンスから得た他のフィールドと既存のレコードをマージします。
これは、この単純なケースにおいて、ストアを更新するコードを記述する必要はありません。

---

## ミューテーションのレスポンス内でフラグメントを使用する

ミューテーションは、ちょうどクエリに似ていることを覚えておいてください。
レンダリングしたいデータが常にミューテーションのレスポンスにむくまれていることを確認するために、手動で最新の状態を維持されなければならない別のフィールドの集合を持つ代わりに、単にフラグメントをミューテーションのレスポンス内に展開するだけです。

```tsx
const StoryLikeButtonLikeMutation = graphql`
  mutation StoryLikeButtonLikeMutation(
    $id: ID,
    $doesLike: Boolean,
  ) {
    likeStory(id: $id, doesLike: $doesLike) {
      story {
        ...StoryLikeButtonFragment
      }
    }
  }
`;
```

現在、データ要求を追加または削除した場合、不必要なデータを除いて、必要なデータすべてがミューテーションのレスポンスに含まれます。
通常、これはミューテーションのレスポンスを記述する良い方法です。
任意のコンポーネントだけでなく、ミューテーションを発行するコンポーネントからの任意のフラグメントを展開できます。
これは、細心な状態にUI全体を維持することに役立ちます。

---

## 楽観的アップデーターを使用したUXの改善

ミューテーションは実行するために時間がかかりますが、ユーザーがアクションを起こしたというフィードバックをユーザーに与えるために、なんあらかの方法でUIをすぐに更新したいと常に考えています。
現在の例において、ライクブタンは、ミューテーションが発生したときに無効になり、そしてミューテーションが終わると、Relayがマージした更新データをそのストアにマージして、影響したコンポーネントを再レンダリングすると、UIは新しい状態に更新されます。

よく、最高のフィードバックは、操作がすでに完了したように単純に見せかけることです。
例えば、ライクボタンを押した場合、そのボタンはすぐに、すでにライクしたことを確認できる状態を示す強調された状態になります。
また、コメント投稿例を取り上げると、投稿したコメントをすぐに表示したいです。
これは、普通、ミューテーションが十分に高速で信頼できるため、ユーザーにミューテーションを別の読み込み状態で煩わす必要がないからです。
しかし、よく、ミューテーションが失敗で終わります。
その場合、行った変更をロールバックして、ミューテーションを試行する前の状態を返してほしいです。
投稿中として表示されたコメントは消えるはずですが、コメントのテキストはそれを書いたコンポーザー内に再表示されるはずです。
これにより、再度投稿を試みる場合にデータが失われることはありません。

*楽観的更新*と呼ばれるこれらの管理は、手動ですることが複雑ですが、Relayは更新を適用またはロールバックする強固なシステムを持っています。
ユーザーが順番にいくつかの場面をクリックした場合など、同時に複数のミューテーションを実行することもでき、Relayは失敗した場合に、どの変更をロールバックするか追跡します。

ミューテーションは次の3つのフェーズで処理します。

- 最初は*楽観的更新*で、予想でき、すぐにユーザーに見せたい状態にローカルデータストアを更新します。
- そして、実際にサーバー上でミューテーションを実行します。
  成功した場合、サーバーは3番目のステップで使用される更新された情報で応答します。
- ミューテーションが完了したとき、楽観的更新をロールバックします。
  ミューテーションが失敗した場合、開始したところに戻ります。
  ミューテーションが成功した場合、Relayは単純に変更をストアにマージして、サーバーから受け取った実際の新しい情報と他の情報を付け加えて、ローカルデータストアを更新する*最終的な更新(concluding update)*を適用します。

![concluding update](https://relay.dev/assets/images/mutations-lifecycle-e7492d8faa5f6a7658b90dc6ceffb42e.png)

> `Apply Optimistic Update`で、ローカルデータストアを一旦更新する。
> ミューテーションが失敗した場合、`Roll Back Optimistic Update`で、ローカルデータストアを元に戻す。
> `Apply Concluding Update`で、ミューテーションの結果とその他の変更をマージして、ローカルデータストアを更新する。

---

このバックグラウンドの知識を踏まえて、先に進み、クリックされたとき、すぐに新しい状態に更新するために、ライクボタンの楽観的アップデータを記述しましょう。

### ステップ1 - commitMutationにoptimisticUpdaterを追加する

`StoryLikeButton`を表示して、`commitMutation`と呼ばれるオプションを追加します。

```tsx
function StoryLikeButton({story}) {
  ...
  function onLikeButtonClicked(newDoesLike) {
    commitMutation({
      variables: {
        id: data.id,
        doesLike: newDoesLike,
      },
      optimisticUpdater: store => {
        // TODO: 楽観的アップデーターで満たします。
      },
    })
  }
  ...
}
```

このコールバックは、Relayのローカルデータストアを表現する`store`引数を受け取ります。
ローカルデータストアは、ローカルデータを読み込みまたは書き込むさまざまなメソッドがあります。
楽観的アップデーターで行うすべての書き込みは、ミューテーションが呼び出されたときにすぐに適用され、完了したときロールバックされます。

### ステップ2 - 更新可能フラグメントを作成する

*更新可能フラグメント*と呼ばれる特別なフラグメントを記述することで、ローカルストアにあるデータを読み書きできます。
普通のフラグメントと異なり、それはクエリ内に展開されたり、サーバーに送信されたりしません。
代わりに、それはすでに理解して愛している同じGraphQL構文を使用して、ローカルストアからデータを読み込みできるようにします。
先に進んで、このフラグメントを追加しましょう。

```tsx
function StoryLikeButton({story}) {
  ...
      optimisticUpdater: store => {
        const fragment = graphql`
          fragment StoryLikeButton_updatable on Story
            @updatable
          {
            likeCount
            doesViewerLike
          }
        `;
      },
  ...
}
```

それは任意の他のフラグメントと正確に似ていますが、`@updatable`ディレクティブで注釈されています。

普通のフラグメントと異なり、更新可能フラグメントはクエリ内に展開されず、サーバーから選択したデータを取得しません。
代わりに、データが更新されるかもしれないため、すでにRelayのローカルデータストア内に存在するデータを選択します。

### ステップ3 - readUpdatableFragmentを呼び出す

そのストーリーが好きかを伝えるプロップとして受け取ったオリジナルのフラグメントの参照と一緒に、このフラグメントを`store.readUpdatableFragment`に渡します。
それは`updatableData`と呼ばれる特別なオブジェクトを返します。

```tsx
function StoryLikeButton({story}) {
  ...
      optimisticUpdater: store => {
        const fragment = graphql`
          fragment StoryLikeButton_updatable on Story @updatable {
            likeCount
            doesViewerLike
          }
        `;
        const {
          updatableData
        } = store.readUpdatableFragment(
          fragment,
          story
        );
      },
  ...
}
```

この例において、そのストーリーが好きなときに、それを好きでないようにするボタンをクリックするために、`doesViewerLike`を切り替えて、それに従ってライク数を増減します。

Relayは変更を作成した`updatableData`に記録して、ミューテーションが成功したらそれらをロールバックします。

これで、ライクボタンをクリックしたとき、UIがすぐに更新されることを確認するはずです。

## コメントの追加 - コネクション上のミューテーション

Relayが完全に自動的にできる唯一なことは、すでに確認してきたことです。
ストア内で同じIDを共有する既存のノードを持ったミューテーションのノードをマージします。
とにかく、いくつか情報をRelayに与える必要があります。

コネクションの例を確認しましょう。
ストーリーに新しいコメントを登録する能力を実装します。

サーバーのミューテーションのレスポンスは、新しく作成されたコメントのみを含んでいます。
ストーリーとそのコメントの間のコネクションに、そのストーリーを挿入する方法をRelayに伝えなくてはなりません。

`StoryCommentsSection`に戻り、新しいコメントを投稿するコンポーネントを追加して、フラグメント内にそのフラグメントを展開することを忘れないでください。

```tsx
import StoryCommentsComposer from './StoryCommentsComposer';

const StoryCommentsSectionFragment = graphql`
  fragment StoryCommentsSectionFragment on Story
    ...
  {
    comments(after: $cursor, first: $count)
      @connection(key: "StoryCommentsSectionFragment_comments")
    {
      ...
    }
    ...StoryCommentsComposerFragment
  }
`

function StoryCommentsSection({story}) {
  ...
  return (
    <>
      <StoryCommentsComposer story={data} />
      ...
    </>
  );
}
```

これで、コメントセクションの上部にコンポーザーを確認できるはずです。

![composer](https://relay.dev/assets/images/mutations-comments-composer-screenshot-a6d04dbbdaf9769f80c8f132f3358faa.png)

ここで、`StoryCommentsComposer.js`の中身を確認してください。

```tsx
function StoryCommentsComposer({story}) {
  const data = useFragment(StoryCommentsComposerFragment, story);
  const [text, setText] = useState('');
  function onPost() {
   // TODO post the comment here
  }
  return (
    <div className="commentsComposer">
      <TextComposer text={text} onChange={setText} onReturn={onPost} />
      <PostButton onClick={onPost} />
    </div>
  );
}
```

### ステップ1 - コメントを投稿するミューテーションを定義する

ちょうど前と同じように、ミューテーションを定義する必要があります。
それはストーリのIDと追加されるコメントのテキストをサーバーに送信します。

```tsx
const StoryCommentsComposerPostMutation = graphql`
  mutation StoryCommentsComposerPostMutation(
    $id: ID!,
    $text: String!,
  ) {
    postStoryComment(id: $id, text: $text) {
      commentEdge {
        node {
          id
          text
        }
      }
    }
  }
`;
```

ここで、スキーマは、新しく作成されたコメントに向かう新しく作成されたエッジをミューテーションのレスポンスの一部として選択できるようにします。
それを選択して、コネクション内にこのエッジを挿入することで、ローカルストアを更新するためにそれを使用します。

### ステップ2 - 投稿するためにcommitMutationを呼び出す

ここで、`commitMutation`コールバックにアクセスする`useMutation`フックを使用して、`onPost`内でそれを呼び出します。

```tsx
function StoryCommentsComposer({story}) {
  const data = useFragment(StoryCommentsComposerFragment, story);
  const [text, setText] = useState('');
  const [commitMutation, isMutationInFlight] = useMutation(StoryCommentsComposerPostMutation);
  function onPost() {
    setText(''); // UIをリセットする。
    commitMutation({
      variables: {
        id: data.id,
        text,
      },
    })
  }
  ...
}
```

### ステップ3 - 宣言的コネクションハンドラを追加する

ここで、投稿のクリックすることで、サーバーにミューテーションのリクエストを送信するネットワークログを見つけることができます。
ページを更新した場合、それが現れるため、投稿されたコメントを見ることもできます。
しかし、UI内では何も発生していません。
新しく作成されたコメントがストーリーからコメントへのコネクションに追加することをRelayに伝える必要があります。

上記で記述したミューテーションのレスポンス内に、`commentEdge`を選択したことに気付いたかもしれません。
これは新しく作成したコメントへのエッジです。
コネクションに何のエッジが追加されたのかを単純にRelayに伝える必要があります。
Relayは`@appendEdge`、`@prependEdge`、`@deleteEdge`と呼ばれるディレクティブを提供しており、それらはミューテーションのレスポンス内にそのエッジを置きます。
そして、ミューテーションを実行したとき、編集したいコネクションのIDを渡します。
Relayは、指定したそれらのコネクションに、エッジを追加、先頭に追加、削除します。

リストの最上部に新しく追加したコメントを表示したいため、`@prependEdge`を使用します。
ミューテーションの定義に、次を追加してください。

```tsx
  mutation StoryCommentsComposerPostMutation(
    $id: ID!,
    $text: String!,
    $connections: [ID!]!,
  ) {
    postStoryComment(id: $id, text: $text) {
      commentEdge
        @prependEdge(connections: $connections)
      {
        node {
          id
          text
        }
      }
    }
  }
```

ミューテーションに`connections`と呼ばれる変数を追加しました。
更新したいコネクションに渡すためにこれを使用します。

> **ℹ 注意事項**
> `$connections`変数は、`@prependEdge`ディレクティブへの引数としてのみ使用され、Relayによってクライアント上で処理されます。
> `$connections`はどのフィールドにも引数として渡されないため、それはサーバーに送信されません。

### ステップ4 - ミューテーションの変数としてコネクションのIDを渡す

新しいエッジを追加するためにコネクションを識別する必要があります。
コネクションは、2つの情報で識別されます。

- どのノードからオフになっているか(Which node it’s off of ) - この場合、コメントを投稿したストーリーです。
- `@connection`ディレクティブ内で提供された*key*は、1つ以上のコネクションが同じノードから離れている場合に、コネクションを区別できます。

Relayによって提供された特別なAPIを使用して、ミューテーション変数内にこの情報を渡します。

```tsx
import {useFragment, useMutation, ConnectionHandler} from 'react-relay';

...

export default function StoryCommentsComposer({story}: Props) {
  ...
  function onPost() {
    setText('');
    const connectionID = ConnectionHandler.getConnectionID(
      data.id,
      'StoryCommentsSection_comments',
    );
    commitMutation({
      variables: {
        id: data.id,
        text,
        connections: [connectionID],
      },
    })
  }
  ...
}
```

`getConnectionID`に渡した`"StoryCommentsSectionFragment_comments"`は、`StoryCommentSection`内でコネクションを取得するときに使用する識別子です。
参考までに、これは次のようなものでした。

```tsx
const StoryCommentsSectionFragment = graphql`
  fragment StoryCommentsSectionFragment on Story
  ...
  {
    comments(after: $cursor, first: $count)
     @connection(key: "StoryCommentsSectionFragment_comments")
    {
      ...
  }
`;
```

その間、引数`data.id`は、接続する特定のストーリーのIDです。

この変更で、ミューテーションが一旦完了したら、コメントのリスト内にコメントが現れるのを確認するはずです。

## まとめ

ミューテーションは、変更することをサーバーに問い合わせさせます。

- ちょうどクエリのように、ミューテーションはフィールドで構成され、変数を受け付け、そしてフィールドへの引数としてそれらの変数を渡します。
- ミューテーションによって先tなくされたフィールドは、ストアを更新するために使用できる*ミューテーションのレスポンス*で構成されています。
- 自動的にRelayは、ストア内でIDが一致したノードとレスポンス内のノードをマージします。
- `@appendEdge`、`prependEdge`そして`@deleteEdge`ディレクティブは、ミューテーションのレスポンスからストア内のコネクションに、アイテムを挿入そして削除してくれます。
- アップデーターは、シスドアを手動で操作させてくれます。
- 楽観的アップデータは、ミューテーションが開始される前に実行され、ミューテーションが完了したときロールバックされます。

# 配列とリスト

<https://relay.dev/docs/tutorial/arrays-lists/>

- [配列とリスト](#配列とリスト)
  - [ステップ1 - フラグメントでリストを選択する](#ステップ1---フラグメントでリストを選択する)
  - [ステップ2 - コンポーネントでリストをマップする](#ステップ2---コンポーネントでリストをマップする)
  - [ステップ3 - ノードIDに基づいたReactキーを追加する](#ステップ3---ノードidに基づいたreactキーを追加する)

これまで、コンポーネントを構成するコンポーネントの単一インスタンスを持つコンポーネントのみを扱ってきました。
例えば、1つのニュースフィードストーリー、そして、そのストーリにある、1つのプロファイル画像を持つ1つの著者のみを表示しています。
1つより多い何かを処理する方法を確認しましょう。

GraphQLは配列をサポートを含んでおり、GraphQLにおいて配列は*リスト*と呼ばれます。
フィールドは1つのスカラー値だけになり得るだけでなくその配列に、または1つのエッジだけでなくエッジの配列になり得ます。
スキーマは、どのフィールドがリストでそうでないか指定しませんが、奇妙なことに、GraphQLクエリ構文は、単一フィールドの選択とリストの選択を区別しません。
すぐにわかる設計原則の例外は、GraphQLレスポンスは、クエリと同じ形状を持つべきであるということです。

次のリクエストは・・・

```graphql
query MyQuery {
  viewer {
    contacts {  // エッジのリスト
      id    // フィールドの単一アイテム
      name
    }
  }
}
```

次のレスポンスを生成します。

```json
{
  "viewer": {
    "contacts": [   // レスポンス内の配列
      {
        "id": "123",
        "name": "Chris",
      },
      {
        "id": "789",
        "name": "Sue",
      }
    ]
  }
}
```

たまたま、アプリ例のスキーマは、ストーリーのリストを返す`topStories`フィールドを持ちますが、現在、使用している`topStories`が単に1つだけ返すものを使用していることと対照的です。

ニュースフィードに複数のストーリーを表示するために、`topStories`を使用するように`Newsfeed.tsx`を修正する必要があります。

## ステップ1 - フラグメントでリストを選択する

`Newsfeed.tsx`を開き、`NewsfeedQuery`を見つけてください。
`topStory`を`topStories`に置き換えてください。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    topStories {    // topStory -> topStories
      ...StoryFragment
    }
  }
`;
```

## ステップ2 - コンポーネントでリストをマップする

これにより、`Newsfeed`コンポーネントにおいて、`data.topStories`はフラグメントを参照した配列になり、それぞれはそのストーリーをレンダリングする子コンポーネントである`Story`に渡されます。

```tsx
export default function Newsfeed({}) {
  const data = useLazyLoadQuery<NewsfeedQueryType>(NewsfeedQuery, {});
  const stories = data.topStories;
  return (
    <div className="newsfeed">
      {stories.map(story => <Story story={story} />)}
    </div>
  );
}
```

## ステップ3 - ノードIDに基づいたReactキーを追加する

この時点で、画面に複数のストーリーを確認できるはずです。
適切なニュースフィードアプリのように見え始めています。

![multiple stories](https://relay.dev/assets/images/arrays-top-stories-screenshot-ba4541cbe06c7d58e74097e07d065508.png)

しかし、[キー属性を提供する](https://reactjs.org/docs/lists-and-keys.html)ことなしに、コンポーネントの配列をレンダリングしているため、Reactの警告を得ます。

![list should have a unique "key" prop](https://relay.dev/assets/images/arrays-keys-warning-screenshot-4e615af12bcb73307b06f9fcb09bf6e2.png)

常に、この警告に注意することは重要で、より具体的には、単純な配列内のインデックスよりも、表示されているモノの識別子に基づいたキーにすることが重要です。
これにより、Reactは順序が変わっても、どの項目がどの項目であるかを認識できるため、リストのアイテムの再整列と削除が正しく処理されるようになります。

幸い、一般的に、GraphQLノードはIDを持っています。
単純に`story`の`id`フィールドを選択して、それをキーとして使用します。

```tsx
const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    topStories {
      id    // ここを追加
      ...StoryFragment
    }
  }
`;

...

export default function Newsfeed({}) {
  const data = useLazyLoadQuery<NewsfeedQueryType>(NewsfeedQuery, {});
  const stories = data.topStories;
  return (
    <div className="newsfeed">
      {stories.map(story => (
        <Story
          key={story.id}  // ここを追加
          story={story}
        />
      )}
    </div>
  );
}
```

それを使用することで、画面にストーリーのコレクションを得られました。
ここで、フラグメントの展開と個別のフィールドをクエリの同じ場所で混ぜていることに注目することに価値があります。
これは、ニュースフィードが、それが注意するフィールドを直接`useLazyLoadQuery`から読み込める一方で、ストーリーはそれが注意するフィールドを`useFragment`を介して読み込めることです。
*同じオブジェクト*は両方とも、ニュースフィードの選択された`id`フィールドを含んでおり、それは`StoryFragment`のためのフラグメントキーでもあります。

> **💡: TIP**
> GraphQLのリストはモノのコレクションを扱う最も基本的な方法にすぎません。
> 後のチュートリアルで、コネクションと呼ばれる特別なシステムを使用して、ページ分けしたり無限スクロールするためにそれらを構築します。
> アイテムのコレクションを持つほとんどの状況で、コネクションを使用したいと考えるでしょう。
> ただし、構成要素として、まだGraphQLのリストを使用します。

先に進みましょう。

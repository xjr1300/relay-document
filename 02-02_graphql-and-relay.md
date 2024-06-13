# GraphQLとRelay

<https://relay.dev/docs/tutorial/graphql/>

- [GraphQLとRelay](#graphqlとrelay)
  - [まとめ](#まとめ)

このセクションは、GraphQL、React、そしてスタックの他の部分と関連したRelayの位置付けの概要を説明します。
詳細をすべて理解することを心配する必要はありません。
要点を理解してから、コードで作業を開始するために次のセクションに進みます。
もっとより特有なことは、チュートリアルを通じて例で作業しながら説明されます。

---

GraphQLは、サーバー上のデータを取得して修正する言語です。
GraphQLの特徴は、固定された集合のAPIエンドポイントを持つのではなく、サーバーはクライアントが必要かもしれないデータの任意の組み合わせをリクエストするためにクライアントが使用できる選択肢のパレットを提供します。
これは、データの要望が変化することで、新しいエンドポイントを記述そしてデプロイする必要がないため、フロントエンドの開発者をより迅速に作業を進めることができます。
また、それはクライアントの新しいバージョンがリリースされたとき、古いバージョンとの互換性のために追加のフィールドを残すことなく、単に必要なデータをリクエストできることを意味します。

GraphQLは、任意の種類のバックエンド間でデータをクエリする統一したインターフェイスを提供します。
データが、リレーショナルSQLデータベース、グラフ指向データベース、またはマイクロサービスの大艦隊のいずれかにあっても、GraphQLサーバーは複数のバックエンドからデータを収集して、単一のレスポンスでクライアントにデータを送信できます。
それは、クライアントからそれぞれのサービスにクエリを分離して発行するよりも効率的です。

従来からのHTTPのAPIにおいて、固定された情報の集合でそれぞれ応答するURLがあります。

```text
Request:
GET /person?id=24601

Response:
{"id": "24601", "name": "Jean Valjean", "age": 64, "occupation": "Mayor"}
```

GraphQLにおいて、クライアントは望む特定の情報を問い合わせして、単にサーバーはリクエストされた情報に応答します。

```text
Request:
query {
  person(id: "24601") {
    name
    occupation
  }
}

Response:
{
  "person": {
    "name": "Jean Valjean",
    "occupation": "Mayor"
  }
}
```

クライアントがリクエストした特定のフィールドのみが、レスポンスに含まれていることに注意してください。

名前が示唆するように、GraphQLは*グラフ*にデータを構造化します。
グラフは、オブジェクトまたはレコードのような*ノード*と、あるノードからほかへのポインタのような*エッジ*で構成されます。

![nodes and edges in graph](https://relay.dev/assets/images/graphql-graph-detail-f6e827091095e5065533a8da93632e1c.png)

GraphQLは、あるノードから他のノードに向かうエッジに従い、訪問するそれぞれのノードの情報を問い合わせさせます。
例えば、個々に、人から彼らの市区町村に向かい、市区町村についての情報を得ます。

```graphql
query {
  person(id: "24601") {
    name
    occupation
    location {
      name
      population
    }
  }
}
```

```json
{
  "person": {
    "name": "Jean Valjean",
    "occupation": "Mayor",
    "location": {
      "name": "Montreuil-sur-Mer",
      "population": 1935
    }
  }
}
```

![a person and their city](https://relay.dev/assets/images/graphql-response-0e5fa1ece4cac735173d89a8cfcd5c73.png)

これは、1つのクエリですべてのオブジェクトの立派な装い全体についての情報を取得できることを意味します。

```text
This means we can retrieve information about a whole panoply of objects all in one query
```

言い換えれば、多くのリクエストを次々に送信する代わりに、単一のリクエストで画面に必要なすべてのデータを効率的に取得できます。
また、UIのそれぞれの画面ごとに、個別なエンドポイントを記述して、維持することなく、これを成し遂げます。

代わりに、GraphQLサーバーはスキーマと提供して、そのスキーマはどのような種類のノードがある、どのようにそれらが接続しているか、そしてそれぞれのノードが含む情報はなにかを説明します。

このチュートリアルにおけるアプリ例は、ニュースフィードアプリであるため、そのスキーマは次のような型で構成されます。

- `Story` - ニュースフィードストーリを表現して、それはそのタイトル、イメージ、それを投稿した人または組織へのへの*エッジ*のようなフィールドを持ちます。
- `Person` - 彼らの名前、Eメール、そして他の人へのエッジで友人のリストのような情報を持ちます。
- `Viewer` - そのアプリを閲覧している人を表現して、ニュースフィードストーリのリストのような情報を持ちます。
- `Image` - それ自身のイメージへのURLと、同様に`alt`テキストドキュメントを持ちます。

GraphQL言語は、型システムとスキーマを記述する言語を含んでいます。
ここに、アプリ例のスキーマ定義のスニペットを示します。
すべての詳細を理解する心配をしないでください。
それは単なる一般的な考えを与えるためのものです。

```graphql
# ニュースフィードストーリです。それはフィールドを持ち、それらのいくつかはstringやnumberなど
# のスカラーで、'thumbnail`と'poster'フィールドのようないくつかはグラフの他のノードへのエッジです。
type Story {
  id: ID!
  category: Category
  title: String
  summary: String
  thumbnail: Image
  poster: Actor
}

# Actorは、そのサイトで何らかの事ができるエンティティです。
# これは、複数の異なる方を実装できるインターフェイスで、この場合PersonとOrganizationです。
interface Actor {
  id: ID!
  name: String
  profilePicture: Image
}

# これはそのインターフェイスを実装する特別な型です。
# Actorインターフェイスを実装するPerson型です。
type Person implements Actor {
  id: ID!
  name: String
  email: String
  profilePicture: Image
  location: Location
}

# スキーマは、ニュースフィードストーリのカテゴリのような列挙型も定義できます。
enum Category {
  EDUCATION
  NEWS
  COOKING
}
```

クエリ以外にも、GraphQLは、そのデータを更新するためにサーバーに問い合わせする*ミューテーション*を送信できます。
クエリは、HTTPのGETリクエストに類似していて、そしてミューテーションはPOSTリクエストと同等です。
また、GraphQLは、開いたコネクションで、リアルタイムな更新を指せる*サブスクリプション*も持っています。

通常、GraphQLはHTTP上に実装されるため、クエリとミューテーションはGETとPOSTに似ているだけではありませんが、そのまま送信することもできます。

---

現在、GraphQLについて話した、Relayについて話しましょう。
Relayは、いくつか異なる部分をコードにダイブする前に簡潔に説明します。

Relayは、GraphQLを中心としたクライアント向けのデータ管理ライブラリですが、Relayから最大限に引き出す非常に特殊な方法でRelayを使用します。

最高のパフォーマンスを得るには、個別のコンポーネントが独自のリクエストを発行するのではなく、アプリが各画面またはページの開始で単一のリクエストを発行するようにします。
しかし、これに伴う問題は、それがコンポーネントと画面を一緒に連結して、大きな保守性の問題を作成します。
特定のフィールドを削除する必要がある場合、再度すべてのクエリからそのフィールドを削除する必要があります。
しかし、この場合、フィールドが*他の*コンポーネントによって使用されていないことを革新できますか？
それは、これら大きな画面と広大なクエリを維持することがとても困難になります。

Relayのユニークな強さは、それぞれのコンポーネントがそれ独自のデータ要求をローカルに宣言させることにより、このトレードオフを回避することです。
このように、性能と保守性の両方を得ます。

Relayは、JavaScriptコードをスキャンしてGraphQLのフラグメントを探し、それらフラグメントを結合して完全なクエリを作成するコンパイラを使用して、これを行います。

![the relay complier](https://relay.dev/assets/images/graphql-compiler-combines-fragments-efd0a81c2506eff9c4c9f0370cce14b4.png)

コンパイラを除いて、RelayはGraphQLの取得と処理を管理するランタイムコードを持っています。
それは、取得されたデータすべてを格納する`Store`と呼ばれるローカルなキャッシュを維持して、それぞれのコンポーネントにそれに属するデータを配布します。

~[the store and components](https://relay.dev/assets/images/graphql-relay-runtime-fetches-query-4f0734093c2d277f1dbe5135c5a519ba.png)

中央集権的な`Store`を持つ利点は、それが更新されたときにデータの一貫性を維持させることです。
例えば、UIが名前を編集する方法がある場合、それを1箇所で更新でき、たとえ、それらが異なる画面にあり、初期にデータを取得するために異なるクエリを使用したとしても、その人の名前を表示するすべてのコンポーネントに新しい情報が表示されます。
これは、Relayがデータを受診するときにデータを*正規化する*ためで、単一のグラフノードで表示されるすべてのデータが1箇所にマージされるため、同じノードの複数のコピーが存在しないことを意味します。

確かに、Relayは単にデータをクエリせず、それは、楽観的な更新とロールバックのサポートを含め、クエリと更新のライフサイクル全体を提供します。
ページ分け、データのリフレッシュができ、Relayは効率的にそれ特有のデータを表示するコンポーネントを再レンダリングします。

## まとめ

GraphQLはデータをグラフとしてモデリングして、サーバーからそのデータをクエリするための言語です（同様にデータを更新します）。
Relayは、それぞれのReactコンポーネントと同じ場所に配置されている個々のフラグメントからクエリを構築できるGraphQL用のReactベースのクライアントライブラリです。
一度、データがクエリされれば、Relayは一貫性を維持して、データが更新された場合にコンポーネントを再レンダリングします。
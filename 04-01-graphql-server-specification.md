# GraphQLサーバー仕様

<https://relay.dev/docs/guides/graphql-server-specification/>

- [GraphQLサーバー仕様](#graphqlサーバー仕様)
  - [序文](#序文)
  - [スキーマ](#スキーマ)
  - [オブジェクトの識別](#オブジェクトの識別)
  - [コネクション](#コネクション)
  - [参考資料](#参考資料)

このドキュメントの目的は、RelayがGraphQLサーバーについて行う仮定を記述して、GraphQLスキーマの例を通じてそれらを説明することです。

コンテンツは次のとおりです。

- 序文
- スキーマ
- オブジェクトの識別
- コネクション
- 参考文献

## 序文

RelayがGraphQLサーバーについて行う主要な2つの仮定は次のとおりです。

1. オブジェクトを再取得する機構
2. コネクションを通じてページ分けする方法の説明

この例は、これら仮定の2つすべてを説明します。
この例は、包括的ではありませんが、これらの主要な仮定を簡単に説明して、ライブラリのより細かな仕様に入り込む前に、何らかの文脈を提供するために設計されています。

例の前提は、オリジナルのスター・ウォーズ3部作の船と派閥に関する情報をクエリするために、GraphQLを使用したいと考えています。

また、例は、読者がすでに[スターウォーズ](https://en.wikipedia.org/wiki/Star_Wars)に親しんでいることを想定しています。
もし、そうでないなら、スターウォーズの1977年版バージョンが開始することが良いですが、この文書の目的のために1977年版特別エディションが役立ちます。

## スキーマ

下に記述されたスキーマは、Relayによって使用されるGraphQLサーバーが実装するべき機能を説明するために使用されます。
2つの主要な型は、スター・ウォーズの世界にある派閥と船で、派閥はそれに関連する船を多く持っています。

```graphql
# ノード型
interface Node {
  id: ID!         # ノードID
}

# 派閥（反乱軍と帝国軍を表現）
type Faction implements Node {
  id: ID!
  name: String
  ships: ShipConnection # 派閥が保有する船へのコネクション
}

type Ship implements Node {
  id: ID!
  name: String
}

type ShipConnection {
  edges: [ShipEdge]    # 船似向かうエッジ
  pageInfo: PageInfo!  # ページ情報
}

type ShipEdge {
  cursor: String!     # カーソル
  node: Ship          # 船
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# ルートクエリ型
type Query {
  rebels: Faction     # 反乱軍
  empire: Faction     # 帝国
  node(id: ID!): Node # nodeルートフィールド
}
```

## オブジェクトの識別

`Faction`と`Ship`両方は、それらを再取得するために使用できる識別子を持っています。
`Node`インターフェイスとルートクエリ型の`node`フィールドを介して、Relayにこの能力を公開します。

`Node`インターフェイスは、`id`という単一のフィルドを含んでおり、それは`ID!`型です。
`node`ルートフィールドは、`ID!`型の単一の引数を受け取り、`Node`型を返します。
これら2つの機能は、協力して再取得できるようにします。
もし、そのフィールド(`node`フィールド)で返された`id`を`node`フィールドに渡すと、オブジェクトが返されます。

反乱軍のIDを問い合わせて、実際にこれを確認しましょう。

```graphql
query RebelsQuery {
  rebels {
    id
    name
  }
}
```

次が返されます。

```json
{
  "rebels" {
    "id": "RmFjdGlvbjox",
    "name": "Alliance to Restore the Republic"
  }
}
```

これにより、現在、システム内の反乱軍のIDを知りました。
これにより、それらを再取得できます。

```graphql
query RebelsRefetchQuery {
  node(id: "RmFjdGlvbjox") {
    id
    ... on Faction {
      name
    }
  }
}
```

次が返されます。

```json
{
  "node" {
    "id": "RmFjdGlvbjox",
    "name": "Alliance to Restore the Republic"
  }
}
```

もし、帝国で同じことをした場合、クエリが異なるIDを返すことを発見するでしょう。
そして、同様に帝国を再取得することができます。

```graphql
query EmpireQuery {
  empire {
    id
    name
  }
}
```

次が生まれます。

```json
{
  "empire": {
    "id": "RmFjdGlvbjoy",
    "name": "Galactic Empire"
  }
}
```

そして、次で帝国を再取得できます。

```graphql
query EmpireRefetchQuery {
  node(id: "RmFjdGlvbjoy") {
    id
    ... on Faction {
      name
    }
  }
}
```

次が生まれます。

```json
{
  "empire": {
    "id": "RmFjdGlvbjoy",
    "name": "Galactic Empire"
  }
}
```

`Node`インターフェイスと`node`フィールド（の`id`フィールドの値）は、この再取得のためにグローバルに一意なIDであることを想定しています。
グローバルに一意なIDがないシステムは、普通、型と型特有のIDを組み合わせることによってそれら(`id`)を合成して、それがこの例で行われたことです。

返されたIDは、base64文字列です。
IDは不透明に設計されており、base64でエンコードされた文字列は、その文字列が不透明な識別子であることを見ている人に思い出させる、GraphQLにおける便利な慣例です。
`node`の`id`引数に渡されるべき唯一のものは、システム内の同じオブジェクトのIDをクエリした変更されていない結果です。

サーバーがするべき振る舞いの完全な仕様は、GraphQLサイト内の[GraphQLオブジェクトの識別](https://graphql.org/learn/global-object-identification/)に記述されており、それは最善の実践ガイドです。

## コネクション

スター・ウォーズの世界で反乱軍は多くの船を持っています。
Relayは、1対多の関連を表現する標準化された方法を使用して、簡単に1対多の関連を操作する機能を含んでいます。
この標準化ｓれた接続モデルは、コネクションを介してスライスまたはページ分けする方法を提供しています。

反乱軍を取り上げ、反乱軍の最初の船を質問しましょう。

```graphql
query RebelsShipsQuery {
  rebels {
    name
    ships(first: 1) {
      edges {
        node {
          name
        }
      }
    }
  }
}
```

次が生まれます。

```json
{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": [
        {
          "node": {
            "name": "X-Wing"
          }
        }
      ]
    }
  }
}
```

上記クエリは、最初の船に結果セットをスライスするために、`ships`に渡す`first`引数を使用しました。
しかし、クエリを介してページ分けしたい場合は何をすればいいのでしょうか？
それぞれのエッジには、ページ分けするために使用できるカーソルが公開されています。
次は、最初の2つを質問して、同様にカーソルも取得しましょう。

```graphql
query MoreRebelShipsQuery {
  rebels {
    name
    ships(first: 2) {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

そして、次が得られます。

```json

{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjA=",
          "node": {
            "name": "X-Wing"
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjE=",
          "node": {
            "name": "Y-Wing"
          }
        }
      ]
    }
  }
}
```

カーソルはbase64文字列であることに注意してください。
base64文字列は前に説明したパターンです。
サーバーは、カーソルが不透明な文字列であることを思い出させています。
`ships`フィールドの`after`引数として、返されたカーソルの文字列を渡すことができ、それにより前の結果の最後の1つより後の次の3隻の船を質問することができます。

```graphql
query EndOfRebelShipsQuery {
  rebels {
    name
    ships(first: 3, after: "YXJyYXljb25uZWN0aW9uOjE=") {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

次が与えられます。

```json


{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjI=",
          "node": {
            "name": "A-Wing"
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjM=",
          "node": {
            "name": "Millenium Falcon"
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjQ=",
          "node": {
            "name": "Home One"
          }
        }
      ]
    }
  }
}
```

すばらしい！
続けて次の4隻を得ましょう！

```graphql
query RebelsQuery {
  rebels {
    name
    ships(first: 4, after: "YXJyYXljb25uZWN0aW9uOjQ=") {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

次が生まれます。

```json
{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": []
    }
  }
}
```

ふむ。
これ以上船はありません。
システム内に反乱軍の船はたった5隻しかないと推測できます。
これ以上船がないことを確認するために他のラウンドトリップをすることなく、コネクションの末尾に到達していたことがわかればよかったです。
コネクションモデルは、`PageInfo`と呼ばれる型を使用して、この能力を公開します。
よって、再度船を取得した2つのクエリを、今回は`hasNextPage`を質問して発行しましょう。

```graphql
query EndOfRebelShipsQuery {
  rebels {
    name
    originalShips: ships(first: 2) {
      edges {
        node {
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
    moreShips: ships(first:3 after: "YXJyYXljb25uZWN0aW9uOjE=") {
      edges {
        node {
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```

そして、次を得ます。

```json
{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "originalShips": {
      "edges": [
        {
          "node": {
            "name": "X-Wing"
          }
        },
        {
          "node": {
            "name": "Y-Wing"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true
      }
    },
    "moreShips": {
      "edges": [
        {
          "node": {
            "name": "A-Wing"
          }
        },
        {
          "node": {
            "name": "Millenium Falcon"
          }
        },
        {
          "node": {
            "name": "Home One"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": false
      }
    }
  }
}
```

船を取得する最初のクエリで、GraphQLは次のページがあることを伝えましたが、次のクエリはコネクションの末尾に到達したことを伝えています。

Relayは、コネクションのすべての機能を使用して、接続に関する抽象化を構築して、クライアントがカーソルを手動で管理する必要をなくして、簡単にコネクションを効率的に機能するようにします。

サーバーが振る舞うべき完全な詳細は、[GraphQLカーソルコネクション](https://relay.dev/graphql/connections.htm)仕様にあります。

## 参考資料

これは、GraphQLサーバーの仕様の概要を結論付けています。
Relayに準拠したGraphQLサーバーの詳細な要求は、より公式な説明の[Relayカーソルコネクション](https://relay.dev/graphql/connections.htm)モデル、[GraphQLグローバルオブジェクト識別](https://graphql.org/learn/global-object-identification/)モデルにあります。

仕様を実装したコードを見るために、[GraphQL.js Relayライブラリ](https://github.com/graphql/graphql-relay-js)は、ノードとコネクションを作成するヘルパー関数を提供しています。
そのリポジトリの[__tests__](https://github.com/graphql/graphql-relay-js/tree/main/src/__tests__)フォルダは、リポジトリの統合テストとして上記の例の実装を含んでいます。

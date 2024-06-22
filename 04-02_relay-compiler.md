# Relayコンパイラー

<https://relay.dev/docs/guides/compiler/>

- [Relayコンパイラー](#relayコンパイラー)
  - [graphql](#graphql)
  - [コンパイラー](#コンパイラー)
    - [GraphQLスキーマ](#graphqlスキーマ)
    - [コンパイラーの実行](#コンパイラーの実行)
    - [生成された定義のインポート](#生成された定義のインポート)

## graphql

Relayによって提供された`graphql`テンプレートタグは、[GraphQL](http://graphql.org/learn/)言語のクエリ、フラグメント、ミューテーションそしてサブスクリプションを記述するための機構として提供されています。
例えば次のように使用します。

```js
import {graphql} from 'react-relay';

graphql`
  query MyQuery {
    viewer {
      id
    }
  }
`;
```

`graphql`テンプレートタグを使用した結果は、GraphQLドキュメントのランタイム表現である`GraphQLTaggedNode`です。

`graphql`テンプレートタグのノードは、**決してランタイムで実行されません**。
代わりに、それらは、事前にRelayコンパイラーによって、ソースコードと一緒に存在して、ランタイムでRelayが操作するために要求する人工物にコンパイルされます。

## コンパイラー

Relayは、[graphql](https://relay.dev/docs/guides/compiler/#graphql)リテラルをソースファイルと一緒に存在する生成されたファイルに変換されます。

次のようなフラグメントは・・・

```js
graphql`
  fragment MyComponent on Type {
    field
  }
`
```

ランタイムの人工物と型安全な記述を支援する[Flow型](https://flow.org/)の両方を持つ、`./__generated__/MyComponent.graphql.js`ファイルを生成します。

Relayコンパイラーは、ビルドステップの一部として、ランタイムで参照されるコードを生成する責任があります。
事前にクエリをビルドすることによって、Relayのランタイムは、クエリ文字列を生成する責任がなくなり、そしてランタイムでとても実行コストが高いクエリに対して様々な最適化を実行できます。
例えば、クエリ内で重複したフィールドは、GraphQLレスポンスの処理の効率を改善するために、ビルドステップでマージされます。

### GraphQLスキーマ

Relayコンパイラーを使用するために、GraphQLサーバーのAPIを記述する`.graphql`（を拡張子を持つ）[GraphQLスキーマ](https://graphql.org/learn/schema/)ファイルが必要です。
典型的に、これらのファイルは、サーバーの真実の源のローカルな表現で、直接編集されません。
例えば、次のような`schema.graphql`ファイルを持つかもしれません。

```graphql
schema {
  query: Root
}

type Root {
  dictionary: [Word]
}

type Word {
  id: String!
  definition: WordDefinition
}

type WordDefinition {
  text: String
  image: String
}
```

### コンパイラーの実行

さらに、GraphQLクエリとフラグメントを説明するために`graphql`タグを使用した`.js`ファイルを含むディレクトリが必要です。
このディレクトリを`./src`と呼びましょう。

そして、事前準備として、`yarn run relay`を実行してください。

これは、`graphql`タグを含む対応したファイルと同じ場所に配置される一連の`__generated__`ディレクトリを作成します。

例えば、次の2つのファイルを与えると・・・

- `src/Components/DictionaryComponent.js`

```js
const DictionaryWordFragment = graphql`
  fragment DictionaryComponent_word on Word {
    id
    definition {
      ...DictionaryComponent_definition
    }
  }
`;

const DictionaryDefinitionFragment = graphql`
  fragment DictionaryComponent_definition on WordDefinition {
    text
    image
  }
`;
```

- `src/Queries/DictionaryQuery.js`

```js
const DictionaryQuery = graphql`
  query DictionaryQuery {
    dictionary {
      ...DictionaryComponent_word
    }
  }
`;
```

これは、3つの生成したファイルと2つの`__generated__`ディレクトリを生産します。

- `src/Components/__generated__/DictionaryComponent_word.graphql.js`
- `src/Components/__generated__/DictionaryComponent_definition.graphql.js`
- `src/Queries/__generated__/DictionaryQuery.graphql.js`

### 生成された定義のインポート

通常に、生成された定義をインポートする必要はありません。
[Relay Babelプラグイン](https://relay.dev/docs/getting-started/installation-and-setup/#setup-babel-plugin-relay)は、コード内の`graphql`リテラルを、生成されたファイルを呼び出す`require()`に変換します。

しかし、またRelayコンパイラーは、自動的に[型コメント](https://flow.org/en/docs/types/comments/)と呼ばれる[Flow](https://flow.org/)型を生成します。
例えば、次のように生成されたFlow型をインポートできます。

```js
import type {DictionaryComponent_word} from './__generated__/DictionaryComponent_word.graphql';
```

とても稀ですが、クエリ、ミューテーション、フラグメントまたはサブスクリプションに複数のファイルからアクセスする必要があるかもしれません。
この場合、それを直接インポートできます。

```js
import DictionaryComponent_word from './__generated__/DictionaryComponent_word.graphql`;
```

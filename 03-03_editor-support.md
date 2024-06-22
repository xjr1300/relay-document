# エディタのサポート

<https://relay.dev/docs/editor-support/>

- [エディタのサポート](#エディタのサポート)
  - [言語サーバー](#言語サーバー)
  - [なぜ、Relay特有のエディタ拡張が必要なのか？](#なぜrelay特有のエディタ拡張が必要なのか)

*TL;DR: [VS Code拡張](https://marketplace.visualstudio.com/items?itemName=meta.relay)があります。*

---

Relayコンパイラーは、コードに埋め込まれたGraphQLを深く理解します。
Relayでアプリを記述する開発者の体験を改善するために、その理解を使用したいと考えています。
よって、[v14.0.0](https://github.com/facebook/relay/releases/tag/v14.0.0)で開始された新しいRustのRelayコンパイラーは、コードエディタに直接言語機能を提供することができます。
これは、次を意味します。

**Relayコンパイラーエラーは、エディタ内に直接赤い波線で表示されます。**

![relay compiler shows red squiggles in editor](https://relay.dev/img/docs/editor-support/diagnostics.png)

**GraphQLでタグ付けされたテンプレートリテラルを自動補完します。**

![relay compiler autocomplete graphql template literals](https://relay.dev/img/docs/editor-support/autocomplete.png)

**ホバーすると、型情報と、Relay特有の機能についてのドキュメントを表示します。**

![relay compiler shows type info and docs on hover](https://relay.dev/img/docs/editor-support/hover.png)

**`@deprecated`フィールドは取り消し線を使用してレンダリングされます。**

![relay compiler renders deprecated fields with strike-through](https://relay.dev/img/docs/editor-support/deprecated.png)

**フラグメントの定義をクリックすると、フィールドと型を確認できます。**

![relay compiler shows fields and types when clicking on fragment definition](https://relay.dev/img/docs/editor-support/go-to-def.gif)

**一般的なエラーを修正する提案を表示します。**

![relay compiler suggests fixes for common errors](https://relay.dev/img/docs/editor-support/code-actions.png)

## 言語サーバー

エディタのサポートは、[言語サーバープロトコル](https://microsoft.github.io/language-server-protocol/)を使用して実装されているため、さまざまなエディターで使用できますが、このリリースと並行して、[Coinbase](https://www.coinbase.com/)の [Terence Bezman](https://twitter.com/b_ez_man)が公式のVS Code拡張機能を提供しました。

[ここにあります。](https://marketplace.visualstudio.com/items?itemName=meta.relay)

## なぜ、Relay特有のエディタ拡張が必要なのか？

GraphQLファウンデーションには、公式の言語サーバーと、GraphQLをサポートするエディタを提供する[VS Code拡張](https://marketplace.visualstudio.com/items?itemName=GraphQL.vscode-graphql)があります。
これにより、優れた体験のベースラインを提供できますが、Relayユーザーにとって、Relayコンパイラーからこの情報を直接得ると、次のような多くの利点が得られます。

- Relayコンパイラーエラーは、"問題"としてエディタ内に直接表示され、時々簡単な修正を提案します。
- ホーバーされた情報は、Relay特有の機能とディレクティブを認識して、関連するドキュメントにリンクできます。

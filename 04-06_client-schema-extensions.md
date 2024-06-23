# クライアントスキーマ拡張

<https://relay.dev/docs/guides/client-schema-extensions/>

- [クライアントスキーマ拡張](#クライアントスキーマ拡張)
  - [目次](#目次)
  - [サーバースキーマを拡張する](#サーバースキーマを拡張する)
  - [ローカル状態をクエリする](#ローカル状態をクエリする)
  - [ローカル状態を変更する](#ローカル状態を変更する)
    - [作成](#作成)
    - [更新](#更新)
    - [削除](#削除)
  - [初期ローカル状態](#初期ローカル状態)

> **ℹ NOTE**
> ガイドツアーの[ローカルデータの更新](https://relay.dev/docs/guided-tour/updating-data/local-data-updates/)と[クライアントのみのデータ](https://relay.dev/docs/guided-tour/updating-data/client-only-data/)のセクションも参照してください。

Relayは、ローカルデータの読み書きするために使用でき、クライアントアプリケーションの*すべての*データの真実の唯一の情報源として行動します。

Relayコンパイラーはスキーマのクライアントサイドの拡張を完全にサポートして、それはローカルなフィールドと型を定義できるようにします。

## 目次

- サーバースキーマを拡張する
- ローカル状態をクエリする
- ローカル状態を変更する
- 初期ローカル状態

## サーバースキーマを拡張する

サーバースキーマを拡張するために、`--src`ディレクトリの内部に、新しい`.graphql`ファイルを作成します。
それを、`./src/clientSchema.graphql`と呼びましょう。
このファイルは、Relay構成の`"schemaExtensions"`で参照されるフォルダ内に存在する必要があります。

このスキーマは、クライアントで問い合わせされるローカルデータを説明します。
またそれは、既存のサーバースキーマを拡張するためにも使用されます。

例えば、`Note`と呼ばれる新しい型を作成できます。

```graphql
type Note {
  id: ID!
  title: String
  body: String
}
```

そして、次に`notes`と呼ばれる`Note`のリストをを使用して、サーバースキーマの型`User`を拡張します。

```graphql
extend type User {
  notes: [Note]
}
```

## ローカル状態をクエリする

ローカルデータにアクセスすることは、GraphQLサーバーに問い合わせすることと違いはありませんが、クエリ内に少なくとも1つのサーバーフィールドを含むことが要求されます。
フィールドはサーバースキーマから取得することも、例えば`__typename`のようなイントロスペクションフィールドのように、スキーマに依存しないものにすることもできます。

ここに、`viewer`フィールドを介して現在の`User`を得るために、それらのフィールドのID、名前、そしてノードのローカルリストと一緒に[useLazyLoadQuery](https://relay.dev/docs/api-reference/use-lazy-load-query/)を使用できます。

```js
// Example.js
import * as React from 'react';
import { useLazyLoadQuery, graphql } from 'react-relay';

const Example = (props) => {
  const data = useLazyLoadQuery(graphql`
    query ExampleQuery {
      viewer {
        id
        name
        notes {
          id
          title
          body
        }
      }
    }
  `, {});
  // ...
}
```

## ローカル状態を変更する

すべてのローカルなデータは[Relayストア](https://relay.dev/docs/api-reference/store/)に存在します。

ローカルな状態の更新は、任意の`updater`関数を使用してすることができます。

`commitLocalUpdate`関数は、これにとって特に理想的なため、ローカル状態への書き込みは、普通ミューテーションの外部で実行されます。

前の例を構築するために、`User`の`notes`のリストから`Note`を追加、更新、そして削除しましょう。

### 作成

```js
import {commitLocalUpdate} from 'react-relay';

let tempID = 0;

function createUserNote(environment) {
  commitLocalUpdate(environment, store => {
    const user = store.getRoot().getLinkedRecord('viewer');
    const userNoteRecords = user.getLinkedRecords('notes') || [];

    // 一意なIDを作成します。
    const dataID = `client:Note:${tempID++}`;

    // 新しいノートレコードを作成します。
    const newNoteRecord = store.create(dataID, 'Note');

    // ユーザーのnotesリストにレコードを追加します。
    user.setLinkedRecords([...userNoteRecords, newNoteRecord], 'notes');
  });
}
```

このレコードは`useLazyLoadQuery`を介して`ExampleQuery`によってレンダリングされるため、クエリデータは自動的に保持され、ガベージコレクションされないことに注意してください。

コンポーネントがローカルデータをレンダリングせず、それを手動で保持したい場合、`environment.retain()`を呼び出すことでできます。

```js
import {createOperationDescriptor, getRequest} from 'relay-runtime';

// レコードを参照するクエリを作成します。
const localDataQuery = graphql`
  query LocalDataQuery {
    viewer {
      notes {
        __typename
      }
    }
  }
`;

// クエリのための操作記述子を作成します。
const request = getRequest(localDataQuery);
const operation = createOperationDescriptor(request, {} /* 変数 */);

// Relayにこの操作を保持するように伝えるので、それによって参照されるデータがガベージコレクションされないようにします。
// この場合、`viewer`にリンクされたすべてのノートが保持されます。
const disposable = environment.retain(operation);

// データがもう必要ない場合、Relayがそれをガベージコレクションしても問題ない場合は、
// 保持を破棄することができます。
disposable.dispose();
```

### 更新

```js
import {commitLocalUpdate} from 'react-relay';

function updateUserNote(environment, dataID, body, title) {
  commitLocalUpdate(environment, store => {
    const note = store.get(dataID);

    note.setValue(body, 'body');
    note.setValue(title, 'title')
  });
}
```

### 削除

```js
import {commitLocalUpdate} from 'react-relay';

function deleteUserNote(environment, dataID) {
  commitLocalUpdate(environment, store => {
    const user = store.getRoot().getLinkedRecord('viewer');
    const userNoteRecords = user.getLinkedRecords('notes');

    // ユーザーのnotesリストからnoteを削除する。
    const newUserNoteRecords = userNoteRecords.filter(x => x.getDataID() !== dataID);

    // ストアからnoteを削除する。
    store.delete(dataID);

    // notesの新しいリストを設定します。
    user.setLinkedRecords(newUserNoteRecords, 'notes');
  });
}
```

## 初期ローカル状態

すべての新しいクライアント型のスキーマのフィールドは、`undefined`値がデフォルトです。
しかし、時々、ローカルデータをクエリする前に初期状態を設定したい場合があります。
ローカル状態を準備するために、`commitLocalUpdate`を介したアップデータ関数を使用できます。

```js
import {commitLocalUpdate} from 'react-relay';

commitLocalUpdate(environment, store => {
  const user = store.getRoot().getLinkedRecord('viewer');

  // ユーザーのnotesを空の配列で初期化します。
  user.setLinkedRecords([], 'notes');
});
```

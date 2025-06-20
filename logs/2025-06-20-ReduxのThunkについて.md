# ReduxのThunkについて

- **学習日**: 2025/06/20
- **学習時間**: 30分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

はい、承知いたしました。この非同期処理のコード、いわゆる**「Thunk（サンク）」**は、Reduxで非同期処理を扱う上で最も基本的かつ重要なパターンです。

なぜこれが必要なのか、そして内部で何が起こっているのかを、ステップ・バイ・ステップで深掘りしていきましょう。

---

### なぜ「Thunk」が必要なのか？ - Reducerの鉄の掟

まず、大前提としてReduxの`reducer`には**「純粋な関数でなければならない」**という鉄の掟があります。これは、`reducer`の内部で以下のような**副作用（Side Effect）**を起こしてはいけない、ということです。

*   API通信（`fetch`など）
*   `localStorage`へのアクセス
*   `Math.random()`や`new Date()`の利用
*   外部の変数の変更

`reducer`は、同じ`state`と`action`を渡されたら、**必ず同じ**新しい`state`を返さなければなりません。しかし、API通信は成功するか失敗するか、いつ結果が返ってくるか分かりません。これでは純粋性を保てません。

では、API通信のような副作用はどこで扱えばいいのか？
その答えこそが、**ミドルウェア（Middleware）**であり、その代表的な実装が**Thunk**なのです。

### Thunkの正体：Actionの代わりに「関数」をdispatchする

通常の`dispatch`は、`{ type: '...' }` という**オブジェクト（Action）**を引数に取ります。

```javascript
dispatch({ type: 'counter/increment' }); // Actionオブジェクトを渡す
```

しかし、Thunkを使うと、`dispatch`にActionオブジェクトの代わりに**「関数」**を渡すことができるようになります。

```javascript
dispatch(fetchUserById(123)); // 関数を呼び出し、その戻り値（＝別の関数）を渡す
```

これを可能にしているのが**`redux-thunk`ミドルウェア**です。このミドルウェアは、`dispatch`されたものが…

*   **オブジェクト（通常のAction）なら**: 何もせず、そのまま`reducer`に渡します。
*   **関数（Thunk）なら**: その関数を`reducer`には渡さず、**横取りして実行します。**

### コードの深掘り：Thunk Action Creatorの解体

それでは、問題のコードを解体してみましょう。

```javascript
// これは「Thunk Action Creator」と呼ばれる関数
export const fetchUserById = (userId) =&gt; {
  // 戻り値はActionオブジェクトではなく、「非同期関数（Thunk）」
  return async (dispatch, getState) =&gt; {
    // ...非同期処理の本体...
  };
};
```

このコードは2つの関数が入れ子になっています。

#### 1. 外側の関数 `fetchUserById = (userId) =&gt; { ... }`

*   **役割**: Thunk本体（内側の関数）を生成し、返すための関数です。「Action Creator」という名前の通り、Actionの代わりとなる「Thunk」を作り出すファクトリーです。
*   **引数**: `userId`のように、非同期処理に必要な情報をコンポーネントから受け取ります。

#### 2. 内側の関数 `async (dispatch, getState) =&gt; { ... }`

*   **これこそがThunkの本体**です。`redux-thunk`ミドルウェアが横取りして実行するのはこの部分です。
*   ミドルウェアは、この関数を実行する際に、2つの非常に強力な引数を渡してくれます。
    *   `dispatch`: なんと、ここでもう一度`dispatch`関数が使えます！これにより、**非同期処理の途中経過を新たなActionとして`dispatch`できる**ようになります。これがThunkの最強の機能です。
    *   `getState`: ストアの現在の状態（`state`）を丸ごと取得できる関数です。例えば、`state`から認証トークンを取り出してAPIリクエストのヘッダーに含める、といったことが可能です。

### 非同期処理の「ライフサイクル」を追跡する

Thunkの最も美しい点は、一つの非同期処理を**3つの状態（ライフサイクル）**に分割し、それぞれのタイミングでActionを`dispatch`できることです。

```javascript
// Thunkの実行開始
async (dispatch, getState) =&gt; {
  // ①【開始】pending: 処理の開始を知らせる
  dispatch({ type: 'users/fetch/pending' });

  try {
    // 実際のAPI通信
    const response = await userAPI.fetchById(userId);

    // ②【成功】fulfilled: 処理の成功と結果を知らせる
    dispatch({ type: 'users/fetch/fulfilled', payload: response.data });
  } catch (error) {
    // ③【失敗】rejected: 処理の失敗とエラー内容を知らせる
    dispatch({ type: 'users/fetch/rejected', payload: error.message });
  }
};
```

これにより、UI側では以下のような状態管理が非常に簡単になります。

*   `pending`を受け取ったら: ローディングスピナーを表示する (`state.loading = true`)。
*   `fulfilled`を受け取ったら: ローディングスピナーを消し、受け取ったデータ（`payload`）を画面に表示する (`state.loading = false`, `state.user = payload`)。
*   `rejected`を受け取ったら: ローディングスピナーを消し、エラーメッセージを表示する (`state.loading = false`, `state.error = payload`)。

### Reducer側での対応

この3つのActionを受け取るために、`reducer`側も対応した`case`を用意しておく必要があります。

```javascript
// usersSlice.js
const initialState = {
  user: null,
  loading: false,
  error: null,
};

function usersReducer(state = initialState, action) {
  switch (action.type) {
    case 'users/fetch/pending':
      return { ...state, loading: true, error: null };
    case 'users/fetch/fulfilled':
      return { ...state, loading: false, user: action.payload };
    case 'users/fetch/rejected':
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}
```

### まとめ

*   **Thunkの目的**: 副作用（API通信など）を`reducer`から分離し、Redux内で非同期処理を扱うため。
*   **Thunkの正体**: `dispatch`と`getState`を引数に取る非同期関数。`redux-thunk`ミドルウェアによって実行される。
*   **Thunkの強力な機能**: 非同期処理の**ライフサイクル（開始、成功、失敗）**に応じて、複数のActionを好きなタイミングで`dispatch`できる。
*   **結果**: コンポーネントは`dispatch(fetchUserById(123))`と呼ぶだけで、裏側の複雑な非同期処理を意識する必要がなくなり、`reducer`は純粋性を保ったまま、非同期処理の結果だけを安全に受け取ることができる。

このパターンは、Redux Toolkitの`createAsyncThunk`という機能で、より簡単かつ定型的に書けるように自動化されていますが、その裏側では全く同じことが行われています。この基本を理解しておくことは、Reduxを使いこなす上で非常に重要です。
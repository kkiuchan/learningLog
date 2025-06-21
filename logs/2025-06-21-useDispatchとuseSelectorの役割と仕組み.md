# useDispatchとuseSelectorの役割と仕組み

- **学習日**: 2025/06/21
- **学習時間**: 15分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

`useDispatch`と`useSelector`、この2つのフックはReduxにおけるコンポーネントとの連携の核です。それぞれの役割と内部の仕組みについて、より深く詳細に解説します。

---

### `useDispatch`: 状態変更を「指令」するフック

`useDispatch`は、コンポーネントからReduxストアに対して状態変更の**「引き金（トリガー）」**を引くための唯一の手段です。

#### 1. 何を返すのか？
`useDispatch`をコンポーネント内で呼び出すと、Reduxストアの**`dispatch`関数そのもの**が返されます。

```jsx
import { useDispatch } from 'react-redux';

function MyComponent() {
  const dispatch = useDispatch(); // このdispatchがストアのdispatch関数
  // ...
}
```
この`dispatch`関数は、アプリケーションのライフサイクルを通じて**参照が安定している**ことが保証されています。つまり、再レンダリングされても`dispatch`関数自体は（通常）再生成されません。これはパフォーマンス上非常に重要で、`useEffect`の依存配列などに安全に含めることができます。

#### 2. どのように機能するのか？
`dispatch`関数の役割は、引数として受け取った`action`オブジェクトを、Reduxのシステム（ストアやミドルウェア）に送り込むことです。

**ステップ・バイ・ステップ:**

1.  **Actionの作成**: まず、`{ type: 'ドメイン/イベント名', payload: ... }`という形式のプレーンなJavaScriptオブジェクト（Action）を用意します。
2.  **Dispatchの実行**: イベントハンドラ（例: `onClick`）内で、作成したActionを引数にして`dispatch(action)`を実行します。
3.  **ミドルウェア（Middleware）へ**: `dispatch`されたActionは、まずミドルウェア（例: `redux-thunk`や`redux-saga`）に渡されます。
    *   **同期Actionの場合**: Actionがただのオブジェクトなら、ミドルウェアはそれを素通しします。
    *   **非同期Action（Thunk）の場合**: もしActionが関数（Thunk）なら、`redux-thunk`ミドルウェアがその関数を実行します。その関数内では、API通信などの非同期処理を行い、処理の各段階（開始、成功、失敗）で新たな同期Actionを`dispatch`できます。

    ```javascript
    // 非同期処理の例（Thunk Action Creator）
    export const fetchUserById = (userId) => async (dispatch, getState) => {
      dispatch({ type: 'users/fetch/pending' }); // ① ローディング開始を指令
      try {
        const response = await userAPI.fetchById(userId);
        dispatch({ type: 'users/fetch/fulfilled', payload: response.data }); // ② 成功を指令
      } catch (error) {
        dispatch({ type: 'users/fetch/rejected', payload: error.message }); // ③ 失敗を指令
      }
    };
    ```
4.  **Reducerへ**: ミドルウェアを通過した同期Actionが、最終的に全ての`reducer`に届けられます。
5.  **状態更新**: 各`reducer`は、受け取ったActionの`type`をチェックし、自身の担当する`state`を更新するかどうかを判断します。

`useDispatch`は、この壮大なデータフローを開始させるための**最初のスイッチ**なのです。

---

### `useSelector`: 状態を「購読」するフック

`useSelector`は、コンポーネントがReduxストアの状態を**「読み取り」**、かつその状態の**「変更を購読（subscribe）」**するためのフックです。

#### 1. 何をするのか？
`useSelector`は引数に**セレクター（selector）関数**を取ります。このセレクター関数は、ストアの`state`全体を受け取り、コンポーネントが必要なデータだけを抽出して返す役割を担います。

```jsx
import { useSelector } from 'react-redux';

function UserDisplay() {
  // state全体から、state.user.nameという特定の値だけを抽出する
  const userName = useSelector((state) => state.user.name);

  return <p>User: {userName}</p>
}
```

#### 2. どのように機能するのか？（ここが重要）
`useSelector`は単にデータを一度読み取るだけではありません。内部的には以下の高度な処理を行っています。

1.  **初回レンダリング時**:
    *   コンポーネントがレンダリングされる際に`useSelector`が呼ばれます。
    *   引数のセレクター関数が実行され、`state`から必要なデータが抽出されます。
    *   コンポーネントは、その抽出された値を使って表示を構築します。
    *   同時に、`react-redux`は「このコンポーネントが、このセレクター関数に依存している」ということを内部的に記録します。

2.  **Actionがdispatchされた後**:
    *   `dispatch`によりストアの状態が更新されると、`react-redux`は**このコンポーネントが使っているセレクター関数を再実行**します。
    *   **前回のセレクターの戻り値**と、**今回のセレクターの戻り値**を、厳密等価演算子（`===`）で比較します。
    *   **もし2つの値が異なる場合**（例: `count`が`0`から`1`に変わった）、`react-redux`は**このコンポーネントに再レンダリングを強制します**。
    *   もし値が同じであれば、再レンダリングは行われません。

この仕組みにより、**コンポーネントは自身が関心を持つデータが変更された場合にのみ、効率的に再レンダリングされる**のです。

#### 3. パフォーマンスに関する注意点
この「戻り値を比較して再レンダリングを判断する」という仕組み上、注意しないと不要な再レンダリングを引き起こします。

**悪い例: 毎回新しいオブジェクトを返すセレクター**
```jsx
// stateが更新されるたびに、たとえuserの値が変わらなくても再レンダリングが走る
const { name, email } = useSelector(state => ({
  name: state.user.name,
  email: state.user.email
}));
```
このコードは、セレクターが毎回`{...}`という新しいオブジェクトを生成して返します。`{a:1} === {a:1}`は`false`なので、`useSelector`は毎回「値が変わった」と判断し、不要な再レンダリングを起こします。

**良い例: 値を個別に取得する**
```jsx
const name = useSelector(state => state.user.name);
const email = useSelector(state => state.user.email);
```
このようにプリミティブな値（文字列、数値など）を個別に取得すれば、値自体が変わらない限り再レンダリングは発生しません。これが`useSelector`の最も基本的な最適化です。

---

### 総括：役割分担の表

| 機能 | `useDispatch` | `useSelector` |
| :--- | :--- | :--- |
| **目的** | 状態の**書き込み** / 変更の**指令** | 状態の**読み取り** / 変更の**購読** |
| **方向** | Component **→** Store | Store **→** Component |
| **戻り値** | `dispatch`関数 | ストアから抽出された**状態の値** |
| **引数** | なし | セレクター関数 `(state) => value` |
| **主な関心事** | 「**いつ**」「**どんな**」Actionを送信するか | 「**どの**」状態を監視し、**効率よく**再描画するか |
| **キーワード** | **Action**, **Thunk**, **副作用**, **指令** | **State**, **Selector**, **購読**, **パフォーマンス** |

この2つのフックを使いこなすことが、`react-redux`をマスターする鍵となります。`useDispatch`で状態変更の「流れ」を作り出し、`useSelector`でその「結果」を効率的に受け取る、という美しい役割分担をぜひ意識してみてください。
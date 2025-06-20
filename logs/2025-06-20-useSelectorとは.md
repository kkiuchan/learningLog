# useSelectorとは

- **学習日**: 2025/06/21
- **学習時間**: 15分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

`useSelector`は、ReactコンポーネントがReduxストアの状態（`state`）から**必要なデータだけを抽出（select）して購読する**ためのフックです。

簡単に言うと、**「ストア全体の中から、このコンポーネントで使いたいデータを取り出す」**ための道具です。

### `useSelector`の主な役割

1.  **状態の読み取り**: Reduxストアに保存されているグローバルな状態にアクセスします。
2.  **必要なデータの選択**: ストアの巨大な状態オブジェクトの中から、コンポーネントが必要とする特定の値だけを効率的に取り出します。
3.  **自動的な再レンダリング**: 選択したデータが更新された場合、コンポーネントにそれを通知し、自動的に再レンダリングをトリガーします。

---

### 具体的な使い方

`useSelector`は引数として**セレクター関数**を受け取ります。この関数は、ストアの`state`全体を引数に取り、コンポーネントが必要とする値を返す純粋な関数です。

```jsx
import { useSelector } from 'react-redux';

function Counter() {
  // useSelectorを使って、stateの中から`counter.value`を取り出す
  const count = useSelector((state) =&gt; state.counter.value);

  return <h1>Count: {count}</h1>;
}

function UserProfile() {
  // stateの中から`user.name`を取り出す
  const userName = useSelector((state) =&gt; state.user.name);
  const userStatus = useSelector((state) =&gt; state.user.status);

  return (
    
      <p>Name: {userName}</p>
      <p>Status: {userStatus}</p>
    
  );
}
```

### `useSelector`の重要なポイント

1.  **セレクター関数は純粋であるべき**:
    セレクター関数は、同じ`state`が渡されたら常に同じ結果を返す必要があります。API呼び出しなどの副作用を含めてはいけません。

2.  **パフォーマンスへの影響**:
    `useSelector`は、セレクター関数が返す値が**前回の値と異なる（参照が異なる）**場合に再レンダリングをトリガーします。

    **悪い例：毎回新しいオブジェクトを返してしまう**
    ```jsx
    // このコンポーネントは、stateが変更されるたびに毎回再レンダリングされてしまう
    const { name, email } = useSelector(state =&gt; ({
      name: state.user.name,
      email: state.user.email
    }));
    ```
    上の例では、`useSelector`が毎回新しいオブジェクト`{ name: ..., email: ... }`を生成するため、たとえ`name`や`email`の値が変わっていなくても、不要な再レンダリングが発生してしまいます。

    **良い例：値を個別に選択する**
    ```jsx
    // nameが変更されたときだけ再レンダリングされる
    const name = useSelector(state =&gt; state.user.name);
    // emailが変更されたときだけ再レンダリングされる
    const email = useSelector(state =&gt; state.user.email);
    ```
    このように値を個別に取り出すか、後述する`shallowEqual`やライブラリ（`reselect`）を使うことで、不要な再レンダリングを防ぎ、パフォーマンスを最適化できます。

### `useDispatch`との関係

*   **`useDispatch`**: 状態を**「書き込む」**ためのフック。`dispatch(action)`で状態変更をリクエストします。
*   **`useSelector`**: 状態を**「読み取る」**ためのフック。`useSelector(state =&gt; ...)`で状態を取得します。

この2つはセットで使われることが多く、Reduxの基本的な「読み書き」操作を担っています。

```jsx
function App() {
  // 書き込み用のdispatchを取得
  const dispatch = useDispatch();
  // 読み取り用のselectorでデータを取得
  const count = useSelector(state =&gt; state.counter.value);

  return (
    
      <p>Count: {count}</p>
      {/* dispatchで状態変更をリクエスト */}
       dispatch({ type: 'counter/increment' })}&gt;
        Increment
      
    
  );
}
```

### まとめ

`useSelector`は、**ReduxストアとReactコンポーネントを繋ぐ重要な橋渡し役**です。コンポーネントが必要とするデータだけを効率的に取り出し、状態の変更に応じてUIを自動で更新する仕組みを提供します。
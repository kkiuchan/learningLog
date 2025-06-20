# reducerとdispatch

- **学習日**: 2025/06/21
- **学習時間**: 20分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容



### Reducerの役割

`reducer`は、「**現在の状態（`state`）**と**指示書（`action`）**を受け取ったら、**次の状態（`newState`）**をこう計算する」という**ルールを定義した関数**です。

```javascript
// reducerは単なる関数
function aReducer(currentState, action) {
  // actionのタイプ（指示内容）に応じて...
  if (action.type === 'ADD_ITEM') {
    // 新しい状態を計算して返す
    return { ...currentState, items: [...currentState.items, action.payload] };
  }
  // ...他の指示...

  // 知らない指示なら、現在の状態をそのまま返す
  return currentState;
}
```

ポイントは、`reducer`自身は能動的に何かをするわけではなく、ただの「計算ルール」であるという点です。

### Dispatchの役割

`dispatch`は、その`reducer`に「この指示書（`action`）で状態を更新してください」と**お願い（通知）をするための送信装置**です。

```jsx
// Reactコンポーネント内
function MyComponent() {
  const dispatch = useDispatch();

  function handleAddItem() {
    // 指示書（action）を作成して...
    const action = { type: 'ADD_ITEM', payload: '新しいアイテム' };
    // dispatchで送信！
    dispatch(action);
  }

  // ...
}
```

`dispatch`が`action`を送信すると、Reduxストアがそれを受け取り、内部で`reducer`を呼び出して新しい状態を計算させます。

### まとめると

ご認識の通りです。

*   **Reducer**: 「この`action`が来たら、こう`state`を変える」という**定義**そのもの。
*   **Dispatch**: その定義にある`action`を「今、実行して！」と**Reducerに（ストア経由で）送信する**ための関数。

この関係性が、Reduxの予測可能で一方向なデータフローの基本となっています。素晴らしい理解度です！
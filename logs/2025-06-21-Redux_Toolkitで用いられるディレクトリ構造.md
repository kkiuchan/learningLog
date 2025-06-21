# Redux Toolkitで用いられるディレクトリ構造

- **学習日**: 2025/06/21
- **学習時間**: 20分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

Thunkを導入する際のディレクトリ構造はプロジェクトの規模や個人の好みにもよりますが、一般的に**関心の分離**と**再利用性**を意識した、スケーラブルなディレクトリ構造が推奨されます。

ここでは、Redux Toolkitでよく使われる**「Ducks」**や**「Re-ducks」**と呼ばれるパターンをベースにした、モダンで実践的なディレクトリ構造を2パターンご紹介します。

---

### パターン1：Features（機能）ベースの構造（推奨）

これが現在最も主流で、**Redux Toolkitが公式に推奨している**アプローチです。特定の機能に関連するファイル（Reducer, Actions, Thunks, Selectorsなど）をすべて同じフォルダにまとめます。

**利点:**
*   **関心事の凝集**: `users`関連のロジックはすべて`features/users`に集まっているので、見通しが良く、修正が楽。
*   **スケーラビリティ**: 新しい機能（例: `products`）を追加する際は、`features/products`フォルダを新しく作るだけで、他の部分に影響を与えない。
*   **再利用と削除が容易**: 機能ごとフォルダをコピーしたり削除したりしやすい。

#### ディレクトリ構造の例

```
src/
├── app/
│   └── store.ts          # Reduxストアの設定（ここで各SliceのReducerを結合）
├── components/
│   └── UserProfile.tsx     # ユーザープロフィールを表示するUIコンポーネント
├── features/
│   ├── users/
│   │   ├── usersAPI.ts     # (任意) ユーザー関連のAPI通信ロジックを分離したファイル
│   │   ├── usersSlice.ts   # ★★★ Thunk, Reducer, Actionsをここにまとめる
│   │   └── UserList.tsx    # (任意) このfeatureに密結合したコンポーネント
│   │
│   ├── counter/
│   │   └── counterSlice.ts # カウンター機能のSlice
│   │
│   └── ... (他の機能)
│
└── ...
```

#### `usersSlice.ts`の中身のイメージ

このファイルに、Thunkを含むユーザー関連のReduxロジックをすべて記述します。
（これはRedux Toolkitを使った記法ですが、基本的な考え方は同じです）

```typescript
// src/features/users/usersSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { userAPI } from './usersAPI'; // APIロジックをインポート

// ① Thunk Action Creatorの定義
export const fetchUserById = createAsyncThunk(
  'users/fetchById', // Actionのtypeプレフィックス
  async (userId, { getState, dispatch }) => {
    // ここがThunkの本体
    const response = await userAPI.fetchById(userId);
    return response.data; // 成功時にreducerのpayloadになる値
  }
);

const initialState = {
  user: null,
  status: 'idle', // 'idle' | 'loading' | 'succeeded' | 'failed'
  error: null,
};

// ② Sliceの作成（Reducer, Action, Stateをまとめたもの）
const usersSlice = createSlice({
  name: 'users',
  initialState,
  reducers: {
    // 同期的なActionはここに書く
    userLoggedOut(state) {
      state.user = null;
    },
  },
  // ③ Thunkのライフサイクルに応じたReducerを定義
  extraReducers: (builder) => {
    builder
      .addCase(fetchUserById.pending, (state, action) => {
        state.status = 'loading';
      })
      .addCase(fetchUserById.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.user = action.payload; // payloadにはThunkが返した値が入る
      })
      .addCase(fetchUserById.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message;
      });
  },
});

export const { userLoggedOut } = usersSlice.actions; // 同期Actionをエクスポート
export default usersSlice.reducer; // Reducerをエクスポートしてstoreで使えるようにする
```

---

### パターン2：Roles（役割）ベースの構造（伝統的）

こちらは伝統的なReduxのディレクトリ構造で、ファイルの「役割」ごとにフォルダを分けます。

**利点:**
*   構造がシンプルで、小規模なプロジェクトでは直感的。
*   Reduxの各要素（`actions`, `reducers`など）の役割が明確になる。

**欠点:**
*   プロジェクトが大きくなると、一つの機能修正のために複数のフォルダを横断する必要があり、非常に手間がかかる。
*   関連ファイルが散らばるため、全体像を把握しにくい。

#### ディレクトリ構造の例

```
src/
├── store/
│   ├── actions/
│   │   ├── userActions.ts    # ★ ユーザー関連のActionとThunk
│   │   └── counterActions.ts
│   │
│   ├── reducers/
│   │   ├── userReducer.ts    # ★ ユーザー関連のReducer
│   │   ├── counterReducer.ts
│   │   └── index.ts          # 全Reducerを結合する（rootReducer）
│   │
│   └── store.ts              # ストアの設定
│
├── api/
│   └── userAPI.ts            # (任意) APIロジック
│
└── ...
```

この場合、`fetchUserById` Thunkは `src/store/actions/userActions.ts` に、それに対応するReducerロジックは `src/store/reducers/userReducer.ts` に記述することになります。

---

### 結論と推奨

**小規模なプロジェクトや学習目的であれば、パターン2（役割ベース）でも問題ありません。** Reduxの各パーツの役割を学ぶには良い構造です。

しかし、**実用的でスケールするアプリケーションを開発するのであれば、圧倒的にパターン1（Featuresベース）をお勧めします。**
関連するコードが一箇所にまとまっていることの恩恵は、開発が進むにつれて非常に大きくなります。

もしこれから実装されるのであれば、ぜひ**パターン1の`features`ディレクトリ構造**を試してみてください。
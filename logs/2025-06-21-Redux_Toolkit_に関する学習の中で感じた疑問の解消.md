# Redux Toolkit に関する学習の中で感じた疑問の解消

- **学習日**: 2025/06/21
- **学習時間**: 30分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

・このuseAppDispatch, useAppSelectorはuseDispatch, useSelectorのtoolkit versionのようなものでしょうか
・また、payloadActionとは
・rejectWithValueとは
・unwrapについて
・extraReducersには全てのthunk関数に関するpending,fulfilled,rejectedを追加するの？
・加えて、この実装にクリーンアーキテクチャを取り入れるにはどうすればいいか　インターフェースを間に入れる場所とかある？


素晴らしい質問です！Redux Toolkitを使いこなす上で、まさに核心を突く重要なポイントばかりです。一つ一つ、丁寧に深掘りしていきましょう。

---

### 1. `useAppDispatch` と `useAppSelector` はToolkit版？

いいえ、これらは**Redux Toolkit版というわけではなく、TypeScriptで型安全性を確保するために自分で設定する「型付け済みラッパーフック」**です。

*   **元のフック**:
    *   `useDispatch`: 返り値の`dispatch`関数が、どんなAction（特にThunk）を受け付けるか型レベルで知らない。
    *   `useSelector`: 引数の`state`が`any`型になってしまい、ストア全体の型構造を知らない。
*   **型付け済みラッパーフック**:
    *   `useAppDispatch`: ストアの`dispatch`関数の型（`AppDispatch`）を事前に教えておくことで、型付けされたThunkなどを正しく推論してくれる。
    *   `useAppSelector`: ストア全体の`state`の型（`RootState`）を事前に教えておくことで、`state.users.items`のようなプロパティへのアクセス時に、強力なオートコンプリートが効き、型間違いを防げる。

**結論**: これらは`react-redux`の標準フックを、自分のストアの型に合わせてカスタマイズしたものです。Redux Toolkitを使うなら、この設定は**必須**と言えます。

---

### 2. `PayloadAction<T>` とは？

`PayloadAction<T>`は、**Reducerに渡される`action`オブジェクトの型を定義するための、Redux Toolkitのユーティリティ型**です。

`action`オブジェクトは、通常`{ type: string, payload: any }`という形をしています。この`payload`が`any`型だと、TypeScriptの恩恵を受けられません。

`PayloadAction<T>`は、`payload`の型をジェネリクス`T`で指定できるようにしてくれます。

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

// ...

const initialState = { value: 0 };

const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    // payloadが数値であることを型定義
    incrementByAmount(state, action: PayloadAction<number>) {
      // action.payload は number型として扱われる
      state.value += action.payload;
    },
    // payloadがユーザーオブジェクトであることを型定義
    setUser(state, action: PayloadAction<User>) {
      // action.payload は User型として扱われる
      state.user = action.payload;
    }
  }
});
```

**結論**: `PayloadAction<T>`は、`action.payload`に具体的な型を与えるための型ヘルパーです。これにより、Reducer内のロジックが型安全になります。

---

### 3. `rejectWithValue` とは？

`rejectWithValue`は、**`createAsyncThunk`が失敗（rejected）した際に、Actionの`payload`に特定の値を設定して返すためのユーティリティ関数**です。

通常、Thunk内で`throw new Error('何らかのエラー')`をすると、`rejected`状態のActionの`action.payload`は`undefined`になり、エラー情報は`action.error`オブジェクトに格納されます。これでも動作しますが、エラーの扱いが少し煩雑です。

`rejectWithValue`を使うと、このフローをより綺麗にできます。

```typescript
export const fetchUserById = createAsyncThunk(
  'users/fetchById',
  async (userId, { rejectWithValue }) => {
    try {
      const response = await userAPI.fetchById(userId);
      return response.data;
    } catch (error: any) {
      // エラーが発生したら、rejectWithValueでカスタムエラーメッセージを返す
      // この値が、rejectedアクションのpayloadになる
      return rejectWithValue(error.response.data.message || 'Failed to fetch user');
    }
  }
);

// Reducer側
builder.addCase(fetchUserById.rejected, (state, action) => {
  state.status = 'failed';
  // action.payloadに、rejectWithValueで返した値が入ってくる！
  // action.error.message を見に行く必要がない。
  state.error = action.payload as string;
});
```

**結論**: `rejectWithValue`は、Thunkの失敗時に、`rejected`アクションの`payload`として好きな値（エラーメッセージなど）を格納するための公式な方法です。これにより、Reducerでのエラーハンドリングが統一され、シンプルになります。

---

### 4. `.unwrap()` とは？

`.unwrap()`は、コンポーネント側で**Thunkの実行結果（成功または失敗）を簡単に扱うためのヘルパー関数**です。

`dispatch(someThunk())`は、Promiseを返します。このPromiseは常に**成功（resolve）**します。
*   Thunkが成功した場合: `fulfilled`アクションを`payload`に持つオブジェクトで解決されます。
*   Thunkが失敗した場合: `rejected`アクションを`payload`に持つオブジェクトで解決されます。

これだと、コンポーネント側で成功/失敗を判定するのが面倒です。`.unwrap()`は、この結果を「アンラップ（展開）」してくれます。

*   **Thunkが成功した場合**: `.unwrap()`は、`fulfilled`アクションの`payload`（APIからのデータなど）を返します。
*   **Thunkが失敗した場合**: `.unwrap()`は、Promiseを**失敗（reject）**させ、`rejected`アクションの`payload`（`rejectWithValue`で設定した値）を持つエラーを**throwします**。

これにより、コンポーネント側で直感的な`async/await`と`try/catch`が使えるようになります。

```tsx
// コンポーネント内
const onSave = async (data) => {
  try {
    // dispatchした結果をunwrap()で受け取る
    const resultPayload = await dispatch(updatePost(data)).unwrap();
    toast.success('更新に成功しました！');
  } catch (errorPayload) {
    // Thunkが失敗した場合、catchブロックが実行される
    toast.error(`更新に失敗: ${errorPayload}`);
  }
};
```

**結論**: `.unwrap()`は、コンポーネント側でThunkのPromiseを扱いやすくするための便利なユーティリティです。`try/catch`で成功/失敗をハンドリングできます。

---

### 5. `extraReducers` には全ての状態を追加する？

いいえ、**必ずしも`pending`, `fulfilled`, `rejected`の3つすべてを追加する必要はありません**。UIの要件に応じて、必要なものだけを定義すればOKです。

*   **`pending`**: ローディングスピナーを表示するなど、処理中に何かをしたい場合に定義します。
*   **`fulfilled`**: データ取得成功時に、そのデータを`state`に格納するために**ほぼ常に必要**です。
*   **`rejected`**: エラーメッセージを表示するなど、失敗時の処理をしたい場合に定義します。

例えば、「バックグラウンドでデータを更新するだけで、ユーザーにローディングやエラーを見せる必要がない」というシンプルなケースでは、`fulfilled`だけを定義することもあります。

```typescript
builder
  .addCase(updateLog.fulfilled, (state, action) => {
    // 更新成功時のロジックだけ書く
  });
// .pendingや.rejectedは定義しなくてもOK
```

**結論**: 必要な状態遷移だけを定義してください。ただし、データ取得（Read）の場合は、ユーザー体験向上のために3つとも定義するのが一般的です。

---

### 6. クリーンアーキテクチャの導入

素晴らしい視点です！ReduxのThunkにクリーンアーキテクチャを取り入れることは可能です。**インターフェースを挟む場所は、Thunk（アプリケーション層）とAPI通信（インフラストラクチャ層）の間**が最適です。

#### ステップ

1.  **Repositoryインターフェースの定義（ドメイン層/アプリケーション層）**
    「どのように」データを取得するかは知らず、「何をするか」だけを定義します。

    ```typescript
    // src/features/logs/logsRepository.ts
    import { LogDTO } from '@/types/log';

    // このインターフェースが「境界」となる
    export interface ILogsRepository {
      getLogs(unitId: string): Promise<LogDTO[]>;
      createLog(unitId: string, logData: any): Promise<LogDTO>;
      // ... update, delete
    }
    ```

2.  **具象Repositoryの実装（インフラストラクチャ層）**
    インターフェースを実装し、具体的な`fetch`ロジックを記述します。

    ```typescript
    // src/infrastructure/supabase/supabaseLogsRepository.ts
    import { ILogsRepository } from '@/features/logs/logsRepository';

    export class SupabaseLogsRepository implements ILogsRepository {
      async getLogs(unitId: string): Promise<LogDTO[]> {
        const response = await fetch(`/api/units/${unitId}/logs`);
        // ...エラーハンドリングなど
        const data = await response.json();
        return data.data;
      }
      // ... createLogなどの実装
    }
    ```

3.  **Thunkでインターフェースに依存する**
    `createAsyncThunk`の第3引数`thunkAPI`の`extra`プロパティを使って、具象クラスを**DI（Dependency Injection）**します。

    **ストアの設定**
    ```typescript
    // src/app/store.ts
    import { configureStore } from '@reduxjs/toolkit';
    import { SupabaseLogsRepository } from '@/infrastructure/supabase/supabaseLogsRepository';

    const logsRepository = new SupabaseLogsRepository();

    export const store = configureStore({
      reducer: { /* ... */ },
      middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware({
          thunk: {
            extraArgument: {
              logsRepository, // thunkに渡す依存関係
            },
          },
        }),
    });
    ```

    **Thunkの実装**
    ```typescript
    // src/features/logs/logsSlice.ts
    export const fetchLogsByUnitId = createAsyncThunk(
      'logs/fetchByUnitId',
      async (unitId: string, { extra, rejectWithValue }) => {
        // DIされたrepositoryを取得
        const { logsRepository } = extra as { logsRepository: ILogsRepository };
        
        try {
          // fetchを直接叩くのではなく、インターフェースのメソッドを呼ぶ
          const logs = await logsRepository.getLogs(unitId);
          return logs;
        } catch (error: any) {
          return rejectWithValue(error.message);
        }
      }
    );
    ```

**メリット**:
*   **テスト容易性**: Thunkのテスト時に、`SupabaseLogsRepository`の代わりにモックのRepositoryを注入できる。
*   **交換可能性**: データソースをSupabaseからFirebaseに変更したくなっても、`FirebaseLogsRepository`を新しく作って`store.ts`の1行を差し替えるだけで済み、**Thunk（`logsSlice.ts`）は一切変更する必要がありません**。

**結論**: ThunkとAPI通信の間にRepositoryインターフェースを設けることで、クリーンアーキテクチャの**依存性逆転の原則**を実現し、柔軟でテストしやすいコードベースを構築できます。
# ZustandからRedux Toolkit への移行方法例を元に考えるRedux Toolkitの使い方

- **学習日**: 2025/06/21
- **学習時間**: 30分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

現在のプロジェクト（Zustandを使用）にRedux Toolkitを導入する場合、これは単なるリファクタリングというよりは、**状態管理ライブラリの置き換え**という大きな変更になります。

もし導入するとしたら、以下のステップでリファクタリングを進めることになります。

---

### Redux Toolkit導入リファクタリング計画

#### フェーズ1：基盤の構築

1.  **ライブラリのインストール**
    まずは必要なパッケージをインストールします。
    ```bash
    npm install @reduxjs/toolkit react-redux
    ```

2.  **ディレクトリ構造の作成**
    推奨される`features`ベースのディレクトリ構造を作成します。
    *   `src/app/store.ts` （ストア設定ファイル）
    *   `src/features/` （各機能のロジックを格納するディレクトリ）

3.  **Reduxストアの作成 (`src/app/store.ts`)**
    アプリケーションの「心臓部」となるストアを作成します。最初は空の`reducer`で構いません。

    ```typescript
    // src/app/store.ts
    import { configureStore } from '@reduxjs/toolkit';

    export const store = configureStore({
      reducer: {
        // ここに将来的に各featureのreducerが追加される
        // users: usersReducer,
        // logs: logsReducer,
      },
    });

    // アプリケーション全体で使う型を定義しておく
    export type RootState = ReturnType<typeof store.getState>;
    export type AppDispatch = typeof store.dispatch;
    ```

4.  **Redux Providerの設置**
    アプリケーション全体をReduxの`Provider`でラップし、どのコンポーネントからでもストアにアクセスできるようにします。これは既存の`src/components/providers.tsx`を修正するのが良いでしょう。

    ```typescript
    // src/components/providers.tsx
    "use client";

    import { store } from '@/app/store'; // 作成したストアをインポート
    import { Provider as ReduxProvider } from 'react-redux'; // react-reduxからProviderをインポート
    import { SWRProvider } from "@/lib/swr";
    // ... 他のimport

    export function Providers({ children }: { children: React.ReactNode }) {
      // ... 既存のuseEffectなど

      return (
        <ReduxProvider store={store}>
          <SWRProvider>{children}</SWRProvider>
        </ReduxProvider>
      );
    }
    ```

#### フェーズ2：Zustandからの段階的な移行

一度に全てを置き換えるのはリスクが高いため、機能ごとに段階的に移行します。まずは認証（Auth）の部分から始めましょう。

1.  **認証Sliceの作成 (`src/features/auth/authSlice.ts`)**
    現在Zustandの`useAuthStore`が担っている役割を、Redux Toolkitの`createSlice`と`createAsyncThunk`を使って再実装します。

    ```typescript
    // src/features/auth/authSlice.ts
    import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
    import { supabase } from '@/lib/supabase-auth';
    import { Session, User } from '@supabase/supabase-js';

    // Thunkで非同期のセッション取得を定義
    export const initializeAuth = createAsyncThunk(
      'auth/initialize',
      async (_, { dispatch }) => {
        const { data: { session } } = await supabase.auth.getSession();
        supabase.auth.onAuthStateChange((_event, session) => {
          // 状態変化があれば、同期Actionをdispatchしてstoreを更新
          dispatch(authSlice.actions.setAuthState({ session, user: session?.user ?? null }));
        });
        return { session, user: session?.user ?? null };
      }
    );

    interface AuthState {
      user: User | null;
      session: Session | null;
      loading: boolean;
    }

    const initialState: AuthState = {
      user: null,
      session: null,
      loading: true,
    };

    const authSlice = createSlice({
      name: 'auth',
      initialState,
      reducers: {
        // onAuthStateChangeから呼ばれる同期Action
        setAuthState(state, action: PayloadAction<{ session: Session | null; user: User | null }>) {
          state.session = action.payload.session;
          state.user = action.payload.user;
        },
      },
      extraReducers: (builder) => {
        builder
          .addCase(initializeAuth.pending, (state) => {
            state.loading = true;
          })
          .addCase(initializeAuth.fulfilled, (state, action) => {
            state.loading = false;
            state.session = action.payload.session;
            state.user = action.payload.user;
          });
      },
    });

    export default authSlice.reducer;
    ```

2.  **ストアに`authSlice`を登録**

    ```typescript
    // src/app/store.ts
    import { configureStore } from '@reduxjs/toolkit';
    import authReducer from '@/features/auth/authSlice'; // 作成したsliceのreducerをインポート

    export const store = configureStore({
      reducer: {
        auth: authReducer, // "auth"という名前で登録
      },
    });
    // ...
    ```

3.  **認証初期化の呼び出し**
    `Providers`コンポーネントで、Zustandの初期化の代わりに、新しく作った`initializeAuth` Thunkを`dispatch`します。

    ```tsx
    // src/components/providers.tsx
    // ...
    import { initializeAuth } from '@/features/auth/authSlice';
    import { useAppDispatch } from '@/app/hooks'; // 下記で作成するカスタムフック

    export function Providers({ children }: { children: React.ReactNode }) {
      const dispatch = useAppDispatch();
      useEffect(() => {
        dispatch(initializeAuth());
      }, [dispatch]);

      // ...
    }
    ```
    ※型付けされた`useDispatch`と`useSelector`を使うために、カスタムフックを作成します。

    ```typescript
    // src/app/hooks.ts (新規作成)
    import { useDispatch, useSelector, TypedUseSelectorHook } from 'react-redux';
    import type { RootState, AppDispatch } from './store';

    export const useAppDispatch = () => useDispatch<AppDispatch>();
    export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
    ```

#### フェーズ3：コンポーネントの置き換え

1.  **`useAuthStore`を`useAppSelector`に置き換える**
    今まで`useAuthStore`を使っていたコンポーネントを、`useAppSelector`を使ってReduxストアからデータを取得するように書き換えます。

    **変更前 (Zustand)**
    ```tsx
    // src/components/layout/Header.tsx
    import { useAuthStore } from "@/stores/SupabaseAuthStore";

    function Header() {
      const session = useAuthStore((state) => state.session);
      // ...
    }
    ```

    **変更後 (Redux Toolkit)**
    ```tsx
    // src/components/layout/Header.tsx
    import { useAppSelector } from "@/app/hooks";

    function Header() {
      const session = useAppSelector((state) => state.auth.session);
      // ...
    }
    ```

#### フェーズ4：他の機能の移行

認証機能の移行が完了したら、同様の手順で他の状態管理（学習ログ、コメントなど）も`features`ディレクトリにそれぞれのSliceを作成し、段階的に移行していきます。

*   `features/logs/logsSlice.ts`
*   `features/comments/commentsSlice.ts`

最終的に、すべての状態管理がRedux Toolkitに移行できたら、Zustandはプロジェクトから削除できます。

---

### まとめ

このリファクタリングは、**ZustandからRedux Toolkitへの完全な移行**を意味します。
キーポイントは以下の通りです。

1.  **基盤構築**: RTKと`react-redux`をインストールし、`store`と`Provider`を設定する。
2.  **段階的移行**: 機能（feature）ごとにSliceを作成し、一つずつ着実にZustandから置き換える。まずは`auth`から。
3.  **コンポーネント修正**: `useAuthStore`を`useAppDispatch`と`useAppSelector`に置き換える。

これは大きな変更ですが、Redux Toolkitが提供する堅牢な型サポート、拡張性、強力な開発者ツール（Redux DevTools）の恩恵は、特にアプリケーションが成長するにつれて大きな価値を発揮するでしょう。
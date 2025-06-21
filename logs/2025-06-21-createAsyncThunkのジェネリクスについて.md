# createAsyncThunkのジェネリクスについて

- **学習日**: 2025/06/21
- **学習時間**: 20分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

`createAsyncThunk` のジェネリクスについて詳しく解説しますね！これはRedux Toolkitの中でも特に重要で、理解すると非同期処理の型安全性が格段に向上します。

## 🎯 createAsyncThunk のジェネリクス構造

### 基本的な構文
```typescript
createAsyncThunk<Returned, ThunkArg, ThunkApiConfig>(
  typePrefix: string,
  payloadCreator: (arg: ThunkArg, thunkAPI: GetThunkAPI<ThunkApiConfig>) => Promise<Returned>
)
```

## 📝 各ジェネリクスの詳細解説

### 1. **Returned** - 戻り値の型

```typescript
// ✅ 例: ログ一覧を取得する場合
export const fetchLogsByUnitId = createAsyncThunk<
  Log[], // 👈 Returned: 非同期処理が成功した時の戻り値の型
  number,
  { extra: { logRepository: ILogRepository } }
>(
  "logs/fetchByUnitId",
  async (unitId: number, { extra: { logRepository } }) => {
    // この関数の戻り値が Log[] 型であることを保証
    return await logRepository.findByUnitId(unitId);
  }
);
```

**ポイント:**
- `fulfilled` アクションの `payload` の型になる
- Redux Slice の `extraReducers` で型安全にアクセス可能

### 2. **ThunkArg** - 引数の型

```typescript
// ✅ 引数が単一の場合
export const fetchLogsByUnitId = createAsyncThunk<
  Log[],
  number, // 👈 ThunkArg: unitId (number型)
  { extra: { logRepository: ILogRepository } }
>(
  "logs/fetchByUnitId",
  async (unitId: number, { extra: { logRepository } }) => {
    // unitId は number 型として扱われる
    return await logRepository.findByUnitId(unitId);
  }
);

// ✅ 引数が複数の場合はオブジェクト型で定義
export const updateLog = createAsyncThunk<
  Log,
  { id: number; updates: Partial<CreateLogRequest> }, // 👈 複数の引数をオブジェクトで
  { extra: { logRepository: ILogRepository } }
>(
  "logs/update",
  async ({ id, updates }, { extra: { logRepository } }) => {
    return await logRepository.update(id, updates);
  }
);

// ✅ 引数が不要な場合
export const fetchCurrentUser = createAsyncThunk<
  User,
  void, // 👈 引数が不要な場合は void
  { extra: { userRepository: IUserRepository } }
>(
  "user/fetchCurrent",
  async (_, { extra: { userRepository } }) => {
    return await userRepository.getCurrentUser();
  }
);
```

### 3. **ThunkApiConfig** - 設定オブジェクト

```typescript
export const fetchLogsByUnitId = createAsyncThunk<
  Log[],
  number,
  {
    // 👇 ThunkApiConfig: 様々な設定を型安全に定義
    state: RootState;           // Redux state の型
    dispatch: AppDispatch;      // dispatch の型
    extra: {                   // 依存注入される追加の値
      logRepository: ILogRepository;
      userRepository: IUserRepository;
    };
    rejectValue: string;        // エラー時の値の型
  }
>(
  "logs/fetchByUnitId",
  async (unitId: number, thunkAPI) => {
    try {
      // thunkAPI.getState() で型安全にstateアクセス
      const currentUser = thunkAPI.getState().auth.user;
      
      // thunkAPI.extra で依存注入された値を使用
      const logs = await thunkAPI.extra.logRepository.findByUnitId(unitId);
      
      return logs;
    } catch (error) {
      // thunkAPI.rejectWithValue で型安全にエラー処理
      return thunkAPI.rejectWithValue("ログの取得に失敗しました");
    }
  }
);
```

## 🔍 実践的な使用例

### パターン1: シンプルなCRUD操作

```typescript
// 作成
export const createLog = createAsyncThunk<
  Log,                          // 作成されたログ
  CreateLogRequest,             // 作成用のデータ
  { extra: { logRepository: ILogRepository } }
>(
  "logs/create",
  async (request, { extra: { logRepository } }) => {
    return await logRepository.create(request);
  }
);

// 更新
export const updateLog = createAsyncThunk<
  Log,                          // 更新されたログ
  { id: number; updates: Partial<CreateLogRequest> },
  { extra: { logRepository: ILogRepository } }
>(
  "logs/update",
  async ({ id, updates }, { extra: { logRepository } }) => {
    return await logRepository.update(id, updates);
  }
);

// 削除
export const deleteLog = createAsyncThunk<
  number,                       // 削除されたログのID
  number,                       // 削除するログのID
  { extra: { logRepository: ILogRepository } }
>(
  "logs/delete",
  async (id, { extra: { logRepository } }) => {
    await logRepository.delete(id);
    return id; // 削除されたIDを返す
  }
);
```

### パターン2: エラーハンドリングを含む複雑なケース

```typescript
export const fetchLogsByUnitId = createAsyncThunk<
  Log[],
  number,
  {
    state: RootState;
    extra: { logRepository: ILogRepository };
    rejectValue: {
      message: string;
      code: string;
    };
  }
>(
  "logs/fetchByUnitId",
  async (unitId, { getState, extra: { logRepository }, rejectWithValue }) => {
    try {
      // 状態チェック
      const currentUser = getState().auth.user;
      if (!currentUser) {
        return rejectWithValue({
          message: "認証が必要です",
          code: "UNAUTHORIZED"
        });
      }

      // 権限チェック
      const hasPermission = getState().auth.permissions.includes('READ_LOGS');
      if (!hasPermission) {
        return rejectWithValue({
          message: "権限がありません",
          code: "FORBIDDEN"
        });
      }

      return await logRepository.findByUnitId(unitId);
    } catch (error) {
      return rejectWithValue({
        message: error instanceof Error ? error.message : "不明なエラー",
        code: "FETCH_ERROR"
      });
    }
  }
);
```

## 🎨 Redux Slice での活用

```typescript
const logsSlice = createSlice({
  name: "logs",
  initialState,
  reducers: {
    clearError: (state) => {
      state.error = null;
    },
  },
  extraReducers: (builder) => {
    builder
      // fetchLogsByUnitId
      .addCase(fetchLogsByUnitId.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchLogsByUnitId.fulfilled, (state, action) => {
        state.loading = false;
        // action.payload は Log[] 型として型安全
        state.logs = action.payload;
      })
      .addCase(fetchLogsByUnitId.rejected, (state, action) => {
        state.loading = false;
        // rejectValue を使っている場合、action.payload は型安全
        if (action.payload) {
          state.error = action.payload.message;
        } else {
          state.error = "不明なエラーが発生しました";
        }
      })
      
      // createLog
      .addCase(createLog.fulfilled, (state, action) => {
        // action.payload は Log 型として型安全
        state.logs.push(action.payload);
      })
      
      // updateLog
      .addCase(updateLog.fulfilled, (state, action) => {
        const index = state.logs.findIndex(log => log.id === action.payload.id);
        if (index !== -1) {
          // action.payload は Log 型として型安全
          state.logs[index] = action.payload;
        }
      })
      
      // deleteLog
      .addCase(deleteLog.fulfilled, (state, action) => {
        // action.payload は number 型として型安全
        state.logs = state.logs.filter(log => log.id !== action.payload);
      });
  },
});
```

## 💡 高度なテクニック

### 1. 条件付き実行

```typescript
export const fetchLogsByUnitId = createAsyncThunk<
  Log[],
  number,
  { state: RootState; extra: { logRepository: ILogRepository } }
>(
  "logs/fetchByUnitId",
  async (unitId, { extra: { logRepository } }) => {
    return await logRepository.findByUnitId(unitId);
  },
  {
    // 条件付き実行: 既にデータがある場合はスキップ
    condition: (unitId, { getState }) => {
      const { logs } = getState().logs;
      return logs.length === 0; // ログが空の場合のみ実行
    },
  }
);
```

### 2. 複数の依存関係を持つ場合

```typescript
export const createLogWithValidation = createAsyncThunk<
  Log,
  CreateLogRequest,
  {
    extra: {
      logRepository: ILogRepository;
      validationService: IValidationService;
      notificationService: INotificationService;
    };
  }
>(
  "logs/createWithValidation",
  async (request, { extra }) => {
    // バリデーション
    const isValid = await extra.validationService.validateLog(request);
    if (!isValid) {
      throw new Error("バリデーションエラー");
    }

    // 作成
    const log = await extra.logRepository.create(request);

    // 通知
    await extra.notificationService.sendCreatedNotification(log);

    return log;
  }
);
```

## 🎯 まとめ

`createAsyncThunk` のジェネリクスは以下の順序で定義されます：

1. **Returned** - 成功時の戻り値の型
2. **ThunkArg** - 関数の引数の型
3. **ThunkApiConfig** - 設定オブジェクト（state, dispatch, extra, rejectValue など）

この型システムにより：
- **コンパイル時のエラー検出**
- **自動補完の効果**
- **リファクタリングの安全性**
- **ドキュメント的な役割**

が実現され、Redux Toolkit を使った非同期処理が格段に安全で保守しやすくなります！
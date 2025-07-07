# Jest Mock: `mockReturnValue` vs `mockResolvedValue`

- **学習日**: 2025/07/03
- **学習時間**: 30分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

# 🎭 Jest Mock: `mockReturnValue` vs `mockResolvedValue` 完全解説

この2つのメソッドの違いを、技術的な背景から実践的な使い分けまで詳しく解説します。

## 📋 基本的な違い

### **概要**
```typescript
// 同期処理用
mockFunction.mockReturnValue(value)

// 非同期処理用（Promise）
mockFunction.mockResolvedValue(value)
```

---

## 🔍 詳細解説

### **1. mockReturnValue - 同期処理**

#### **基本的な使用法**
```typescript
// 同期関数のモック
const mockCalculate = jest.fn();

// 固定値を返すように設定
mockCalculate.mockReturnValue(42);

// テスト実行
const result = mockCalculate(1, 2, 3);
console.log(result); // 42

// 型定義
mockFunction.mockReturnValue(value: any): Mock
```

#### **実際の使用例**
```typescript
// ユーティリティ関数のモック
const mockFormatCurrency = jest.fn();
mockFormatCurrency.mockReturnValue("¥1,000");

// テスト
expect(mockFormatCurrency(1000)).toBe("¥1,000");
```

### **2. mockResolvedValue - 非同期処理（成功）**

#### **基本的な使用法**
```typescript
// 非同期関数のモック
const mockFetchUser = jest.fn();

// Promiseの成功値を設定
mockFetchUser.mockResolvedValue({ id: 1, name: "太郎" });

// テスト実行（awaitが必要）
const result = await mockFetchUser("user-1");
console.log(result); // { id: 1, name: "太郎" }

// 型定義
mockFunction.mockResolvedValue(value: any): Mock
```

#### **実際の使用例**
```typescript
// API呼び出しのモック
const mockApiClient = {
  get: jest.fn(),
};

mockApiClient.get.mockResolvedValue({
  data: { users: [{ id: 1, name: "太郎" }] }
});

// テスト
const response = await mockApiClient.get("/users");
expect(response.data.users).toHaveLength(1);
```

---

## ⚡ 動作メカニズムの違い

### **内部的な実装イメージ**

#### **mockReturnValue の動作**
```typescript
// 内部的にはこのような動作
function mockReturnValue(value) {
  this._mockImplementation = () => value;
}

// 実行時
const result = mockFunction(); // 即座にvalueを返す
```

#### **mockResolvedValue の動作**
```typescript
// 内部的にはこのような動作
function mockResolvedValue(value) {
  this._mockImplementation = () => Promise.resolve(value);
}

// 実行時
const result = mockFunction(); // Promise<value>を返す
const actualValue = await result; // awaitが必要
```

---

## 📊 実践的な比較例

### **学習ジャーナルプロジェクトでの実例**
Ran tool

### **実際の使用例での比較**

```typescript
// ============================================
// 🔄 同期処理での使用例
// ============================================

// 1. バリデーション関数のモック
const mockValidate = jest.fn().mockReturnValue(true);
const result = mockValidate({ title: "テスト" });
console.log(result); // true （同期的に即座に返される）

// 2. ユーティリティ関数のモック
const mockFormatDate = jest.fn().mockReturnValue("2023-12-25");
const formatted = mockFormatDate(new Date());
console.log(formatted); // "2023-12-25"

// ============================================
// ⚡ 非同期処理での使用例
// ============================================

// 1. API クライアントのモック
const mockApiClient = {
  post: jest.fn().mockResolvedValue({
    data: { id: 123, title: "学習ログ" }
  })
};

// 非同期処理 - awaitが必要
const response = await mockApiClient.post("/api/logs", data);
console.log(response.data); // { id: 123, title: "学習ログ" }

// 2. データベース操作のモック
const mockUserSave = jest.fn().mockResolvedValue({
  success: true,
  messageId: "12345"
});

const result = await mockUserSave(userData);
console.log(result); // { success: true, messageId: "12345" }
```

---

## 🚨 よくある間違いと注意点

### **❌ 間違った使用例**

```typescript
// 間違い：非同期関数にmockReturnValueを使用
const mockApiCall = jest.fn().mockReturnValue({
  data: { success: true }
});

// これは Promise<{data: {success: true}}> を返さない
const result = await mockApiCall(); // エラー！awaitできない
```

### **✅ 正しい使用例**

```typescript
// 正しい：非同期関数にmockResolvedValueを使用
const mockApiCall = jest.fn().mockResolvedValue({
  data: { success: true }
});

// これは Promise<{data: {success: true}}> を返す
const result = await mockApiCall(); // 正常に動作
```

### **⚠️ 併用時の注意**

```typescript
// 同期と非同期の混在パターン
const mockFunction = jest.fn()
  .mockReturnValueOnce("同期の値")        // 1回目：同期
  .mockResolvedValueOnce("非同期の値");   // 2回目：非同期

// 使用時
const result1 = mockFunction();         // "同期の値"
const result2 = await mockFunction();   // "非同期の値"
```

---

## 📋 実践的な選択指針

### **🎯 mockReturnValue を選ぶべき場合**

1. **バリデーション関数**
   ```typescript
   const mockValidate = jest.fn().mockReturnValue(true);
   ```

2. **ユーティリティ関数**
   ```typescript
   const mockFormatDate = jest.fn().mockReturnValue("2023-12-25");
   ```

3. **計算関数**
   ```typescript
   const mockCalculate = jest.fn().mockReturnValue(100);
   ```

4. **プロパティアクセス**
   ```typescript
   const mockGetUserName = jest.fn().mockReturnValue("太郎");
   ```

### **🎯 mockResolvedValue を選ぶべき場合**

1. **API呼び出し**
   ```typescript
   const mockFetch = jest.fn().mockResolvedValue({ data: {} });
   ```

2. **データベース操作**
   ```typescript
   const mockSave = jest.fn().mockResolvedValue({ success: true });
   ```

3. **ファイル操作**
   ```typescript
   const mockReadFile = jest.fn().mockResolvedValue("file content");
   ```

4. **外部サービス呼び出し**
   ```typescript
   const mockSendEmail = jest.fn().mockResolvedValue({ messageId: "123" });
   ```

---

## 🔧 高度な使用パターン

### **複数の戻り値パターン**

```typescript
// 複数回呼び出しでの異なる戻り値
const mockFn = jest.fn()
  .mockReturnValueOnce("1回目")      // 同期
  .mockResolvedValueOnce("2回目")    // 非同期
  .mockReturnValue("デフォルト");    // 3回目以降

// 使用例
const result1 = mockFn();           // "1回目"
const result2 = await mockFn();     // "2回目"
const result3 = mockFn();           // "デフォルト"
```

### **エラーハンドリングパターン**

```typescript
// 同期エラー
const mockSyncFn = jest.fn()
  .mockImplementation(() => {
    throw new Error("同期エラー");
  });

// 非同期エラー
const mockAsyncFn = jest.fn()
  .mockRejectedValue(new Error("非同期エラー"));

// テスト
expect(() => mockSyncFn()).toThrow("同期エラー");
await expect(mockAsyncFn()).rejects.toThrow("非同期エラー");
```

---

## 🎯 まとめ

| 特徴 | `mockReturnValue` | `mockResolvedValue` |
|------|------------------|-------------------|
| **用途** | 同期処理 | 非同期処理（Promise成功） |
| **戻り値** | 直接値 | `Promise<値>` |
| **使用場面** | バリデーション、計算、ユーティリティ | API、DB操作、ファイル処理 |
| **await** | 不要 | 必要 |
| **エラー版** | `mockImplementation` | `mockRejectedValue` |

**🔑 選択の基準**
- テスト対象の関数が **同期** → `mockReturnValue`
- テスト対象の関数が **非同期** → `mockResolvedValue`
- エラーテストが必要 → `mockImplementation` または `mockRejectedValue`

この違いを理解することで、適切なモック戦略を選択し、効果的なテストを書くことができます！
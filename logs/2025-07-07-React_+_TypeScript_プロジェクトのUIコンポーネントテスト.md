# React + TypeScript プロジェクトのUIコンポーネントテスト

- **学習日**: 2025/07/03
- **学習時間**: 90分
- **効果実感**: 4/5
- **効果の種類**: 理解が深まった

## 内容

# 🧪 React + TypeScript プロジェクトのUIコンポーネントテスト完全ガイド

> Next.js 15、React 19、TypeScript環境における実践的なテスト戦略と実装例

## 📋 目次

1. [プロジェクト概要](#プロジェクト概要)
2. [テスト技術スタック](#テスト技術スタック)
3. [段階的テスト戦略](#段階的テスト戦略)
4. [実装したテストケース](#実装したテストケース)
5. [技術的詳細解説](#技術的詳細解説)
6. [ベストプラクティス](#ベストプラクティス)
7. [トラブルシューティング](#トラブルシューティング)
8. [まとめと今後の展望](#まとめと今後の展望)

---

## 🎯 プロジェクト概要

### **学習ジャーナルアプリケーション**
- **フレームワーク**: Next.js 15.2.2
- **UIライブラリ**: React 19
- **言語**: TypeScript
- **認証**: Supabase Auth
- **データベース**: Prisma + PostgreSQL
- **決済**: Stripe統合
- **UIコンポーネント**: Radix UI + Tailwind CSS

### **テスト対象コンポーネント**
- **基本コンポーネント**: Badge, Loading, Card
- **インタラクティブコンポーネント**: TagInput
- **ビジネスロジック**: DashboardStats
- **複雑な状態管理**: DashboardHeader

---

## 🛠️ テスト技術スタック

### **主要ライブラリ**
```json
{
  "jest": "^29.7.0",
  "@testing-library/react": "^14.0.0",
  "@testing-library/jest-dom": "^6.1.4",
  "@testing-library/user-event": "^14.5.1",
  "@types/jest": "^29.5.8"
}
```

### **設定ファイル**
```javascript
// jest.config.js
const nextJest = require('next/jest')

const createJestConfig = nextJest({
  dir: './',
})

const customJestConfig = {
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  testEnvironment: 'jsdom',
  moduleNameMapper: {
    '^@/components/(.*)$': '<rootDir>/src/components/$1',
    '^@/lib/(.*)$': '<rootDir>/src/lib/$1',
    '^@/stores/(.*)$': '<rootDir>/src/stores/$1',
    '^@/hooks/(.*)$': '<rootDir>/src/hooks/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/app/**/*.{js,jsx,ts,tsx}',
  ],
  testPathIgnorePatterns: ['<rootDir>/.next/', '<rootDir>/node_modules/'],
  transform: {
    '^.+\\.(js|jsx|ts|tsx)$': ['babel-jest', { presets: ['next/babel'] }],
  },
  transformIgnorePatterns: [
    '/node_modules/',
    '^.+\\.module\\.(css|sass|scss)$',
  ],
}

module.exports = createJestConfig(customJestConfig)
```

---

## 📊 段階的テスト戦略

### **レベル1: 基本コンポーネント**
**特徴**: シンプルなProps受け取り、基本的な表示機能

#### **Badge コンポーネント (4テスト)**
```typescript
describe("Badge", () => {
  test("基本的なレンダリング", () => {
    render(<Badge>テストバッジ</Badge>);
    
    const badge = screen.getByText("テストバッジ");
    expect(badge).toBeInTheDocument();
  });

  test("異なるvariantが適用される", () => {
    const { rerender } = render(<Badge variant="secondary">Secondary</Badge>);
    
    let badge = screen.getByText("Secondary");
    expect(badge.className).toContain("bg-secondary");
    
    rerender(<Badge variant="destructive">Destructive</Badge>);
    badge = screen.getByText("Destructive");
    expect(badge.className).toContain("bg-destructive");
  });
});
```

#### **Loading コンポーネント (7テスト)**
```typescript
describe("Loading", () => {
  test("基本的なローディングスピナーが表示される", () => {
    render(<Loading />);

    const spinner = document.querySelector("svg");
    expect(spinner).toBeTruthy();
    expect(spinner?.getAttribute("class")).toContain("animate-spin");
  });

  test("異なるサイズが適用される", () => {
    const { rerender } = render(<Loading size="sm" />);

    let spinner = document.querySelector("svg");
    expect(spinner?.getAttribute("class")).toContain("w-4 h-4");

    rerender(<Loading size="lg" />);
    spinner = document.querySelector("svg");
    expect(spinner?.getAttribute("class")).toContain("w-8 h-8");
  });
});
```

#### **Card コンポーネント (3テスト)**
```typescript
describe("Card", () => {
  test("カスタムクラスが適用される", () => {
    const { container } = render(
      <Card className="custom-card">
        <CardContent>テスト</CardContent>
      </Card>
    );

    // Card要素を直接取得
    const card = container.firstChild as HTMLElement;
    expect(card?.className).toContain("custom-card");
  });
});
```

### **レベル2: インタラクティブコンポーネント**
**特徴**: ユーザーイベント処理、状態変更、複雑な条件分岐

#### **TagInput コンポーネント (8テスト)**
```typescript
describe("TagInput", () => {
  const mockSetTags = jest.fn();

  beforeEach(() => {
    mockSetTags.mockClear();
  });

  test("Enterキーでタグが追加される（小文字に変換）", () => {
    render(<TagInput tags={["react"]} setTags={mockSetTags} />);
    
    const input = screen.getByRole("textbox");
    fireEvent.change(input, { target: { value: "TypeScript" } });
    fireEvent.keyDown(input, { key: "Enter" });
    
    expect(mockSetTags).toHaveBeenCalledWith(["react", "typescript"]);
  });

  test("重複するタグは追加されない", () => {
    render(<TagInput tags={["react"]} setTags={mockSetTags} />);
    
    const input = screen.getByRole("textbox");
    fireEvent.change(input, { target: { value: "React" } });
    fireEvent.keyDown(input, { key: "Enter" });
    
    expect(mockSetTags).not.toHaveBeenCalled();
  });
});
```

### **レベル3: ビジネスロジック**
**特徴**: 数値処理、データ変換、計算結果の表示

#### **DashboardStats コンポーネント (7テスト)**
```typescript
describe("DashboardStats", () => {
  test("学習時間の小数点が正しく処理される", () => {
    const dataWithDecimals = {
      totalLearningTime: 123.456,
      completedUnitsCount: 8,
      activeUnitsCount: 3,
      streakDays: 12,
    };
    
    render(<DashboardStats data={dataWithDecimals} />);
    
    expect(screen.getByText("123.5時間")).toBeInTheDocument();
  });

  test("ゼロ値が正しく表示される", () => {
    const zeroData = {
      totalLearningTime: 0,
      completedUnitsCount: 0,
      activeUnitsCount: 0,
      streakDays: 0,
    };
    
    render(<DashboardStats data={zeroData} />);
    
    expect(screen.getByText("0.0時間")).toBeInTheDocument();
    expect(screen.getByText("0日")).toBeInTheDocument();
    
    // 複数の「0個」を確認
    const zeroCountElements = screen.getAllByText("0個");
    expect(zeroCountElements).toHaveLength(2);
  });
});
```

### **レベル4: 高度な状態管理**
**特徴**: 外部依存関係、モック、非同期処理、エラーハンドリング

#### **DashboardHeader コンポーネント (9テスト)**
```typescript
// モック定義
jest.mock("@/stores/SupabaseAuthStore", () => ({
  useAuthStore: jest.fn(),
}));

jest.mock("@/stores/ModalStore", () => ({
  useModalStore: jest.fn(),
}));

jest.mock("next/navigation", () => ({
  useRouter: jest.fn(),
}));

describe("DashboardHeader", () => {
  const mockPush = jest.fn();
  const mockOpenCreateUnitModal = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
    
    mockUseRouter.mockReturnValue({
      push: mockPush,
    });
    
    mockUseModalStore.mockReturnValue({
      openCreateUnitModal: mockOpenCreateUnitModal,
    });
  });

  test("ナビゲーション中のローディング状態", async () => {
    // プロミスを制御するためのモック
    let resolvePromise: () => void;
    const promise = new Promise<void>((resolve) => {
      resolvePromise = resolve;
    });
    
    mockPush.mockImplementation(() => promise);

    render(<DashboardHeader />);

    const profileButton = screen.getByText("プロフィール");
    fireEvent.click(profileButton);

    // ローディング状態を確認
    await waitFor(() => {
      expect(profileButton).toBeDisabled();
    });

    // プロミスを解決
    resolvePromise!();
    
    await waitFor(() => {
      expect(profileButton).not.toBeDisabled();
    });
  });
});
```

---

## 🔧 技術的詳細解説

### **1. セレクタ戦略**

#### **基本セレクタ**
```typescript
// テキストベース
screen.getByText("テキスト内容")
screen.queryByText("存在しない可能性のあるテキスト")

// ロールベース
screen.getByRole("button", { name: "送信" })
screen.getByRole("textbox")

// 属性ベース
screen.getByPlaceholderText("プレースホルダー")
screen.getByTestId("test-id")
```

#### **DOM クエリ**
```typescript
// 直接DOM操作
document.querySelector("svg")
document.querySelectorAll("div")

// 相対的な要素取得
element.closest("div")
element.parentElement
container.firstChild
```

#### **属性アクセス**
```typescript
// className の取得（SVGでも動作）
element.getAttribute("class")

// 従来の方法（SVGでは動作しない場合がある）
element.className
```

### **2. モック戦略**

#### **外部ストアのモック**
```typescript
jest.mock("@/stores/SupabaseAuthStore", () => ({
  useAuthStore: jest.fn(),
}));

const mockUseAuthStore = useAuthStore as jest.MockedFunction<typeof useAuthStore>;

// テスト内で状態を設定
mockUseAuthStore.mockReturnValue({
  session: { user: { id: "test-user" } },
  loading: false,
});
```

#### **Next.js ルーターのモック**
```typescript
jest.mock("next/navigation", () => ({
  useRouter: jest.fn(),
}));

const mockUseRouter = useRouter as jest.MockedFunction<typeof useRouter>;
const mockPush = jest.fn();

mockUseRouter.mockReturnValue({
  push: mockPush,
});
```

#### **関数のモック**
```typescript
// 基本的なモック
const mockCallback = jest.fn();

// 戻り値を設定
mockCallback.mockReturnValue("戻り値");

// 非同期の戻り値
mockCallback.mockResolvedValue("非同期戻り値");
mockCallback.mockRejectedValue(new Error("エラー"));

// 実装を設定
mockCallback.mockImplementation((arg) => {
  return `処理結果: ${arg}`;
});
```

### **3. 非同期処理テスト**

#### **waitFor を使った非同期テスト**
```typescript
test("非同期処理のテスト", async () => {
  render(<AsyncComponent />);
  
  fireEvent.click(screen.getByText("非同期処理開始"));
  
  await waitFor(() => {
    expect(screen.getByText("処理完了")).toBeInTheDocument();
  });
});
```

#### **Promise の制御**
```typescript
test("Promise制御のテスト", async () => {
  let resolvePromise: () => void;
  const promise = new Promise<void>((resolve) => {
    resolvePromise = resolve;
  });
  
  mockAsyncFunction.mockImplementation(() => promise);
  
  // 処理開始
  fireEvent.click(button);
  
  // ローディング状態確認
  await waitFor(() => {
    expect(button).toBeDisabled();
  });
  
  // Promise解決
  resolvePromise!();
  
  // 完了状態確認
  await waitFor(() => {
    expect(button).not.toBeDisabled();
  });
});
```

### **4. エラーハンドリングテスト**

#### **Console スパイ**
```typescript
test("エラーログのテスト", async () => {
  const consoleErrorSpy = jest.spyOn(console, 'error').mockImplementation(() => {});
  
  // エラーを発生させる処理
  mockFunction.mockRejectedValue(new Error("テストエラー"));
  
  fireEvent.click(button);
  
  await waitFor(() => {
    expect(consoleErrorSpy).toHaveBeenCalledWith("Error message:", expect.any(Error));
  });
  
  consoleErrorSpy.mockRestore();
});
```

### **5. 条件付きレンダリングテスト**

#### **要素の存在確認**
```typescript
// 存在確認
expect(screen.getByText("存在する要素")).toBeInTheDocument();

// 非存在確認
expect(screen.queryByText("存在しない要素")).not.toBeInTheDocument();

// 複数要素の確認
expect(screen.getAllByText("複数の要素")).toHaveLength(2);

// 条件付き表示
expect(screen.queryAllByRole("textbox")).toHaveLength(0);
```

---

## 🎯 ベストプラクティス

### **1. テスト構造**

#### **Arrange-Act-Assert パターン**
```typescript
test("機能説明", () => {
  // Arrange: テスト準備
  const mockProps = { /* ... */ };
  
  // Act: 実行
  render(<Component {...mockProps} />);
  fireEvent.click(screen.getByText("ボタン"));
  
  // Assert: 検証
  expect(mockCallback).toHaveBeenCalledWith(expectedValue);
});
```

#### **beforeEach でのクリーンアップ**
```typescript
describe("ComponentName", () => {
  const mockFunction = jest.fn();
  
  beforeEach(() => {
    jest.clearAllMocks();
    // 必要な初期設定
  });
  
  // テストケース...
});
```

### **2. 実装に依存しないテスト**

#### **ユーザー視点のテスト**
```typescript
// ❌ 実装詳細に依存
expect(component.state.isLoading).toBe(true);

// ✅ ユーザー視点
expect(screen.getByText("読み込み中...")).toBeInTheDocument();
```

#### **適切なセレクタ優先度**
```typescript
// 1. アクセシビリティ (推奨)
screen.getByRole("button")
screen.getByLabelText("ラベル")

// 2. テキスト
screen.getByText("テキスト")

// 3. test-id (最後の手段)
screen.getByTestId("test-id")
```

### **3. モックの適切な使用**

#### **必要最小限のモック**
```typescript
// ❌ 過度なモック
jest.mock("react", () => ({
  useState: jest.fn(),
  useEffect: jest.fn(),
}));

// ✅ 必要な部分のみ
jest.mock("@/stores/AuthStore", () => ({
  useAuthStore: jest.fn(),
}));
```

---

## 🔧 トラブルシューティング

### **1. よくある問題と解決策**

#### **Jest 型定義エラー**
```typescript
// 問題: expect().toBeTruthy() が認識されない
// 解決: jest.setup.js で適切にインポート

// jest.setup.js
import "@testing-library/jest-dom";
```

#### **SVG className 取得エラー**
```typescript
// 問題: SVG要素のclassNameが取得できない
// 解決: getAttribute を使用

// ❌ 動作しない場合がある
expect(svgElement.className).toContain("class-name");

// ✅ 確実に動作
expect(svgElement.getAttribute("class")).toContain("class-name");
```

#### **複数要素の選択エラー**
```typescript
// 問題: "Found multiple elements" エラー
// 解決: より具体的なセレクタまたは getAllBy を使用

// ❌ エラーが発生
expect(screen.getByText("0個")).toBeInTheDocument();

// ✅ 複数要素を適切に処理
const elements = screen.getAllByText("0個");
expect(elements).toHaveLength(2);
```

### **2. パフォーマンス最適化**

#### **重いコンポーネントのモック**
```typescript
// 重いチャートライブラリをモック
jest.mock("recharts", () => ({
  ResponsiveContainer: ({ children }: any) => children,
  BarChart: ({ children }: any) => <div data-testid="bar-chart">{children}</div>,
  Bar: () => <div data-testid="bar" />,
}));
```

---

## 📊 実装結果サマリー

### **テスト統計**
```
総テスト数: 38テスト
成功率: 100%
カバレッジ: 基本機能の完全カバー

コンポーネント別内訳:
├── Badge: 4テスト (基本レベル)
├── Loading: 7テスト (基本レベル)
├── Card: 3テスト (基本レベル)
├── TagInput: 8テスト (中級レベル)
├── DashboardStats: 7テスト (中級レベル)
└── DashboardHeader: 9テスト (高級レベル)
```

### **習得したテスト技術**
- ✅ **基本レンダリング**テスト
- ✅ **Props 検証**テスト
- ✅ **ユーザーインタラクション**テスト
- ✅ **状態管理**テスト
- ✅ **モック・依存関係**テスト
- ✅ **エラーハンドリング**テスト
- ✅ **非同期処理**テスト
- ✅ **DOM操作・セレクタ**テスト

---

## 🚀 まとめと今後の展望

### **達成したこと**
1. **段階的なテスト戦略**の確立
2. **実践的なテストパターン**の習得
3. **堅牢なテスト基盤**の構築
4. **保守性の高いテストコード**の作成

### **今後の発展方向**

#### **1. テストカバレッジの拡張**
- **API レイヤー**のテスト
- **カスタムフック**のテスト
- **ユーティリティ関数**のテスト

#### **2. E2E テストの充実**
- **Cypress** による統合テスト
- **ユーザーフロー**の完全テスト
- **クロスブラウザ**テスト

#### **3. CI/CD パイプライン統合**
- **自動テスト実行**
- **テストレポート**生成
- **品質ゲート**の設定

#### **4. 高度なテスト技術**
- **Visual Regression Testing**
- **Performance Testing**
- **Accessibility Testing**

---

## 🏆 最終的な価値

このテスト実装により、以下の価値を実現しました：

### **開発効率の向上**
- 🔄 **迅速なリファクタリング**
- 🐛 **早期バグ発見**
- 📈 **継続的な品質向上**

### **コードの信頼性**
- ✅ **回帰テスト**の自動化
- 🛡️ **破綻変更**の防止
- 📊 **品質メトリクス**の可視化

### **チーム開発の安全性**
- 🤝 **コラボレーション**の促進
- 📚 **仕様の明文化**
- 🎯 **一貫した品質基準**

---

**この記事が、React + TypeScript プロジェクトでのテスト実装の参考になれば幸いです。テストは単なる品質保証ツールではなく、より良いコードを書くための強力なパートナーです。** 🚀
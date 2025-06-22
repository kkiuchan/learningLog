# useEffectのクリーンアップ処理のすすめ

- **学習日**: 2025/06/22
- **学習時間**: 30分
- **効果実感**: 4/5
- **効果の種類**: 理解が深まった

## 内容

# React useEffectクリーンアップ実践ガイド：非同期処理・リアルタイム通信・DOM監視の完全攻略

ReactのuseEffectにおけるクリーンアップ処理は、メモリリークを防ぎ、安全で効率的なアプリケーションを構築するための重要な技術です。

この記事では、**非同期API呼び出し**、**WebSocket・SSE通信**、**DOM Observer系API**の3つの重要なケースについて、実践的なクリーンアップ手法を詳しく解説します。

---

## 🚀 非同期API呼び出しのクリーンアップ

### ❌ 危険なパターン：クリーンアップなしの非同期処理

```tsx
// 問題のあるコード例
useEffect(() => {
  fetch("/api/user-data")
    .then(response => response.json())
    .then(data => {
      setUserData(data); // ⚠️ アンマウント後に実行される可能性
    })
    .catch(error => {
      setError(error.message); // ⚠️ 同様に危険
    });
}, []);
```

**発生する問題：**
- React警告: `Can't perform a React state update on an unmounted component`
- メモリリーク：不要なAPI呼び出しが継続
- 競合状態（Race Condition）の発生

### ✅ 解決法①：AbortControllerを使ったAPIキャンセル（推奨）

```tsx
useEffect(() => {
  const abortController = new AbortController();

  const fetchData = async () => {
    try {
      setLoading(true);
      const response = await fetch('/api/data', {
        signal: abortController.signal // 🔑キャンセル可能にする
      });

      if (!response.ok) throw new Error('API Error');
      const data = await response.json();
      setData(data);
    } catch (error) {
      // 正常なキャンセル（AbortError）は無視
      if (error.name !== 'AbortError') {
        console.error('Fetch error:', error);
        setError(error.message);
      }
    } finally {
      setLoading(false);
    }
  };

  fetchData();

  return () => {
    abortController.abort(); // ✅ クリーンアップ時にリクエスト中断
  };
}, []);
```

### ✅ 解決法②：カスタムフック化で再利用性向上

```tsx
function useApi<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const abortController = new AbortController();

    const fetchData = async () => {
      try {
        setLoading(true);
        setError(null);
        
        const response = await fetch(url, {
          signal: abortController.signal
        });
        
        const result = await response.json();
        setData(result);
      } catch (err) {
        if (err.name !== 'AbortError') {
          setError(err.message);
        }
      } finally {
        setLoading(false);
      }
    };

    fetchData();

    return () => abortController.abort(); // URLが変わったときも中断
  }, [url]);

  return { data, loading, error };
}

// 使用例
function UserProfile({ userId }: { userId: string }) {
  const { data, loading, error } = useApi<User>(`/api/users/${userId}`);

  if (loading) return <div>読み込み中...</div>;
  if (error) return <div>エラー: {error}</div>;
  
  return <div>{data?.name}</div>;
}
```

### 📊 AbortControllerの仕組み

| 要素 | 説明 |
|------|------|
| `AbortController` | fetch()にキャンセル可能なsignalを渡すための標準API |
| `signal: abortController.signal` | fetch()に中断可能なハンドルを渡す |
| `abortController.abort()` | アンマウント時などにfetchリクエストを中断 |
| `error.name !== "AbortError"` | 正常な中断は無視し、他のエラーのみを処理 |

---

## 🌐 WebSocket・SSEのリアルタイム通信クリーンアップ

### WebSocket接続のクリーンアップ

```tsx
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/ws');
  
  ws.onopen = () => {
    console.log('WebSocket接続開始');
    setConnectionStatus('connected');
  };
  
  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    setMessages(prev => [...prev, data]);
  };
  
  ws.onerror = (error) => {
    console.error('WebSocket エラー:', error);
    setConnectionStatus('error');
  };

  ws.onclose = () => {
    setConnectionStatus('disconnected');
  };

  // 🧹 クリーンアップで接続を確実に終了
  return () => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.close();
    }
  };
}, []);
```

### Server-Sent Events (SSE) のクリーンアップ

```tsx
useEffect(() => {
  const eventSource = new EventSource('/api/events');

  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    setLiveData(data);
  };

  eventSource.onerror = (error) => {
    console.error('SSE エラー:', error);
    setConnectionStatus('error');
  };

  return () => {
    eventSource.close(); // 🧹 クリーンアップ時に接続を閉じる
  };
}, []);
```

### 動的接続の管理例

```tsx
// チャットルームなど、動的に接続先が変わる場合
useEffect(() => {
  const ws = new WebSocket(`wss://example.com/ws?roomId=${roomId}`);

  ws.onmessage = (event) => {
    const message = JSON.parse(event.data);
    setRoomMessages(prev => [...prev, message]);
  };

  return () => {
    ws.close(); // roomIdが変わったら前の接続を閉じる
  };
}, [roomId]); // 依存配列にroomIdを指定
```

### 📊 WebSocket vs SSE比較

| 通信手段 | 開始方法 | クリーンアップ方法 | 特徴 |
|----------|----------|-------------------|------|
| **WebSocket** | `new WebSocket()` | `ws.close()` | 双方向通信、手動再接続制御 |
| **SSE** | `new EventSource()` | `eventSource.close()` | 単方向通信、自動再接続機能 |

### ⚠️ クリーンアップが必要な理由

| 問題 | 起こる現象 |
|------|-----------|
| 接続を閉じない | メモリリーク・通信の浪費・イベント多重登録 |
| 再レンダー時に新接続 | 前の接続が残り、複数回同じイベントが反応 |
| 長時間アイドル後の想定外復旧 | ユーザー体験の低下、バグの温床 |

---

## 🔭 DOM Observer系APIのクリーンアップ

### IntersectionObserver のクリーンアップ

```tsx
useEffect(() => {
  const targetElement = elementRef.current;
  if (!targetElement) return;

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach(entry => {
        setIsVisible(entry.isIntersecting);
        
        // 画像の遅延読み込み例
        if (entry.isIntersecting) {
          const img = entry.target as HTMLImageElement;
          if (img.dataset.src) {
            img.src = img.dataset.src;
            img.removeAttribute('data-src');
          }
        }
      });
    },
    { 
      threshold: 0.1,
      rootMargin: '50px' // 50px手前で発火
    }
  );

  observer.observe(targetElement);

  return () => {
    observer.unobserve(targetElement); // 特定要素の監視解除
    observer.disconnect(); // 監視を完全に終了
  };
}, []);
```

### MutationObserver のクリーンアップ

```tsx
useEffect(() => {
  const targetNode = document.getElementById('dynamic-content');
  if (!targetNode) return;

  const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
      if (mutation.type === 'childList') {
        // 子要素の数をカウント
        setChildCount(targetNode.children.length);
        
        // 新しく追加された要素に対する処理
        mutation.addedNodes.forEach(node => {
          if (node.nodeType === Node.ELEMENT_NODE) {
            console.log('新しい要素が追加されました:', node);
          }
        });
      }
      
      if (mutation.type === 'attributes') {
        console.log('属性が変更されました:', mutation.attributeName);
      }
    });
  });

  observer.observe(targetNode, {
    childList: true,    // 子ノードの変更を監視
    subtree: true,      // 子孫ノードも監視
    attributes: true,   // 属性の変更も監視
    attributeOldValue: true // 変更前の値も取得
  });

  return () => {
    observer.disconnect(); // 必須！監視終了
  };
}, []);
```

### ResizeObserver のクリーンアップ

```tsx
useEffect(() => {
  const targetElement = containerRef.current;
  if (!targetElement) return;

  const observer = new ResizeObserver((entries) => {
    entries.forEach(entry => {
      const { width, height } = entry.contentRect;
      setDimensions({ width, height });
      
      // レスポンシブレイアウトの調整
      if (width < 768) {
        setIsMobile(true);
      } else {
        setIsMobile(false);
      }
    });
  });

  observer.observe(targetElement);

  return () => {
    observer.unobserve(targetElement);
    observer.disconnect();
  };
}, []);
```

### 📊 Observer系APIの比較

| API | 用途 | 監視対象 | 主な使用例 |
|-----|------|----------|-----------|
| **IntersectionObserver** | 要素の表示状態 | ビューポートとの交差 | 遅延読み込み、無限スクロール |
| **MutationObserver** | DOM構造の変更 | 要素の追加・削除・属性変更 | 動的コンテンツの監視 |
| **ResizeObserver** | 要素サイズの変更 | 要素の寸法変更 | レスポンシブレイアウト |

---

## 🛠️ 実践的なクリーンアップパターン

### パターン1：複数のクリーンアップを統合管理

```tsx
useEffect(() => {
  const cleanupFunctions: (() => void)[] = [];

  // WebSocket接続
  const ws = new WebSocket('wss://api.example.com/ws');
  cleanupFunctions.push(() => ws.close());

  // タイマー設定
  const timer = setInterval(() => {
    // 定期処理
  }, 5000);
  cleanupFunctions.push(() => clearInterval(timer));

  // Observer設定
  const observer = new IntersectionObserver(() => {});
  const target = document.getElementById('target');
  if (target) {
    observer.observe(target);
    cleanupFunctions.push(() => {
      observer.unobserve(target);
      observer.disconnect();
    });
  }

  return () => {
    cleanupFunctions.forEach(cleanup => {
      try {
        cleanup();
      } catch (error) {
        console.error('クリーンアップエラー:', error);
      }
    });
  };
}, []);
```

### パターン2：条件付きクリーンアップ

```tsx
useEffect(() => {
  let cleanup: (() => void) | undefined;

  if (isActive) {
    // アクティブ時のみリソースを使用
    const ws = new WebSocket('wss://api.example.com/active');
    const timer = setInterval(() => {
      ws.send(JSON.stringify({ type: 'heartbeat' }));
    }, 30000);

    cleanup = () => {
      clearInterval(timer);
      ws.close();
    };
  }

  return cleanup; // undefinedでも問題なし
}, [isActive]);
```

### パターン3：エラーハンドリングを含むクリーンアップ

```tsx
useEffect(() => {
  const resources: Array<{ type: string; cleanup: () => void }> = [];

  try {
    // WebSocket
    const ws = new WebSocket('wss://api.example.com/ws');
    resources.push({ 
      type: 'websocket', 
      cleanup: () => ws.close() 
    });

    // API呼び出し
    const abortController = new AbortController();
    resources.push({ 
      type: 'fetch', 
      cleanup: () => abortController.abort() 
    });

    // タイマー
    const timer = setInterval(() => {}, 1000);
    resources.push({ 
      type: 'timer', 
      cleanup: () => clearInterval(timer) 
    });

  } catch (error) {
    console.error('リソース初期化エラー:', error);
  }

  return () => {
    resources.forEach(({ type, cleanup }) => {
      try {
        cleanup();
        console.log(`${type} クリーンアップ完了`);
      } catch (error) {
        console.error(`${type} クリーンアップエラー:`, error);
      }
    });
  };
}, []);
```

---

## 🎯 クリーンアップのベストプラクティス

### ✅ 推奨事項

1. **AbortControllerを積極活用**
   ```tsx
   // ✅ 良い例
   const abortController = new AbortController();
   fetch(url, { signal: abortController.signal });
   return () => abortController.abort();
   ```

2. **カスタムフックで共通化**
   ```tsx
   // ✅ 再利用可能なフック
   function useWebSocket(url: string) {
     // WebSocket接続とクリーンアップをカプセル化
   }
   ```

3. **早期returnで不要な処理を回避**
   ```tsx
   // ✅ 条件チェック
   useEffect(() => {
     if (!shouldConnect) return;
     // 接続処理...
   }, [shouldConnect]);
   ```

4. **エラーハンドリングを忘れずに**
   ```tsx
   // ✅ 安全なクリーンアップ
   return () => {
     try {
       cleanup();
     } catch (error) {
       console.error('クリーンアップエラー:', error);
     }
   };
   ```

### ❌ 避けるべきパターン

```tsx
// ❌ クリーンアップなしの非同期処理
useEffect(() => {
  fetch('/api/data').then(setData);
}, []);

// ❌ 接続を閉じないWebSocket
useEffect(() => {
  const ws = new WebSocket('wss://example.com');
  // return文なし
}, []);

// ❌ Observerの解除忘れ
useEffect(() => {
  const observer = new IntersectionObserver(() => {});
  observer.observe(element);
  // disconnect()なし
}, []);
```

---

## 📊 クリーンアップ判断チェックリスト

| 処理の種類 | クリーンアップ | 理由 |
|-----------|---------------|------|
| `fetch`、`axios` | ✅ **必要** | リクエストキャンセル、setState防止 |
| `WebSocket`、`SSE` | ✅ **必要** | 接続解放、リソース節約 |
| `setTimeout`、`setInterval` | ✅ **必要** | タイマー解放、メモリリーク防止 |
| Observer系API | ✅ **必要** | 監視停止、メモリ解放 |
| イベントリスナー | ✅ **必要** | 重複登録防止、パフォーマンス向上 |
| 単純なstate更新 | ❌ **不要** | Reactが自動管理 |
| `console.log` | ❌ **不要** | 副作用なし |

---

## 🔧 デバッグとテスト手法

### 開発環境での確認

```tsx
useEffect(() => {
  console.log('🚀 副作用開始');
  
  const ws = new WebSocket('wss://example.com');
  const timer = setInterval(() => {
    console.log('⏰ タイマー実行中');
  }, 1000);

  return () => {
    console.log('🧹 クリーンアップ実行');
    ws.close();
    clearInterval(timer);
  };
}, []);
```

### React DevToolsでの監視

1. **Profiler**タブでメモリ使用量を監視
2. **Components**タブでuseEffectの実行回数を確認
3. **Console**でクリーンアップログを追跡

### テスト例

```tsx
import { render, unmountComponentAtNode } from '@testing-library/react';

test('コンポーネントアンマウント時にWebSocketが閉じられる', () => {
  const mockClose = jest.fn();
  global.WebSocket = jest.fn(() => ({
    close: mockClose,
    addEventListener: jest.fn(),
  }));

  const { unmount } = render(<MyComponent />);
  unmount();

  expect(mockClose).toHaveBeenCalled();
});
```

---

## 📝 まとめ

### 重要なポイント

1. **非同期処理には必ずAbortControllerを使用**
   - APIリクエストのキャンセル
   - アンマウント後のsetState防止

2. **リアルタイム通信は確実に接続を終了**
   - WebSocketは`ws.close()`
   - SSEは`eventSource.close()`

3. **Observer系APIは必ずdisconnect()**
   - IntersectionObserver、MutationObserver等
   - メモリリークとパフォーマンス低下を防止

4. **カスタムフックで再利用性を向上**
   - 共通のクリーンアップロジックを抽象化
   - テストしやすい構造に

### 理想的なuseEffectの構造

```tsx
useEffect(() => {
  // 早期return
  if (!shouldExecute) return;

  // リソースの初期化
  const abortController = new AbortController();
  const ws = new WebSocket('wss://example.com');
  const observer = new IntersectionObserver(() => {});

  // 非同期処理
  const fetchData = async () => {
    try {
      const response = await fetch('/api/data', {
        signal: abortController.signal
      });
      const data = await response.json();
      setData(data);
    } catch (error) {
      if (error.name !== 'AbortError') {
        setError(error.message);
      }
    }
  };

  fetchData();

  // クリーンアップ関数
  return () => {
    abortController.abort();
    ws.close();
    observer.disconnect();
  };
}, [shouldExecute]);
```

適切なクリーンアップ処理により、**メモリ効率が良く、予測可能で、保守しやすい**Reactアプリケーションを構築できます。

```tsx
return () => {
  // 後始末を忘れずに！
  cleanup();
};
```

未来の自分とチームのために、クリーンアップの習慣を身につけましょう 🚀
# useReducerとuseContextを使用したグローバル状態管理の簡易的な事例とその解説

- **学習日**: 2025/06/20
- **学習時間**: 40分
- **効果実感**: 4/5
- **効果の種類**: 理解が深まった

## 内容

では、useReducer + Context の構成において、useCounter の中で操作関数（アクションディスパッチャ）を定義するパターンと、それを実際に使う例を以下に示します。

⸻

✅ 1. reducer.ts

// reducer.ts
export type CounterState = {
  count: number;
};

export type CounterAction =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "RESET" };

export const initialCounterState: CounterState = {
  count: 0,
};

export function counterReducer(
  state: CounterState,
  action: CounterAction
): CounterState {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    case "DECREMENT":
      return { count: state.count - 1 };
    case "RESET":
      return { count: 0 };
    default:
      return state;
  }
}


⸻

✅ 2. counterContext.tsx

// counterContext.tsx
import {
  createContext,
  useContext,
  useReducer,
  ReactNode,
  Dispatch,
} from "react";
import {
  counterReducer,
  initialCounterState,
  CounterState,
  CounterAction,
} from "./reducer";

const StateContext = createContext(undefined);
const DispatchContext = createContext | undefined&gt;(undefined);

export const CounterProvider = ({ children }: { children: ReactNode }) =&gt; {
  const [state, dispatch] = useReducer(counterReducer, initialCounterState);
  return (
    
      
        {children}
      
    
  );
};

// 通常のstateとdispatch単体で使うフック
export const useCounterState = () =&gt; {
  const context = useContext(StateContext);
  if (!context) throw new Error("useCounterState must be used within CounterProvider");
  return context;
};

export const useCounterDispatch = () =&gt; {
  const context = useContext(DispatchContext);
  if (!context) throw new Error("useCounterDispatch must be used within CounterProvider");
  return context;
};

// 🔽 まとめてロジック関数付きで返すカスタムフック
export const useCounter = () =&gt; {
  const state = useCounterState();
  const dispatch = useCounterDispatch();

  const increment = () =&gt; dispatch({ type: "INCREMENT" });
  const decrement = () =&gt; dispatch({ type: "DECREMENT" });
  const reset = () =&gt; dispatch({ type: "RESET" });

  return {
    count: state.count,
    increment,
    decrement,
    reset,
  };
};


⸻

✅ 3. 使用例：CounterComponent.tsx

// CounterComponent.tsx
"use client";

import { useCounter } from "@/contexts/counterContext";

export default function CounterComponent() {
  const { count, increment, decrement, reset } = useCounter();

  return (
    
      <p>カウント: {count}</p>
      
        +1
        -1
        リセット
      
    
  );
}


⸻

✅ 4. app.tsxなどでProviderをラップ

// app.tsx や layout.tsx
import { CounterProvider } from "@/contexts/counterContext";
import CounterComponent from "@/components/CounterComponent";

export default function App() {
  return (
    
      
    
  );
}


⸻

🔍 このパターンのメリット
	•	useCounter() ひとつで状態と操作をまとめて取得でき、使い方が簡潔
	•	dispatch({ type: "..." }) を直接書かずに済むため、バグやタイプミス防止
	•	状態更新の共通ロジックをカスタムフック内に集約できる

⸻

それぞれのポイントについて、以下のように確認・補足します：

⸻

✅ 1. reducer.tsではaction typeとreducer関数を定義するの？

→ はい、その通りです。
	•	State（状態の型）
	•	Action（更新を表す型）
	•	reducer関数（stateとactionを受け取り、新しいstateを返す）

// reducer.ts
export type State = { count: number };
export type Action = { type: "INCREMENT" } | { type: "RESET" };

export function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    case "RESET":
      return { count: 0 };
    default:
      return state;
  }
}


⸻

✅ 2. context.tsxではuseReducerでreducer関数と初期stateをもとにstateとdispatchを作成し、その両方をcontextに渡してproviderとするの？

→ はい、まさにその構成です。

// context.tsx
const StateContext = createContext(undefined);
const DispatchContext = createContext | undefined&gt;(undefined);

export function CounterProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    
      
        {children}
      
    
  );
}


⸻

✅ 3. 状態と更新関数がProvider内であることを確認し、contextを使用するためのカスタムフック、useCounterState, useCounterDispatchを作成しているの？

→ はい、正しいです。

それぞれの Context に対して安全なアクセスを提供するカスタムフックを作成します。

export function useCounterState() {
  const context = useContext(StateContext);
  if (!context) throw new Error("useCounterState must be used within CounterProvider");
  return context;
}

export function useCounterDispatch() {
  const context = useContext(DispatchContext);
  if (!context) throw new Error("useCounterDispatch must be used within CounterProvider");
  return context;
}


⸻

✅ 4. useCounterState, useCounterDispatchの両方を使用して、状態とロジック関数（dispatchをラップした関数）をまとめて提供するuseCounterカスタムフックを作成しているの？

→ はい、これも完璧です。

export function useCounter() {
  const state = useCounterState();
  const dispatch = useCounterDispatch();

  const increment = () =&gt; dispatch({ type: "INCREMENT" });
  const reset = () =&gt; dispatch({ type: "RESET" });

  return {
    count: state.count,
    increment,
    reset,
  };
}


⸻

🔚 結論

あなたの理解に誤りはありません。すべてのポイントでReactの useReducer + Context による状態管理の構成を正確に捉えています。
この構成は中〜大規模の状態管理においてとても有効で、後からZustandなどに移行する際にも基礎として役立ちます。

さらに知りたいことがあれば、以下のような関連テーマもお伝えできます：
	•	非同期処理を含む dispatch のパターン（thunk的な実装）
	•	複数 reducer/context の構成パターン
	•	Zustand や Redux Toolkit との比較ポイント
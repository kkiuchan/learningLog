# forwardRefとuseImperativeHandleを使用して親コンポーネントからアクセスできるメソッドをカスタムする

- **学習日**: 2025/06/19
- **学習時間**: 30分
- **効果実感**: 4/5
- **効果の種類**: 理解が深まった

## 内容



✅ あなたの理解と補足

🔹 forwardRef の役割

「親から ref を受け取ることのできるカスタムコンポーネントを定義するもの」

➡️ その通りです。
	•	通常の関数コンポーネントは ref を props として受け取れません。
	•	forwardRef を使うことで、親から渡された ref を内部の要素や任意のロジックに接続できるようになります。

📌 つまり、forwardRef は 「ref をバイパスして転送する仕組み」。

⸻

🔹 useImperativeHandle の役割

「forwardRef の定義の中で、親から渡される ref に対して使用可能なメソッドを制限・カスタム定義するもの」

➡️ これもその通りです。
	•	通常、ref.current は DOMノードそのものになります（例：HTMLInputElement）。
	•	useImperativeHandle を使うと、ref.current に任意のオブジェクト（カスタムAPI）を割り当てることができる。

📌 つまり、useImperativeHandle は 「ref 経由で外部に公開するインターフェースを制御する仕組み」。

⸻

💡 例えるなら：

機能	役割の比喩
forwardRef	「親に向けてドア（refの入り口）を用意する」
useImperativeHandle	「そのドアの向こうでどんなボタン（API）を操作できるかを決める」


⸻

✨ 補足ポイント
	•	forwardRef だけで使う場合、ref.current は 素の DOMノード。
	•	useImperativeHandle を組み合わせると、任意のメソッドや値を持つ オブジェクトにすり替えられる。
	•	TypeScriptと併用すると、型安全に公開APIを制限できる（これが非常に便利）。

⸻

TypeScriptと併用する forwardRef と useImperativeHandle の実装パターンを、以下の2つに分けて解説します：

⸻

✅ パターン1：forwardRef のみを使って DOM ノードを渡す（基本パターン）

import React, { forwardRef, useRef, useEffect } from 'react';

const SimpleInput = forwardRef&gt;(
  (props, ref) =&gt; {
    return ;
  }
);

// 親コンポーネント
function Parent() {
  const inputRef = useRef(null);

  useEffect(() =&gt; {
    inputRef.current?.focus(); // 直接DOMを操作
  }, []);

  return ;
}

🔍 特徴
	•	ref.current は HTMLInputElement | null。
	•	直接 .focus(), .value, .blur() などの DOM API が使える。

⸻

✅ パターン2：useImperativeHandle を使ってカスタムAPIを提供する（拡張パターン）

import React, {
  forwardRef,
  useRef,
  useImperativeHandle,
  useEffect,
} from 'react';

// 公開したいメソッドの型
type CustomInputHandle = {
  focus: () =&gt; void;
  clear: () =&gt; void;
};

// props には input 要素の標準属性を渡せるようにしておく
const CustomInput = forwardRef&lt;
  CustomInputHandle,
  React.InputHTMLAttributes
&gt;((props, ref) =&gt; {
  const inputRef = useRef(null);

  // ref に公開するメソッドを定義
  useImperativeHandle(ref, () =&gt; ({
    focus: () =&gt; inputRef.current?.focus(),
    clear: () =&gt; {
      if (inputRef.current) inputRef.current.value = '';
    },
  }));

  return ;
});

// 親コンポーネント
function Parent() {
  const ref = useRef(null);

  useEffect(() =&gt; {
    ref.current?.focus(); // カスタムAPI経由でアクセス
  }, []);

  return (
    &lt;&gt;
      
       ref.current?.clear()}&gt;クリア
    &lt;/&gt;
  );
}

🔍 特徴
	•	ref.current は DOM ではなく、任意のメソッド群を持つオブジェクト。
	•	外部に公開したい振る舞いだけを厳選して提供できる。
	•	inputRef はコンポーネント内部専用で隠蔽されている。

⸻

✅ 比較まとめ

特徴	forwardRefのみ	forwardRef + useImperativeHandle
ref.current の中身	DOMノード（例: HTMLInputElement）	任意のメソッド群（カスタムAPI）
アクセスできる操作	.focus(), .value など	.focus(), .clear() など、制御されたAPI
型の安全性	DOMの型に依存	自分で型定義できる（型安全）
カプセル化	弱い（DOM操作を親に渡す）	強い（内部構造を隠蔽できる）


⸻

✨ どんなときにどちらを使う？
	•	🔹 DOM操作だけで十分なとき → forwardRef のみ
	•	🔹 コンポーネント独自の操作を提供したいとき → useImperativeHandle を併用

⸻
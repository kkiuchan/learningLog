# useImperativeHandleについて

- **学習日**: 2025/06/19
- **学習時間**: 25分
- **効果実感**: 3/5
- **効果の種類**: 理解が深まった

## 内容

useImperativeHandle 徹底解説（TypeScript対応）

React でカスタムコンポーネントに対して「親から操作できるメソッド」を提供したい場面はよくあります。そのときに活躍するのが useImperativeHandle フックです。

この記事では、基本的な使い方から TypeScript との併用パターン、活用時の注意点までをわかりやすく解説します。

⸻

🔰 useImperativeHandle とは？

useImperativeHandle は、forwardRef と組み合わせて使用される React のフックで、ref 経由で外部に公開したいメソッドや値をカスタム定義するためのものです。

通常、ref は DOM ノード（例: input 要素）を直接参照しますが、useImperativeHandle を使うことで、自分で定義したメソッド群を ref.current に割り当てることができます。

⸻

🧩 使い方の基本（TypeScript対応）

import React, {
  useRef,
  forwardRef,
  useImperativeHandle,
} from 'react';

type CustomInputHandle = {
  focus: () =&gt; void;
  clear: () =&gt; void;
};

const CustomInput = forwardRef&gt;(
  (props, ref) =&gt; {
    const inputRef = useRef(null);

    useImperativeHandle(ref, () =&gt; ({
      focus: () =&gt; inputRef.current?.focus(),
      clear: () =&gt; {
        if (inputRef.current) inputRef.current.value = '';
      },
    }));

    return ;
  }
);

function Parent() {
  const ref = useRef(null);

  return (
    &lt;&gt;
      
       ref.current?.focus()}&gt;フォーカス
       ref.current?.clear()}&gt;クリア
    &lt;/&gt;
  );
}


⸻

✅ 型安全の利点
	•	ref.current にアクセスするときに 自動補完が効く。
	•	不正なプロパティアクセスがあると コンパイルエラーで検出できる。
	•	大規模プロジェクトでも ドキュメント代わりになる。

⸻

🚧 注意点・ベストプラクティス

注意点	内容
ref の型は省略しない	useRef(null) のように型注釈を必ず付ける
返すオブジェクトの形を一致させる	型で定義したすべてのメソッドを返すようにする
UI操作とロジックの責務分離	UIの表示よりも「操作APIの公開」に特化させる


⸻

📚 応用アイデア
	•	入力検証ロジック（validate()）の公開
	•	カレンダーの外部制御（open(), close()）
	•	チャートやグラフコンポーネントの resetZoom()
	•	モーダルコンポーネントの open() close()

⸻

📝 まとめ

要素	内容
useImperativeHandle	ref経由で任意のメソッド/値を公開できる
TypeScriptとの相性	型で公開APIを制限・補完・検出できる
必須条件	forwardRefとセットで使う必要がある

カスタムコンポーネントの再利用性と安全性を高めるために、useImperativeHandle は非常に強力なツールです。特に TypeScript と併用することで、堅牢な API 設計が可能になります。

⸻

さらに応用例や UI ライブラリ（shadcn/ui や Headless UI）との組み合わせ例が知りたい方は、ぜひご相談ください！
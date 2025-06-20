# ReactのforwardRefについて

- **学習日**: 2025/06/19
- **学習時間**: 40分
- **効果実感**: 4/5
- **効果の種類**: 理解が深まった

## 内容

# ReactのforwardRefを徹底解説！

ReactでフォームやUIコンポーネントをカスタムする中でよく登場するのが `forwardRef`。特に、最近よく使われるUIライブラリ「shadcn/ui」でも多用されており、理解しておくと非常に便利です。

この記事では、`forwardRef` の基本から実用例、shadcn/uiとの関係まで、わかりやすくまとめていきます。

---

## 🔰 forwardRefとは？

`forwardRef` は React が提供する **関数コンポーネントに `ref` を渡すための仕組み** です。通常、関数コンポーネントは `ref` を props のように受け取ることができません。

### ❌ 通常の関数コンポーネント

```tsx
function MyInput(props) {
  return ;
}

const ref = useRef();
; // これは動かない！
```

### ✅ forwardRefを使ったコンポーネント

```tsx
const MyInput = React.forwardRef((props, ref) =&gt; {
  return ;
});

const ref = useRef();
; // OK！
```

このように、`forwardRef` を使うことで `ref` を内部の DOM 要素に渡せるようになります。

---

## 🎯 実用例：親から子のinputにフォーカスを当てる

```tsx
const Input = React.forwardRef(
  (props, ref) =&gt; {
    return ;
  }
);

function Parent() {
  const inputRef = useRef(null);

  const focusInput = () =&gt; {
    inputRef.current?.focus();
  };

  return (
    &lt;&gt;
      
      フォーカス
    &lt;/&gt;
  );
}
```

* `forwardRef` を使った `Input` コンポーネントに `ref` を渡すことで、
* 親のボタンから `inputRef.current.focus()` を実行できるようになります。

---

## 🧩 JSX.IntrinsicElements\['input']とは？

```tsx
React.forwardRef
```

これは TypeScript で型安全に `` の属性（type, onChange など）を受け取れるようにするための書き方です。`IntrinsicElements` は `JSX` 内の標準要素とその props 型の辞書のようなものです。

---

## 🚀 shadcn/uiとforwardRef

人気の UI ライブラリ `shadcn/ui` では、コンポーネントが多くのユースケースに柔軟に対応するために `forwardRef` が頻繁に使われています。

```tsx
const Button = React.forwardRef&lt;
  HTMLButtonElement,
  React.ButtonHTMLAttributes
&gt;(({ className, ...props }, ref) =&gt; {
  return (
    
  );
});
```

このように、shadcn/ui のコンポーネントは DOM 操作が必要な場面（モーダルの初期フォーカス制御など）でも `ref` を通じて制御できるように設計されています。

### ✅ 実際に活用されている例：

* フォームで初期フォーカスを当てる
* モーダルやポップオーバーの中で、特定の要素にスクロールやクリックを当てる
* アニメーション制御や測定系のカスタム処理

---

## 📝 まとめ

| 項目              | 内容                                                      |
| --------------- | ------------------------------------------------------- |
| forwardRefの役割   | 親から子のDOMにrefを渡せるようにする                                   |
| 使い方             | `React.forwardRef((props, ref) =&gt; )` |
| TypeScriptでの型付け | `forwardRef` を指定する                        |
| shadcn/uiとの関係   | 多くのコンポーネントが `forwardRef` を使っており、カスタマイズや制御がしやすい          |

---

Reactの柔軟で強力な `ref` 制御を活用する上で、`forwardRef` は避けて通れない基礎知識です。shadcn/ui のようなコンポーネントライブラリと組み合わせて活用することで、より洗練されたUI体験を実現できます！
---
title: "最速攻略！　Reactの `use` RFC"
emoji: "🔬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

皆さんこんにちは。最近のReact界隈で話題になっているのは次のRFCです。

https://github.com/reactjs/rfcs/pull/229

そこで、この記事ではさっそくRFCを理解することを目指します。

ただし、このRFCはSuspenseに深く関わるものです。SuspenseはReact 18でもう正式リリースされていますから、この記事ではSuspenseは前提知識とします。もしまだSuspenseをよく知らないのであれば、ぜひ次の記事で学習してください。

https://zenn.dev/uhyo/books/react-concurrent-handson

また、RFCはあくまでReactの新機能のアイデアを公開するものであり、これが必ず実装されるとは限らない点にご注意ください。例えば、過去には`useEvent`というRFCが注目を集めていましたが、意見が集まった結果としてそのRFCは実装されずにクローズされました（RFCが無駄だったというわけではなく、再度検討してよりアイデアがブラッシュアップされることになります）。

# 新しい `use` API

このRFCには大きく分けて2つの特徴があります。一つはServer Componentsに関係するもので、もう一つはどんなコンポーネントでも使えるものです。現在の状況だと後者のほうに興味がある方が多いでしょうから、そちらを中心に据えて説明します。

このRFCでは、新しい`use`という関数が実装されます。RFCの説明では、これは「**特殊なフック**」であるとされています。使い方は次のようになります（RFCから引用）。

```ts
function Note({id, shouldIncludeAuthor}) {
  const note = use(fetchNote(id));

  let byline = null;
  if (shouldIncludeAuthor) {
    const author = use(fetchNoteAuthor(note.authorId));
    byline = <h2>{author.displayName}</h2>;
  }

  return (
    <div>
      <h1>{note.title}</h1>
      {byline}
      <section>{note.body}</section>
    </div>
  );
}
```

`fetchNote`や`fetchNoteAuthor`の返り値であることから察せられるように、`use`に渡されるのはPromiseです。コードを見ると、`use`にPromiseを渡すとその中身を取得できています。おおよそ、`use`のシグネチャは次のようであると考えられます。

```ts
const use: <T>(promise: Promise<T>) => T;
```

つまり、`await`のようにPromiseから中身を取り出すのが`use`の役目ということです。

`use`に渡されたPromiseが未解決だった場合の挙動は従来と同じです。つまり、その場合は`use`の返り値を用意できないため、コンポーネントのサスペンドが発生してその関数の実行は中断され、Promiseが解決してから再試行されます。

従来のSuspenseと異なる点は、従来はReactは「投げられたPromiseが解決したら再度レンダリングを試みる」ことのみをサポートしており、Promiseの中身を取り出すことは我々に任されていました。それに加えて、`use`ではPromiseの中身を取り出すところまでやってくれるのが新しい点です。

従来はReactのSuspense機構を生で使うのが難しく、何らかのライブラリを経由して使うのが主流でした。`use`の登場により簡単なケースならライブラリを使わずにPromiseを取り扱えるようになります。

## 特殊なフック？

RFCでは、`use`はフックの一種であるとされています。しかし、`use`は従来のフックと違うところがあります。それは、**条件分岐内でも使用してよい**ということです。上の例でも、if文の中で`use`が使われているのが見て取れます。

Reactにおけるフックのルールは次の2つに大別されました。

- 関数コンポーネントの中（および関数コンポーネントから呼び出されるカスタムフックの中）でしか使用できない。
- 常に同じ順序で同じ数だけ呼ばれなければならない。（＝条件分岐により、呼んだり呼ばなかったりすることはできない）

`use`は、この2つのルールのうち前者のみを制約として持ち、後者のルールは適用されないことになります。

なぜ`use`に後者のルールが適用されないのかは、そもそもなぜこのようなフックのルールが存在しているのかを考えれば理解できます。

まず、前者のルールは「フックはコンポーネントに属する」ことから説明できます。例えば`useState`はコンポーネントのステートを宣言するものですから、コンポーネントの外で使うとどのコンポーネントのステートを宣言しているのか分からないので意味がありません。また、`useContext`は、コンポーネントがコンポーネントツリーのどこに存在するのか分からないとコンテキストの値を取得できませんから、これもコンポーネントに属しています。

次に後者のルールは「フックがコンポーネントの記憶領域にアクセスする」ことから説明できます。`useState`がコンポーネント内にステートを用意するのはもちろん、`useRef`や`useMemo`などもコンポーネント内の記憶領域を利用しています。他にも、`useEffect`もクリーンアップ関数がコンポーネント内の記憶領域に保存されていると考えられます。クラスコンポーネント時代に`this`として自由にアクセスできた記憶領域が、関数コンポーネントではフックの裏に隠されていると考えると分かりやすいでしょう。

フックのAPIの特徴は、コンポーネントの記憶領域内のデータに名前を付けないということです。代わりに、「何番目のフック用の記憶領域なのか」を頼りにフックの記憶領域の読み書きが行われます。この点については詳しい解説記事が探せばあると思うのでここでは詳しい説明を省略します。

本題に戻ると、なぜ`use`には後者のルールが適用されないのでしょうか。それは「コンポーネント内の記憶領域を使用しないから」であると考えられます。`use`は与えられたPromiseの中身を取り出すだけなので、実は記憶領域が必要ありません。

一方で、前者の制約は必要です。なぜなら、Promiseがまだ解決していない場合にコンポーネントをサスペンドさせる必要があるからです。

以上のことを考えると、`use`は特殊なフックというよりも、フックの新しい分類を立ち上げる存在であると考えられます。従来のフックたちを「記憶領域を必要とするフック」として、新たに「記憶領域を必要としないフック」という分類ができたというイメージです。

両者に別々の名前を付けてもよいと思うのですが、用語を増やすとユーザーが混乱するでしょうからそれは避けたのではないかと思います。また、`use`以外に今後「記憶領域を必要としないフック」が出てくるかどうかは不透明なので（それを防ぐために`use`という超汎用的な名前を付けたとも推測できます）、今回`use`は「特殊なフック」という立ち位置にしたのでしょう。

# `use`は何を解決するのか

以下の記事を読んだ方は、ReactのSuspenseを利用してコンポーネントを書く方法を理解したはずです。

https://zenn.dev/uhyo/books/react-concurrent-handson

この記事を読むと分かる通り、コンポーネントのサスペンドを有効に活用するためにはコンポーネントの外部にデータを保存することが必要でした。上記の記事の6章「[コンポーネントの外部にデータを持とう](https://zenn.dev/uhyo/books/react-concurrent-handson/viewer/data-fetching-2)」では、データの保存のためにグローバルなキャッシュキーを用意する必要があると説明しました。実際にこれは現在[SWR](https://swr.vercel.app/)や[TanStack Query](https://tanstack.com/query/v4/)（元React Query）で使われているアプローチです。

また、Promiseをthrowするとコンポーネントが必ずサスペンドするので、コンポーネントをサスペンドさせるかどうかの判断をするためには「読み込み完了したかどうか」というフラグを別途持っておくことが必要です。上記の記事の7章「[Render-as-you-fetchパターンの実装](https://zenn.dev/uhyo/books/react-concurrent-handson/viewer/render-as-you-fetch)」においても、Promiseと読み込み完了フラグをセットにした`Loadable`というデータ構造を紹介しました。

以上のことは、**Promiseが一級市民ではなかった**と説明することができます。Promiseは、JavaScriptの言語仕様においては非同期処理そのものを表す汎用性の高いオブジェクトです。しかし、ReactのSuspenseの文脈においては今のところコンポーネントをサスペンドさせる道具という程度の位置づけです。そのため、アプリケーションロジックにおいて便利に使われるPromiseをコンポーネントツリーに持ち込む際には、`useSWR`, `useQuery`あるいは`Loadable`といった中間層が必要になっていました。

今回のRFCで導入される`use`はこのギャップを埋めてくれます。Reactコンポーネント内から直接Promiseの中身を取り出せるようにすることで、Reactコンポーネント内においてもPromiseを「非同期処理そのもの」として取り扱うことができます。筆者は、**中間層を除去することによってReactとPromiseの親和性が向上し、Reactにおける非同期処理の取り扱いがよりエンジニアにとって分かりやすくなる**と期待しています。

RFCのタイトルもよく見ると「First class support for promises and async/await」となっています。これは、上で説明したように、ReactがPromiseを直接サポートするということを表しているのでしょう（async/awaitがどう関わってくるのかについてはもう少しあとで説明します）。

## `use`はキャッシュと組み合わせよう

RFCには次のような注意が書かれています。これが意味するところについて説明します。

> Caveat: Data requests must be cached between replays

冒頭の`use`の例を再掲します。

```ts
function Note({id, shouldIncludeAuthor}) {
  const note = use(fetchNote(id));

  let byline = null;
  if (shouldIncludeAuthor) {
    const author = use(fetchNoteAuthor(note.authorId));
    byline = <h2>{author.displayName}</h2>;
  }

  return (
    <div>
      <h1>{note.title}</h1>
      {byline}
      <section>{note.body}</section>
    </div>
  );
}
```

ここで使われている`fetchNode`の実装がこんな感じだとすると、このコンポーネントは再レンダリングのたびに再び`fetch`が発火してしまうことになります。例えば、`id`はそのままで`shouldIncludeAuthor`だけ変化した場合も、`fetchNote(id)`が再実行されます。それどころか、`use`はサスペンド後に関数コンポーネントを再実行するので、1回のレンダリングでも複数回発火してしまいます。

```ts
const fetchNote = async (id: string) => {
  const res = await fetch(`/api/note/${id}`);
  return res.json();
};
```

:::message
この記事では当初「複数回発火するので再びサスペンドしてしまう」と説明していましたが、それは誤りでした。コメントでのご指摘ありがとうございました。`use`によるサスペンドで関数コンポーネントが再実行された際は、新しく作られたPromiseは無視されて以前のPromiseの結果が利用されるので、再サスペンドとはなりません。これは`useState`に渡された初期値が2回目以降無視されるのと似ていますね。
:::

このように、`use`に対してPromiseを渡すという性質上、うまくやらないと無駄な非同期処理が発生してしまいます。例えば同じ`id`に対するリクエストは一定時間キャッシュしておくというような、何らかのキャッシュの機構はいまだに必要だということです。

「それなら1回だけ`fetchNote`を呼ぶように制御すればいいじゃん」と思われそうですが、Reactの思想的にはコンポーネントは極力冪等にして、キャッシュを使ってパフォーマンスを確保してほしいようです。また、コンポーネントの最初のレンダリングでサスペンドする場合を考えると、どのみちコンポーネントの外部にデータを持つ必要があるため、面倒くささはそこまで減っていません。

キャッシュをわざわざ導入するのは面倒くさいように思えますが、現在のところSuspenseはそもそも`useQuery`などサードパーティのライブラリと組み合わせて使うことが多く、その場合はこのようなキャッシュ制御は元々ライブラリが行ってくれています。

つまるところ、非同期データの読み込みに関する役割の分担は、`use`の登場前後で次のように変化することになります。

| | 処理 | 出力 | `use`前の分担 | `use`後の分担 |
| - | - | - | - | - |
| 1 | 非同期データの読み込み | Promise | ユーザーのコード | ユーザーのコード |
| 2 | 非同期データのキャッシュ | Promise | サードパーティ（`useQuery`など） | サードパーティ（`useQuery`など） |
| 3 | コンポーネントをサスペンドさせるかどうかの制御 | Promiseをthrowする/しない | サードパーティ（`useQuery`など） | React本体 |

つまり、非同期データの読み込み中にコンポーネントをサスペンドするという一連の処理を1～3に分けるとすると、`use`の登場によって3がサードパーティのライブラリからReact本体に移管されることになります。

そうなると、`useSWR`や`useQuery`を使用する顕著な理由として残るのはキャッシュの制御をしてくれる点だということになりますね。

また、コンポーネントをサスペンドさせるかどうかの制御をしてもらうという責務を`useSWR`や`useQuery`から除去したとすると、これらがフックである理由はもはや無くなります。ライブラリ側からすると、「Promiseをthrowする」というReact特有のプロトコルに縛られる必要が無くなり、単なるPromiseを出力すれば良くなります。これにより、ライブラリ側はもはやReact専用のAPIを提供する必要がなくなり、ライブラリ側にもメリットがあります。

このように、ライブラリとReact本体の間のインターフェースが単なるPromiseになったというのも目覚ましい進化だと言えます。これもPromiseの一級市民化の一環です。

さらに言えば、実はRFCでは`use`と相性の良い`cache` APIのRFCも出てくるということが予告されています。つまり、上の表の2もReact本体に移管される可能性があります。

そうなると、非同期データフェッチングのライブラリは不要になるか、あるいは「キャッシュ戦略」を提供する薄いライブラリとして残るという未来が予想できます。その方向性にベットしてReactアプリケーションを設計するのも悪くない選択でしょう。

# `use`のためのReactコアの変化

`use`はただ新しいAPIが実装されるというだけではなく、それに対応するためにReactのコアにも変化が加えられます。

`use`のRFCではReactがPromiseの中身の読み取りを担当するということを思い出してください。つまり、キャッシュの機構が入ったとしても、非同期処理の結果はPromiseでよいということになります。すでにキャッシュされていた場合はすでに解決されたPromiseが返されます。

このように、キャッシュの有無にかかわらずPromiseが結果となるというのはインターフェースの簡潔化に有効であり、JavaScriptにおいてasync関数が常にPromiseを返すという事情にも適合し、JavaScriptフレンドリーです。

ところが、ひとつ問題があります。それは、JavaScriptにおいてはPromiseから同期的に値を読みだす方法が存在しないということです。すでに解決済みのPromiseだとしても、かならず非同期処理で読みだす必要があります。

```js
// 解決済みのPromiseを作成
const promise = Promise.resolve("chu");
// Promiseから値を読みだす
promise.then((value) => console.log(value));
// 同期的な処理
console.log("pika");
```

上の例では「`pika`」「`chu`」の順に出力されます。このように、すでに解決済みのPromiseだとしても、同期的な処理のほうが読み出しより先に処理されます。

一方で、Reactのレンダリングは同期処理です。これは、関数コンポーネントがasync関数ではなく普通の関数であることから分かります。

これが意味するところは、たとえ解決済みのPromiseだとしても、Reactコンポーネントのレンダリング中にPromiseの中身を取得することは不可能だということです。従来のReactではこれに対して「読み込み完了しているならPromiseをthrowしない」という形で対処していましたが、これはPromiseの読み込み完了状態をPromiseとは別に持っている必要があるのでPromiseが一級市民とは言えませんでした。

## `use`の魔法

前の例を思い返すと、同期的に実行されるコンポーネントの中で`use`を使うとPromiseの中身が同期的に読み出せるというAPIになっていました。普通に考えるとこれは不可能なので、この魔法のような挙動を実現するためにReactが裏で何かやってくれているということになります。

もちろん`use`もSuspenseをベースとしているので、Promiseが読み込み中だったときはコンポーネントをサスペンドさせます（そのPromiseをthrowしたときと同等の挙動）。その場合、Promiseが解決したときは再度レンダリングが行われます。

ここで問題となるのは、リクエストがすでにキャッシュされていた場合において`use`に渡されるのは「すぐに解決されるPromise」であるということです。つまり、Reactは`use`に渡されたPromiseが「すぐに解決されるかどうか」を判別して、サスペンドするかどうか判断する必要があるということです。

この判断機構が、`use`に伴ってReactコアに新たに実装される機能であると考えられます。そして、これがどのように行われるのかについては実はRFCをよく読むと書いてあります。

> What we can do in this case is rely on the fact that the promise returned from fetchTodo will resolve in a microtask. Rather than suspend, React will wait for the microtask queue to flush. If the promise resolves within that period, React can immediately replay the component and resume rendering, without triggering a Suspense fallback. Otherwise, React must assume that fresh data was requested, and will suspend like normal.

つまり、「Promiseがすぐに解決される」ということを「レンダリング後にマイクロタスクキューが全部消化されるまでに解決される」として定義し、この場合はコンポーネントのレンダリングが成功したと判断し、サスペンドを省略します。

ただし、Promiseの中身が判明してからコンポーネントの再レンダリングをするという機構は依然として必要となるので、1回のレンダリングで関数コンポーネントが複数回呼び出されるのは従来と変わりません。

従来のサスペンドにおける「レンダリング→サスペンド発生→完了したら再レンダリング」というフローは変わらないまま、これが一瞬で終了したらサスペンド扱いにならずにレンダリング成功と見なされるようになると理解しましょう。

これにより、従来は完全に同期的だった「レンダリング」という処理が、マイクロタスクレベルの遅延であれば待ってくれるという意味で、非同期的な処理になったと考えることができます。この点が今回のRFCの本質的なポイントでしょう。

## `use`の裏側と記憶領域

この記事の前半で「`use`はコンポーネント内の記憶領域を必要としないので条件分岐の中で呼び出すことができる」と説明しました。しかし、`use`が記憶領域をまったく使用しないというわけではありません。

`use`に渡されたPromiseの結果はどこかに保存されており、再レンダリング時は`use`はそちらの記憶領域から結果を読み込んで返すことによって、`use`にPromiseを渡すとその中身が取り出されるという挙動を実現しています。

そうなると結局記憶領域が必要になりますね。しかし、ここで使われる記憶領域は「コンポーネント単位」ではなく「レンダリング単位」です。つまり、`use`用の記憶領域はレンダリング時に確保され、そのレンダリングが成功裡に完了すれば記憶領域は破棄されます。

そして、`use`も普通のフックと同様に「何番目の呼び出しか」に依存して記憶領域からデータを読みだします。

それにも関わらず`use`を条件分岐の中で使用できるのは、**レンダリングの最中に条件分岐の結果が変わることはないという仮定**を設けているからです。Reactコンポーネントはもともと純粋性（propsやstateが同じなら同じ結果になること）が必要とされています。この性質から、propsやstateが同じならコンポーネント内の計算も同じように行われるはずで、if文の分岐が前回と異なる方向に進むことはないだろうから、分岐の中で`use`を使われても大丈夫ということです[^note_purity]。

[^note_purity]: 関数の結果だけでなく実行過程にまで制約がかかるという点がやや特殊で、純粋性の定義によっては納得できないかもしれません。副作用がないという意味での純粋性であれば、多分実行過程も同じになりそうです。

次の例（再掲）においても、if文の中を通るかどうかは`shouldIncludeAuthor`というpropのみに依存しているので、propsが変わらなければ`use`の実行回数や順序も変わらないはずです。この仮定により、条件分岐の中で`use`を使っても問題ないのです。

```ts
function Note({id, shouldIncludeAuthor}) {
  const note = use(fetchNote(id));

  let byline = null;
  if (shouldIncludeAuthor) {
    const author = use(fetchNoteAuthor(note.authorId));
    byline = <h2>{author.displayName}</h2>;
  }

  return (
    <div>
      <h1>{note.title}</h1>
      {byline}
      <section>{note.body}</section>
    </div>
  );
}
```

逆に言えば、純粋性を崩す形で条件分岐の中で`use`を使うのはやはり不可能です。例えば次のようにするとうまく動かないはずです。

```ts
// これはだめ
const note = use(fetchNote(id));

if (Math.random() < 0.1) {
  // このようにuseを使うのはだめ
  const rareData = use(fetchRareData(id));
  return (...);
}

return (...);
```

非純粋なReactコンポーネントを書く人は今どきいないとは思いますが、このようにReactの新しいAPIはどんどん純粋性の上に乗っかってきています。純粋なコンポーネントを書く習慣を大事にしましょう。

ちなみに、「コンポーネント単位の記憶領域」は、コンポーネントが初回レンダリング時にサスペンドしてしまった場合は破棄されます。これにより、`use`のような機能を現在のReactの機能を使って再現するのが難しくなっています。`use`はコンポーネント単位ではなくレンダリング単位の記憶領域という概念を導入してこれを克服しているのです。この点が、わざわざReactのコアに`use`を導入する理由となります。`use`により、（Promiseを返す側でキャッシュが依然として必要になるのはすでに説明した通りですが）Promiseの状態をトラッキングする処理をコンポーネントの外部で行う必要がなくなるのです。

## 余談: `use`のほかの用法

`use`はコンポーネント単位の記憶領域を使用しないという点で従来のフックとは異なると説明しました。

実は、皆さんが普段よく使う既存のフックの中にも、コンポーネント単位の記憶領域を使用しないものが紛れています。そう、それは`useContext`です。

本質的には`useContext`は記憶領域を使用しないため条件分岐の中で使ったりしても別に問題なかったのですが、フックという統一された機構の上に乗ったために、`useContext`にも「条件分岐の中で使えない」というルールが適用されていたのです。

ということで、RFCでは`use`に対する将来的な拡張として、`use(context)`とすることで`useContext`と同様にコンテキストの中身を取り出せるようになるかもしれないと述べられています。もちろん、`use(context)`は`useContext(context)`とは異なり、条件分岐の中で使用可能です。

`useContext`についてはたまたま`use(context)`としても違和感のないAPIであるため`use`に統合されそうですが、それ以外に記憶領域が必要ないフックが現れたときに`use`に続く第2の「条件分岐の中でも使えるフック」となるかどうかは不透明です。わざわざ`use`という特別な名前をここで持ち出したとなれば、そのような方向性には進まなさそうにも思われますが。

# サーバーサイドasyncコンポーネント

ここで、`use`を離れて次の話題に移ります。同じRFCでは「Server Componentではコンポーネントを`async`関数にできる」という提案もされています。

軽く復習しておくと、React Server Componentsではコンポーネントがサーバーサイド用とクライアント用に分類され、両者のレンダリング結果を組み合わせることでReactアプリケーション全体のレンダリングが完成するという構成になります。React Server Componentsはまだ正式リリースされていません。

このRFCでは、サーバー側のコンポーネントをこのように`async`で書けるとされています（RFCから引用）。

```ts
async function Note({id, isEditing}) {
  const note = await db.posts.get(id);
  return (
    <div>
      <h1>{note.title}</h1>
      <section>{note.body}</section>
      {isEditing ? <NoteEditor note={note} /> : null}
    </div>
  );
}
```

`async`関数であるということは、Promiseの中身を得るために`use`ではなく標準のawaitを使えるということです。Server Componentの主要なユースケースとしてデータの取得がありますから、これは相性がよいですね。

この機能の導入により、コンポーネントでPromiseを扱うベストプラクティスがサーバーサイド（await）とクライアントサイド（`use`）で異なることになりますが、async/awaitの導入はそのデメリットを上回るメリットがあると判断されたことからこのRFCに至りました。詳しい理由の説明はRFCにありますから気になる方は読んでみましょう。

ちなみに、RFCを読む限り、`async`コンポーネントが返したPromiseは普通に解決されるまで待たれます。サスペンドとかそういうややこしい概念はありません。

その裏返しとして、`async`コンポーネントではフックが使えません。フックが出る前の関数コンポーネントのような味わいです。これについては、そもそもサーバーコンポーネントはステートレスな計算をユースケースとしており、`useState`や`useEffect`などは元々サーバーコンポーネントでは使えなかったので大した問題ではありません。RFCでも言及されていますが、`async`コンポーネントと`useId`を組み合わせられないのがちょっと不便な程度だと思います。

RFCではFAQとして「クライアントサイドでも`async`コンポーネントをサポートしないのか？」という質問があり、技術的には可能だが、利用にあたって注意すべき点が多くなってしまうので推奨していないという旨の説明がされています。こちらも詳しくはRFCを読んでみましょう。

# まとめ

この記事ではReactの新しいRFCに記述された`use`APIについて説明しました。

`use`を使う場合、従来Reactコンポーネントで非同期処理を扱う際に必要だった「Promiseをthrowする」というプロトコルをReact内部に隠蔽することができます。それに伴って、Reactとサードパーティライブラリの間のインターフェースが単なるPromiseの受け渡しになります。

Reactを使うエンジニアにとっては、慣れ親しんだPromiseをReactが直接サポートしてくれるのは嬉しいし、Suspenseを取り扱う際にライブラリを介さなくても良いケースが増えるのも魅力的です。一方、ライブラリ側にとってもReact特有のプロトコルをわざわざサポートする必要がなく責務が単純になるという利点があります。

Reactはこれまで、サードパーティライブラリと協調しながら何をReactのコアに導入すべきか慎重に検討してきました。今回のRFCもその流れに乗り、サードパーティライブラリの負担を軽減してくれるものとなっています。その点で、今回もReactらしいRFCだと言えるでしょう。筆者としては、ぜひ導入されてほしいと思います。

好評につき続編ができました↓

https://zenn.dev/uhyo/articles/react-use-rfc-2
# Observable

Observables は、複数の値のレイジープッシュコレクションです。 次の表の欠落している箇所を埋めます:

| | Single | Multiple |
| --- | --- | --- |
| **Pull** | [`Function`](https://developer.mozilla.org/en-US/docs/Glossary/Function) | [`Iterator`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) |
| **Push** | [`Promise`](https://developer.mozilla.org/en-US/docs/Mozilla/JavaScript_code_modules/Promise.jsm/Promise) | [`Observable`](/api/index/class/Observable) |

**Example.**  以下は、サブスクライブ時にすぐに（同期的に）値 `1`, `2`, `3` をプッシュし、 subscribe の呼び出しから 1 秒が経過した後に値 `4` をプッシュして完了する Observable です。

```ts
import { Observable } from 'rxjs';

const observable = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
});
```

Observable を呼び出してこれらの値を表示するには、それに *サブスクライブ* する必要があります:

```ts
import { Observable } from 'rxjs';

const observable = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
});

console.log('just before subscribe');
observable.subscribe({
  next(x) { console.log('got value ' + x); },
  error(err) { console.error('something wrong occurred: ' + err); },
  complete() { console.log('done'); }
});
console.log('just after subscribe');
```

これはコンソールで次のように実行されます:

```none
just before subscribe
got value 1
got value 2
got value 3
just after subscribe
got value 4
done
```

## プルとプッシュ

*プル* と *プッシュ* は、データ *プロデューサー* がデータ *コンシューマー* と通信する方法を記述する2つの異なるプロトコルです。

**プルとは？** プルシステムでは、コンシューマーはデータプロデューサーからデータを受信するタイミングを決定します。 プロデューサー自体は、データがコンシューマーに配信されるタイミングを認識していません。

すべての JavaScript 関数はプルシステムです。 関数はデータのプロデューサーであり、関数を呼び出すコードは、その呼び出しからの *単一の* 戻り値を "プル" して消費します。

ES2015では、別のタイプのプルシステムである [ジェネレータ関数とイテレータ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) （`関数*`）が導入されました。 `iterator.next()` を呼び出すコードはコンシューマーであり、イテレーター（プロデューサー）から *複数* の値を "引き出し" ます。


| | Producer | Consumer |
| --- | --- | --- |
| **Pull** | **Passive:** 要求されたときにデータを生成します。 | **Active:** データがいつ要求されるかを決定します。 |
| **Push** | **Active:** 独自のペースでデータを生成します。 | **Passive:** 受信したデータに反応します。 |

**プッシュとは？** プッシュシステムでは、プロデューサーがコンシューマーにデータを送信するタイミングを決定します。 消費者は、いつそのデータを受信するかを認識していません。

今日の JavaScript では、プロミスは最も一般的なタイプのプッシュシステムです。 Promise（プロデューサー）は、登録されたコールバック（コンシューマー）に解決された値を配信しますが、関数とは異なり、その値がコールバックに "プッシュ" されるタイミングを正確に決定するのは Promise です。

RxJS は、JavaScript の新しいプッシュシステムである Observables を導入しています。 Observable は複数の値のプロデューサーであり、それらをオブザーバー（コンシューマー）に "プッシュ" します。

- **Function** は呼び出し時に単一の値を同期的に返す、遅延評価された計算です。
- **generator** は遅延評価された計算であり、反復時にゼロから（潜在的に）無限に値を同期的に戻します。
- **Promise** は最終的に単一の値を返す可能性がある（またはできない可能性がある）計算です。
- **Observable** は遅延評価された計算であり、それが呼び出されたときから同期的または非同期的にゼロから（潜在的に）無限に値を返すことができます。

## 関数の一般化としてのオブザーバブル

一般的な主張とは異なり、 Observable は EventEmitters のようなものではなく、複数の値に対する Promise のようなものでもありません。 Observable は、 RxJS サブジェクトを使用してマルチキャストされる場合など、 EventEmitters のように動作する場合がありますが、通常、 EventEmitters のようには動作しません。

<span class="informal">Observable は引数なしの関数に似ていますが、複数の値を許可するように一般化します。</span>

以下を検討してください:

```ts
function foo() {
  console.log('Hello');
  return 42;
}

const x = foo.call(); // same as foo()
console.log(x);
const y = foo.call(); // same as foo()
console.log(y);
```

出力として表示されることが期待されます:

```none
"Hello"
42
"Hello"
42
```

上記と同じ動作を記述できますが、Observables を使用します:

```ts
import { Observable } from 'rxjs';

const foo = new Observable(subscriber => {
  console.log('Hello');
  subscriber.next(42);
});

foo.subscribe(x => {
  console.log(x);
});
foo.subscribe(y => {
  console.log(y);
});
```

そして出力は同じです:

```none
"Hello"
42
"Hello"
42
```

これは関数と Observable の両方が遅延計算であるために発生します。 もし関数を呼び出さないと、 `console.log('Hello')` は発生しません。 また、 Observables では（ `subscribe` を使用して）呼び出しを行わないと、 `console.log('Hello')` は発生しません。 さらに、 "呼び出し" または "サブスクライブ" は独立した操作です。 2 つの関数呼び出しは 2 つの別個の副作用をトリガーし、 2 つの Observable サブスクライブは 2 つの別個の副作用をトリガーします。 サブスクライバーの存在に関係なく副作用を共有し、熱心に実行する EventEmitter とは対照的に、 Observable は共有実行がなく、遅延します。

<span class="informal">Observable にサブスクライブすることは、関数を呼び出すことに似ています。</span>

Observable は非同期であると主張する人もいます。 それは真実ではありません。 次のように、関数呼び出しをログで囲む場合:

```js
console.log('before');
console.log(foo.call());
console.log('after');
```

次の出力が表示されます:

```none
"before"
"Hello"
42
"after"
```

そして、これは Observable での同じ動作です:

```js
console.log('before');
foo.subscribe(x => {
  console.log(x);
});
console.log('after');
```

そして出力は:

```none
"before"
"Hello"
42
"after"
```

これは、 `foo` のサブスクリプションが関数のように完全に同期していたことを証明しています。

<span class="informal">Observable は、同期的または非同期的に値を提供できます。</span>

Observable と関数の違いは何ですか？ Observable は、時間の経過とともに複数の値を "返す" ことができますが、関数ではできません。 あなたはこれを行うことができません:

```js
function foo() {
  console.log('Hello');
  return 42;
  return 100; // dead code. will never happen
}
```

関数は 1 つの値のみを返すことができます。 ただし、Observable はこれを行うことができます:

```ts
import { Observable } from 'rxjs';

const foo = new Observable(subscriber => {
  console.log('Hello');
  subscriber.next(42);
  subscriber.next(100); // "return" another value
  subscriber.next(200); // "return" yet another
});

console.log('before');
foo.subscribe(x => {
  console.log(x);
});
console.log('after');
```

同期出力あり:

```none
"before"
"Hello"
42
100
200
"after"
```

ただし、非同期で値を "返す" こともできます:

```ts
import { Observable } from 'rxjs';

const foo = new Observable(subscriber => {
  console.log('Hello');
  subscriber.next(42);
  subscriber.next(100);
  subscriber.next(200);
  setTimeout(() => {
    subscriber.next(300); // happens asynchronously
  }, 1000);
});

console.log('before');
foo.subscribe(x => {
  console.log(x);
});
console.log('after');
```

出力あり:

```none
"before"
"Hello"
42
100
200
"after"
300
```

結論:

- `func.call()` は "*同期的に 1 つの値を与える*" ことを意味します
- `observable.subscribe()` は "*同期的または非同期的に、任意の量の値を提供する*" ことを意味します

## Observable の解剖学

Observable は、新しい Observable または作成演算子を使用して **作成** され、 Observer で **サブスクライブ** され、 `next` / `error` / `complete` を Observer に配信するために **実行** され、実行は **破棄** される場合があります。 これらの 4 つの側面はすべて Observable インスタンスでエンコードされますが、これらの側面のいくつかは、Observer や Subscription などの他のタイプに関連しています。

Observable主な関心:
- **Creating** Observable を作成する
- **Subscribing** Observable の購読
- **Executing** Observable の実行
- **Disposing** Observable を破棄する

### Observable を作成する

`Observable` コンストラクターは 1 つの引数（ `subscribe` 関数）を取ります。

次の例では、 Observable を作成して、サブスクライバーに毎秒文字列 `'hi'` を送信します。

```ts
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  const id = setInterval(() => {
    subscriber.next('hi')
  }, 1000);
});
```

<span class="informal">Observable は、 `new Observable` で作成できます。 最も一般的には、 Observable は、 `of`, `from`, `interval` などの creation function を使用して作成されます。</span>

上記の例では、 `subscribe` 関数は Observable を説明する最も重要な部分です。 サブスクライブの意味を見てみましょう。

### Observable の購読

この例の Observable `observable` は次のように *サブスクライブ* できます:

```ts
observable.subscribe(x => console.log(x));
```

`new Observable(function subscribe(subscriber) {...})` での `subscribe` と `observable.subscribe` が同じ名前を持つのは偶然ではありません。 ライブラリでは、これらは異なりますが、実用的には概念的に等しいと考えることができます。

これは、同じ Observable の複数の Observer 間でサブスクライブコールが共有されないことを示しています。 オブザーバーで `observable.subscribe` を呼び出すと、 `new Observable(function subscribe(subscriber) {...})` での `subscribe` 関数が実行されます。 `observable.subscribe` を呼び出すたびに、その特定のサブスクライバー用に独自のセットアップがトリガーされます。

<span class="informal">Observable へのサブスクライブは関数の呼び出しに似ており、データが配信されるコールバックを提供します。</span>

これは、 `addEventListener` / `removeEventListener` などのイベントハンドラ API とは大きく異なります。 `observable.subscribe` では、指定された Observer は Observable のリスナーとして登録されていません。 Observable は、接続された Observer のリストも維持しません。

`subscribe` の呼び出しは、単に "監視可能な実行" を開始し、その実行の Observer に値またはイベントを配信する方法です。

### Observable の実行

`new Observable(function subscribe(subscriber) {...})` 内のコードは、 "監視可能な実行" 、つまりサブスクライブする各 Observer に対してのみ発生する遅延計算を表します。 実行により、同期的または非同期的に、時間の経過とともに複数の値が生成されます。

監視可能な実行 が提供できる値には 3 つのタイプがあります:

- "Next" notification: 数値、文字列、オブジェクトなどの値を送信します。
- "Error" notification: JavaScript エラーまたは例外を送信します。
- "Complete" notification: 値を送信しません。

"Next" 通知は最も重要で最も一般的なタイプであり、サブスクライバーに配信される実際のデータを表します。 "Error" と"Complete" の通知は、監視可能な実行中に一度だけ発生する可能性があり、どちらか 1 つしか存在できません。

これらの制約は、いわゆる *Observable Grammar* または *Contract* で最もよく表現され、正規表現として記述されます:

```none
next*(error|complete)?
```

<span class="informal">監視可能な実行では、ゼロから無限の Next 通知が配信される場合があります。 Error または Complete 通知のいずれかが配信された場合、その後は何も配信できません。</span>

以下は、 3 つの Next 通知を配信して完了する Observable の実行例です:

```ts
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
});
```

Observable は Observable 契約に厳密に従うため、次のコードは Next 通知 `4` を配信しません:

```ts
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
  subscriber.next(4); // Is not delivered because it would violate the contract
});
```

例外をキャッチした場合にエラー通知を配信するために、 `subscribe` のコードは `try / catch` ブロックでラップすることをお勧めします:

```ts
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  try {
    subscriber.next(1);
    subscriber.next(2);
    subscriber.next(3);
    subscriber.complete();
  } catch (err) {
    subscriber.error(err); // delivers an error if it caught one
  }
});
```

### 監視可能な実行の破棄

監視可能な実行は無限である可能性があり、Observer が有限時間で実行を中止したい場合が多いため、実行をキャンセルするための API が必要です。 各実行は 1 つの Observer にのみ排他的であるため、値の受信が終了すると、計算能力やメモリリソースの浪費を避けるために、 Observer は実行を停止する方法を備えている必要があります。

`observable.subscribe` が呼び出されると、 Observer は新しく作成された監視可能な実行にアタッチされます。 この呼び出しは、オブジェクトである `Subscription` も返します:

```ts
const subscription = observable.subscribe(x => console.log(x));
```

サブスクリプションは実行中の実行を表し、その実行をキャンセルできる最小限の API を備えています。 サブスクリプションタイプの詳細については、 [こちら](./guide/subscription) をご覧ください。 `subscription.unsubscribe()` を使用すると、実行中の実行をキャンセルできます:

```ts
import { from } from 'rxjs';

const observable = from([10, 20, 30]);
const subscription = observable.subscribe(x => console.log(x));
// Later:
subscription.unsubscribe();
```

<span class="informal">サブスクライブすると、実行中の実行を表す Subscription が返されます。 `unsubscribe()`を呼び出すだけで、実行をキャンセルできます。</span>

各Observableは、 `create()` を使用して Observable を作成するときに、その実行のリソースを破棄する方法を定義する必要があります。 これは、 `function subscribe()` から独自の `unsubscribe` 関数を返すことで実行できます。

たとえば、これは `setInterval` を使用して設定されたインターバル実行をクリアする方法です:

```js
const observable = new Observable(function subscribe(subscriber) {
  // Keep track of the interval resource
  const intervalId = setInterval(() => {
    subscriber.next('hi');
  }, 1000);

  // Provide a way of canceling and disposing the interval resource
  return function unsubscribe() {
    clearInterval(intervalId);
  };
});
```

`observable.subscribe` が `new Observable(function subscribe() {...})` に似ているように、 `subscribe` から返される `unsubscribe` は概念的に `subscription.unsubscribe` と同じです。 実際、これらの概念を取り巻く ReactiveX タイプを削除すると、かなり単純な JavaScript が残ります。

```js
function subscribe(subscriber) {
  const intervalId = setInterval(() => {
    subscriber.next('hi');
  }, 1000);

  return function unsubscribe() {
    clearInterval(intervalId);
  };
}

const unsubscribe = subscribe({next: (x) => console.log(x)});

// Later:
unsubscribe(); // dispose the resources
```

Observable, Observer, Subscription などの Rx タイプを使用する理由は、安全性（ Observable Contract など）とオペレーターとの構成可能性を確保するためです。

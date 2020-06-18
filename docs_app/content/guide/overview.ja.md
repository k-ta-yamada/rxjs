# Introduction

RxJSは、監視可能なシーケンスを使用して非同期でイベントベースのプログラムを作成するためのライブラリです。 これは、1つのコアタイプ、 [Observable](./guide/observable)、サテライトタイプ（Observer、Schedulers、Subjects）、および [Array#extras](https://developer.mozilla.org/en-US/docs/Web/JavaScript/New_in_JavaScript/1.6)（map、filter、reduce、everyなど）から着想を得たオペレーターを提供し、非同期イベントをコレクションとして処理できるようにします。

<span class="informal">RxJSをイベントのLodashと考えてください。</span>

ReactiveXは、 [オブザーバーパターン](https://en.wikipedia.org/wiki/Observer_pattern) と [イテレーターパターン](https://en.wikipedia.org/wiki/Iterator_pattern) 、および [コレクションを使った関数型プログラミング](http://martinfowler.com/articles/collection-pipeline/#NestedOperatorExpressions) を組み合わせて、イベントのシーケンスを管理する理想的な方法の必要性を満たします。

非同期イベント管理を解決するRxJSの重要な概念は次のとおりです:

- **Observable (観察可能):** 将来の値またはイベントの呼び出し可能なコレクションの概念を表します。
- **Observer (観察者):** Observable によって配信された値をリッスンする方法を知っているコールバックのコレクションです。
- **Subscription (購読):** Observable の実行を表し、主に実行のキャンセルに役立ちます。
- **Operators (演算子):** `map`, `filter`, `concat`, `reduce` などの操作でコレクションを処理する関数型プログラミングスタイルを可能にする純粋な関数です。
- **Subject (サブジェクト):** EventEmitter と同等であり、値またはイベントを複数のオブザーバーにマルチキャストする唯一の方法です。
- **Schedulers (スケジューラ):** 同時実行性を制御する集中型ディスパッチャであり、計算が発生したときに調整できます。 `setTimeout` または `requestAnimationFrame` など。

## 最初の例

通常、イベントリスナーを登録します。

```ts
document.addEventListener('click', () => console.log('Clicked!'));
```

RxJS を使用して、代わりにオブザーバブルを作成します。

```ts
import { fromEvent } from 'rxjs';

fromEvent(document, 'click').subscribe(() => console.log('Clicked!'));
```

### Purity (純度)

RxJS を強力にするのは、純粋な関数を使用して値を生成する機能です。 つまり、コードでエラーが発生しにくくなります。

通常、不純な関数を作成します。
この関数では、コードの他の部分が状態を台無しにする可能性があります。

```ts
let count = 0;
document.addEventListener('click', () => console.log(`Clicked ${++count} times`));
```

RxJS を使用して、状態を分離します。

```ts
import { fromEvent } from 'rxjs';
import { scan } from 'rxjs/operators';

fromEvent(document, 'click')
  .pipe(scan(count => count + 1, 0))
  .subscribe(count => console.log(`Clicked ${count} times`));
```

**scan** オペレーターは、配列の **reduce** と同じように機能します。 コールバックに公開される値を取ります。 コールバックの戻り値は、次にコールバックが実行されるときに公開される次の値になります。

### Flow (フロー)

RxJS には、オブザーバブル内のイベントの流れを制御するのに役立つさまざまなオペレーターがあります。

これは、プレーン JavaScript を使用して、1 秒あたり最大 1 回のクリックを許可する方法です:

```ts
let count = 0;
let rate = 1000;
let lastClick = Date.now() - rate;
document.addEventListener('click', () => {
  if (Date.now() - lastClick >= rate) {
    console.log(`Clicked ${++count} times`);
    lastClick = Date.now();
  }
});
```

RxJS の場合:

```ts
import { fromEvent } from 'rxjs';
import { throttleTime, scan } from 'rxjs/operators';

fromEvent(document, 'click')
  .pipe(
    throttleTime(1000),
    scan(count => count + 1, 0)
  )
  .subscribe(count => console.log(`Clicked ${count} times`));
```

その他のフロー制御オペレーターは [**filter**](../api/operators/filter), [**delay**](../api/operators/delay), [**debounceTime**](../api/operators/debounceTime), [**take**](../api/operators/take), [**takeUntil**](../api/operators/takeUntil), [**distinct**](../api/operators/distinct), [**distinctUntilChanged**](../api/operators/distinctUntilChanged) などです。

### Values (価値)

オブザーバブルを通じて渡された値を変換できます。

クリックごとに現在のマウスの x position をプレーン JavaScript で追加する方法は次のとおりです:

```ts
let count = 0;
const rate = 1000;
let lastClick = Date.now() - rate;
document.addEventListener('click', event => {
  if (Date.now() - lastClick >= rate) {
    count += event.clientX;
    console.log(count);
    lastClick = Date.now();
  }
});
```

RxJS の場合:

```ts
import { fromEvent } from 'rxjs';
import { throttleTime, map, scan } from 'rxjs/operators';

fromEvent(document, 'click')
  .pipe(
    throttleTime(1000),
    map(event => event.clientX),
    scan((count, clientX) => count + clientX, 0)
  )
  .subscribe(count => console.log(count));
```

他の価値を生み出すオペレーターは [**pluck**](../api/operators/pluck), [**pairwise**](../api/operators/pairwise), [**sample**](../api/operators/sample) などです。

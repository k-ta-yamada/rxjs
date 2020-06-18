Observable は基盤ですが、 RxJS は Operator が便利です。 Operator は、複雑な非同期コードを宣言的な方法で簡単に構成できるようにするための重要な要素です。

## What are operators?

Operator は **functions** です。 Operator には次の2種類があります:

**Pipeable Operators (パイプ可能演算子)** は、構文 `observableInstance.pipe(operator())` を使用して Observable にパイプできる種類です。 これらには [`filter(...)`](/api/operators/filter) および [`mergeMap(...)`](/api/operators/mergeMap) が含まれます。 呼び出されても、既存の Observable インスタンスは *変更* されません。 代わりに、サブスクリプションロジックが最初の Observable に基づいている *新しい* Observable を返します。

<span class="informal">パイプ可能演算子は、 Observable を入力として受け取り、別の Observable を返す関数です。 これは純粋な操作です: 以前の Observable は変更されません。</span>

Pipeable Operator は基本的に、1つの Observable を入力として受け取り、別の Observable を出力として生成する純粋な関数です。 出力 Observable をサブスクライブすると、入力 Observable もサブスクライブされます。

**Creation Operators (作成演算子)** は別の種類の Operatorで、スタンドアロン関数として呼び出して新しい Observable を作成できます。 例: `of(1, 2, 3)` は、1, 2, 3 を次々に放出するオブザーバブルを作成します。 作成演算子については、後のセクションで詳しく説明します。

たとえば、 [`map`](/api/operators/map) という Operator は、同じ名前の Array メソッドに似ています。 `[1, 2, 3].map(x => x * x)` が `[1、4、9]` を生成するのと同じように、 Observable は次のように作成されます:

```ts
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

map(x => x * x)(of(1, 2, 3)).subscribe((v) => console.log(`value: ${v}`));

// Logs:
// value: 1
// value: 4
// value: 9

```

`1`, `4`, `9` が生成されます。別の便利な Operator が [`first`](/api/operators/first) です:

```ts
import { of } from 'rxjs';
import { first } from 'rxjs/operators';

first()(of(1, 2, 3)).subscribe((v) => console.log(`value: ${v}`));

// Logs:
// value: 1
```

`map` にはマッピング関数を与える必要があるため、論理的にオンザフライで構築する必要があることに注意してください。
対照的に、 `first` は定数である可能性がありますが、それでもオンザフライで構築されます。
一般的な方法として、引数が必要かどうかに関係なく、すべての Operator が構築されます。

## Piping

パイプ可能演算子は関数なので、通常の関数のように使用 *できます* : `op()(obs)` - しかし、実際には、それらの多くが一緒に畳み込まれ、すぐに読み取り不可能になる傾向があります: `op4()(op3()(op2()(op1()(obs))))`。 そのため、 Observable には `.pipe()` と呼ばれるメソッドがあり、読みやすくしながら同じことを実行します。

```ts
obs.pipe(
  op1(),
  op2(),
  op3(),
  op4()
)
```

文体的には、 Operator が 1 つしかない場合でも、 `op()(obs)` は使用されません。 `obs.pipe(op())` が広く推奨されています。

## Creation Operators

**作成演算子とは何ですか？** パイプ可能演算子とは異なり、作成演算子は、いくつかの一般的な事前定義された動作を使用して Observable を作成するため、または他の Observable を結合することによって使用できる関数です。

作成演算子の典型的な例は、 `interval` 関数です。 入力引数として（ Observable ではなく）数値を取り、 Observable を出力として生成します:

```ts
import { interval } from 'rxjs';

const observable = interval(1000 /* number of milliseconds */);
```

[ここですべての静的作成演算子](#creation-operators-list) のリストを参照してください。

## Higher-order Observables

Observable は通常、文字列や数値などの通常の値を出力しますが、驚くべきことに、 Observable の Observable、いわゆる高次 Observable を処理する必要があります。 たとえば、表示したいファイルの URL をエミットする Observable があるとします。 コードは次のようになります:

```ts
const fileObservable = urlObservable.pipe(
   map(url => http.get(url)),
);
```

`http.get()` は、個々の URL ごとに（おそらく文字列または文字列配列の） Observable を返します。 これで、 Observable の Observable、つまりより高次の Observable ができました。

しかし、高次の Observable をどのように操作しますか？ 通常、 _フラット化_ によって:（何らかの形で）高次 Observable を通常の Observable に変換します。 例えば:

```ts
const fileObservable = urlObservable.pipe(
   map(url => http.get(url)),
   concatAll(),
);
```

[`concatAll()`](/api/operators/concatAll) オペレーターは、 "外部" の Observable から出てくる各 "内部" の Observable をサブスクライブし、その Observable が完了するまですべての発行された値をコピーして、次の Observable に進みます。 このようにして、すべての値が連結されます。 その他の便利なフラット化演算子（[*join operators* (結合演算子)](#join-operators)と呼ばれます）は

- [`mergeAll()`](/api/operators/mergeAll) — 到着した各内部 Observable をサブスクライブし、到着時に各値を発行します。
- [`switchAll()`](/api/operators/switchAll) — 最初の内部　Observable　が到着するとサブスクライブし、到着時に各値を発行しますが、次の内部　Observable　が到着すると、前の値をサブスクライブ解除し、新しい値をサブスクライブします。
- [`exhaust()`](/api/operators/exhaust) — 最初の内部 Observable が到着したときにサブスクライブし、到着時に各値を発行します。最初の内部 Observable が完了するまで、新しく到着したすべての内部 Observable を破棄し、次の内部 Observable を待ちます。

多くの配列ライブラリが [`map()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) と [`flat()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flat) （または `flatten()` ）を1つの [`flatMap()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flatMap) に結合するのと同じように、 RxJS のすべての平坦化演算子 [`concatMap()`](/api/operators/concatMap), [`mergeMap()`](/api/operators/mergeMap), [`switchMap()`](/api/operators/switchMap), そして [`exhaustMap()`](/api/operators/exhaustMap) に相当するマッピングがあります。



## Marble diagrams

Operator がどのように機能するかを説明するには、テキストによる説明では不十分な場合があります。 多くの Operator は時間に関係しており、例えば、値の排出をさまざまな方法で遅延、サンプリング、スロットル、またはデバウンスします。 多くの場合、図はそのための優れたツールです。 *マーブルダイアグラム* は、 Operator の動作を視覚的に表したもので、入力 Observable(s)、オペレーターとそのパラメーター、および出力 Observable が含まれます。

<span class="informal">マーブルダイアグラムでは、時間は右側に流れ、図は値（"マーブル"）が観測可能な実行でどのように放出されるかを示します。</span>

以下に、マーブルダイアグラムの構造を示します。

<img src="../src/assets/images/guide/marble-diagram-anatomy.svg">

このドキュメントサイト全体で、マーブルダイアグラムを広範囲に使用して、 Operator の動作を説明しています。 ホワイトボードや、単体テスト（ASCIIダイアグラムなど）の場合など、他のコンテキストでも非常に役立つ場合があります。

## Categories of operators

さまざまな目的の Operator があり、それらは、作成、変換、フィルタリング、結合、マルチキャスト、エラー処理、ユーティリティなどに分類できます。次のリストでは、すべての Operator がカテゴリに分類されています。

完全な概要については、 [リファレンスページ](/api) をご覧ください。

### <a id="creation-operators-list"></a>Creation Operators

- [`ajax`](/api/ajax/ajax)
- [`bindCallback`](/api/index/function/bindCallback)
- [`bindNodeCallback`](/api/index/function/bindNodeCallback)
- [`defer`](/api/index/function/defer)
- [`empty`](/api/index/function/empty)
- [`from`](/api/index/function/from)
- [`fromEvent`](/api/index/function/fromEvent)
- [`fromEventPattern`](/api/index/function/fromEventPattern)
- [`generate`](/api/index/function/generate)
- [`interval`](/api/index/function/interval)
- [`of`](/api/index/function/of)
- [`range`](/api/index/function/range)
- [`throwError`](/api/index/function/throwError)
- [`timer`](/api/index/function/timer)
- [`iif`](/api/index/function/iif)

### <a id="join-creation-operators"></a>Join Creation Operators
これらは、結合機能も備えた Observable 作成オペレーターです。複数のソース Observable の値を出力します。

- [`combineLatest`](/api/index/function/combineLatest)
- [`concat`](/api/index/function/concat)
- [`forkJoin`](/api/index/function/forkJoin)
- [`merge`](/api/index/function/merge)
- [`partition`](/api/index/function/partition)
- [`race`](/api/index/function/race)
- [`zip`](/api/index/function/zip)

### Transformation Operators

- [`buffer`](/api/operators/buffer)
- [`bufferCount`](/api/operators/bufferCount)
- [`bufferTime`](/api/operators/bufferTime)
- [`bufferToggle`](/api/operators/bufferToggle)
- [`bufferWhen`](/api/operators/bufferWhen)
- [`concatMap`](/api/operators/concatMap)
- [`concatMapTo`](/api/operators/concatMapTo)
- [`exhaust`](/api/operators/exhaust)
- [`exhaustMap`](/api/operators/exhaustMap)
- [`expand`](/api/operators/expand)
- [`groupBy`](/api/operators/groupBy)
- [`map`](/api/operators/map)
- [`mapTo`](/api/operators/mapTo)
- [`mergeMap`](/api/operators/mergeMap)
- [`mergeMapTo`](/api/operators/mergeMapTo)
- [`mergeScan`](/api/operators/mergeScan)
- [`pairwise`](/api/operators/pairwise)
- [`partition`](/api/operators/partition)
- [`pluck`](/api/operators/pluck)
- [`scan`](/api/operators/scan)
- [`switchMap`](/api/operators/switchMap)
- [`switchMapTo`](/api/operators/switchMapTo)
- [`window`](/api/operators/window)
- [`windowCount`](/api/operators/windowCount)
- [`windowTime`](/api/operators/windowTime)
- [`windowToggle`](/api/operators/windowToggle)
- [`windowWhen`](/api/operators/windowWhen)

### Filtering Operators

- [`audit`](/api/operators/audit)
- [`auditTime`](/api/operators/auditTime)
- [`debounce`](/api/operators/debounce)
- [`debounceTime`](/api/operators/debounceTime)
- [`distinct`](/api/operators/distinct)
- [`distinctKey`](../class/es6/Observable.js~Observable.html#instance-method-distinctKey)
- [`distinctUntilChanged`](/api/operators/distinctUntilChanged)
- [`distinctUntilKeyChanged`](/api/operators/distinctUntilKeyChanged)
- [`elementAt`](/api/operators/elementAt)
- [`filter`](/api/operators/filter)
- [`first`](/api/operators/first)
- [`ignoreElements`](/api/operators/ignoreElements)
- [`last`](/api/operators/last)
- [`sample`](/api/operators/sample)
- [`sampleTime`](/api/operators/sampleTime)
- [`single`](/api/operators/single)
- [`skip`](/api/operators/skip)
- [`skipLast`](/api/operators/skipLast)
- [`skipUntil`](/api/operators/skipUntil)
- [`skipWhile`](/api/operators/skipWhile)
- [`take`](/api/operators/take)
- [`takeLast`](/api/operators/takeLast)
- [`takeUntil`](/api/operators/takeUntil)
- [`takeWhile`](/api/operators/takeWhile)
- [`throttle`](/api/operators/throttle)
- [`throttleTime`](/api/operators/throttleTime)

### <a id="join-operators"></a>Join Operators
上記の [Join Creation Operators](#join-creation-operators) のセクションも参照してください。

- [`combineAll`](/api/operators/combineAll)
- [`concatAll`](/api/operators/concatAll)
- [`exhaust`](/api/operators/exhaust)
- [`mergeAll`](/api/operators/mergeAll)
- [`switchAll`](/api/operators/switchAll)
- [`startWith`](/api/operators/startWith)
- [`withLatestFrom`](/api/operators/withLatestFrom)

### Multicasting Operators

- [`multicast`](/api/operators/multicast)
- [`publish`](/api/operators/publish)
- [`publishBehavior`](/api/operators/publishBehavior)
- [`publishLast`](/api/operators/publishLast)
- [`publishReplay`](/api/operators/publishReplay)
- [`share`](/api/operators/share)

### Error Handling Operators

- [`catchError`](/api/operators/catchError)
- [`retry`](/api/operators/retry)
- [`retryWhen`](/api/operators/retryWhen)

### Utility Operators

- [`tap`](/api/operators/tap)
- [`delay`](/api/operators/delay)
- [`delayWhen`](/api/operators/delayWhen)
- [`dematerialize`](/api/operators/dematerialize)
- [`materialize`](/api/operators/materialize)
- [`observeOn`](/api/operators/observeOn)
- [`subscribeOn`](/api/operators/subscribeOn)
- [`timeInterval`](/api/operators/timeInterval)
- [`timestamp`](/api/operators/timestamp)
- [`timeout`](/api/operators/timeout)
- [`timeoutWith`](/api/operators/timeoutWith)
- [`toArray`](/api/operators/toArray)

### Conditional and Boolean Operators

- [`defaultIfEmpty`](/api/operators/defaultIfEmpty)
- [`every`](/api/operators/every)
- [`find`](/api/operators/find)
- [`findIndex`](/api/operators/findIndex)
- [`isEmpty`](/api/operators/isEmpty)

### Mathematical and Aggregate Operators

- [`count`](/api/operators/count)
- [`max`](/api/operators/max)
- [`min`](/api/operators/min)
- [`reduce`](/api/operators/reduce)



## Creating custom operators

### `pipe()` 関数を使用して新しい Operator を作成する

コードによく使用される演算子のシーケンスがある場合は `pipe()` 関数を使用して、シーケンスを新しい Operator に抽出します。 シーケンスがそれほど一般的ではない場合でも、シーケンスを 1 つの演算子に分割すると、読みやすさが向上します。

たとえば、次のように奇数の値を破棄して偶数の値を 2 倍にする関数を作成できます:

```ts
import { pipe } from 'rxjs';
import { filter, map } from 'rxjs/operators';

function discardOddDoubleEven() {
  return pipe(
    filter(v => ! (v % 2)),
    map(v => v + v),
  );
}
```

（ `pipe()` 関数は、 Observable の `.pipe()` メソッドに類似していますが、同じものではありません。）

### ゼロから新しい Operator を作成する

より複雑ですが、既存の演算子の組み合わせから作成できない演算子を作成する必要がある場合（まれに）、次のように、 Observable コンストラクターを使用してスクラッチで Operator を作成できます:


```ts
import { Observable } from 'rxjs';

function delay(delayInMillis) {
  return (observable) => new Observable(observer => {
    // this function will called each time this
    // Observable is subscribed to.
    const allTimerIDs = new Set();
    const subscription = observable.subscribe({
      next(value) {
        const timerID = setTimeout(() => {
          observer.next(value);
          allTimerIDs.delete(timerID);
        }, delayInMillis);
        allTimerIDs.add(timerID);
      },
      error(err) {
        observer.error(err);
      },
      complete() {
        observer.complete();
      }
    });
    // the return value is the teardown function,
    // which will be invoked when the new
    // Observable is unsubscribed from.
    return () => {
      subscription.unsubscribe();
      allTimerIDs.forEach(timerID => {
        clearTimeout(timerID);
      });
    }
  });
}
```

あなたがする必要があることに注意してください

1. 入力 Observable にサブスクライブするときに、 3 つのオブザーバー関数 `next()`, `error()`, `complete()` をすべて実装します。
2. Observable が完了したときにクリーンアップする "ティアダウン" 関数を実装します（この場合は、保留中のタイムアウトを解除してクリアします）。
3. Observable コンストラクターに渡された関数からそのティアダウン関数を返します。

もちろん、これは単なる例です。 `delay()` Operator は [すでに存在しています](/api/operators/delay)。

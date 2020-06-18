# Subject

**Subjectとは何ですか？** RxJS Subject は、値を多数のオブザーバーにマルチキャストできるようにする特別なタイプの Observable です。 プレーンな Observable はユニキャストですが（サブスクライブされた各 Observer は Observable の独立した実行を所有しています）、 Subject はマルチキャストです。

<span class="informal">Subject は Observable に似ていますが、多くの Observer にマルチキャストできます。 Subject は EventEmitter のようなものです: 多くのリスナーのレジストリを維持します。</span>

**すべての Subject は Observable です。** Subject を指定すると、 `subscribe` して、通常どおり値の受信を開始する Observer を提供できます。 Observer の観点から見ると、 Observable の実行がプレーンユニキャストの Observable から発生しているか、 Subject から発生しているかはわかりません。

Subject の内部では、 `subscribe` は値を配信する新しい実行を呼び出しません。 `addListener` が他のライブラリや言語で通常どのように機能するかと同様に、指定された Observer を Observer のリストに登録するだけです。

**すべての Subject は Observer です。** これは、メソッド `next(v)`, `error(e)`, `complete()` を持つオブジェクトです。 新しい値を Subject にフィードするには、 `next(theValue)` を呼び出すだけで、 Subject をリッスンするように登録された Observer にマルチキャストされます。

以下の例では、 Subject に 2 つの Observer がアタッチされており、 Subject にいくつかの値をフィードしています:

```ts
import { Subject } from 'rxjs';

const subject = new Subject<number>();

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

subject.next(1);
subject.next(2);

// Logs:
// observerA: 1
// observerB: 1
// observerA: 2
// observerB: 2
```

Subject は Observer であるため、以下の例のように、 Observable を `subscribe` するための引数として Subject を提供することもできます:

```ts
import { Subject, from } from 'rxjs';

const subject = new Subject<number>();

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

const observable = from([1, 2, 3]);

observable.subscribe(subject); // You can subscribe providing a Subject

// Logs:
// observerA: 1
// observerB: 1
// observerA: 2
// observerB: 2
// observerA: 3
// observerB: 3
```

上記のアプローチでは、 Subject を通じて、ユニキャストの監視可能な実行をマルチキャストに変換するだけです。 これは、 Subject が Observable の実行を複数の Observer で共有できるようにする唯一の方法であることを示しています。

Subject タイプには、 `BehaviorSubject`, `ReplaySubject`, および `AsyncSubject` といういくつかの特殊化もあります。

## Multicasted Observables

"マルチキャスト Observable" は、多くのサブスクライバーを持つ Subject を介して通知を渡しますが、プレーンな "ユニキャスト Observable" は、単一の Observer にのみ通知を送信します。

<span class="informal">マルチキャストされた Observable は、内部で Subject を使用して、複数の Observer に同じ Observable の実行を認識させます。</span>

内部では、これが `multicast` オペレーターの動作方法です: Observer は基になる Subject にサブスクライブし、 Subject はソース Observable にサブスクライブします。 次の例は、 `observable.subscribe(subject)` を使用した前の例に似ています:

```ts
import { from, Subject } from 'rxjs';
import { multicast } from 'rxjs/operators';

const source = from([1, 2, 3]);
const subject = new Subject();
const multicasted = source.pipe(multicast(subject));

// These are, under the hood, `subject.subscribe({...})`:
multicasted.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
multicasted.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

// This is, under the hood, `source.subscribe(subject)`:
multicasted.connect();
```

`multicast` は、通常の Observable のように見えますが、サブスクライブに関しては Subject のように機能する Observable を返します。 マルチキャストは、 `ConnectableObservable` を返します。これは、単に `connect()` メソッドを使用した Observable です。

`connect()` メソッドは、共有の Observable の実行がいつ開始されるかを正確に決定するために重要です。 `connect()` は内部で `source.subscribe(subject)` を実行するため、`connect()` は Subscription を返します。 Subscription をサブスクライブ解除すると、共有の Observable 実行をキャンセルできます。

### Reference counting

多くの場合、 `connect()` を手動で呼び出して Subscription を処理するのは面倒です。 通常、最初の Observer が到着したときに *自動的* に接続し、最後の Observer がサブスクライブを解除したときに共有実行を自動的にキャンセルします。

このリストで概説されているようにサブスクリプションが発生する次の例を考えてみます:

1. 最初の Observer はマルチキャストされた Observable にサブスクライブします
2. **マルチキャストされた Observable が接続されています**
3. `next` の値 `0` は最初の Observer に配信されます
4. 2番目の Observer は、マルチキャストされた Observable にサブスクライブします
5. `next` の値 `1` は最初の Observer に配信されます
6. `next` の値 `1` は 2 番目の Observer に配信されます
7. 最初の Observer はマルチキャストされた Observable からサブスクライブを解除します
8. `next` の 値 `2` は 2 番目のオブザーバーに配信されます
9. 2 番目の Observer はマルチキャストされた Observable からサブスクライブを解除します
10. **マルチキャストされた Observable への接続は登録解除されています**

`connect()` を明示的に呼び出すことでこれを実現するには、次のコードを記述します:

```ts
import { interval, Subject } from 'rxjs';
import { multicast } from 'rxjs/operators';

const source = interval(500);
const subject = new Subject();
const multicasted = source.pipe(multicast(subject));
let subscription1, subscription2, subscriptionConnect;

subscription1 = multicasted.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
// We should call `connect()` here, because the first
// subscriber to `multicasted` is interested in consuming values
subscriptionConnect = multicasted.connect();

setTimeout(() => {
  subscription2 = multicasted.subscribe({
    next: (v) => console.log(`observerB: ${v}`)
  });
}, 600);

setTimeout(() => {
  subscription1.unsubscribe();
}, 1200);

// We should unsubscribe the shared Observable execution here,
// because `multicasted` would have no more subscribers after this
setTimeout(() => {
  subscription2.unsubscribe();
  subscriptionConnect.unsubscribe(); // for the shared Observable execution
}, 2000);
```

`connect()` への明示的な呼び出しを避けたい場合は、 ConnectableObservable の `refCount()` メソッド（参照カウント）を使用できます。 このメソッドは、サブスクライバーの数を追跡する Observable を返します。 サブスクライバーの数が `0` から `1` に増えると、 `connect()` が呼び出され、共有実行が開始されます。 サブスクライバーの数が `1` から `0` に減少した場合にのみ、完全にサブスクライブ解除され、それ以上の実行は停止します。

<span class="informal">`refCount` は、最初のサブスクライバーが到着したときにマルチキャスト Observable の実行を自動的に開始し、最後のサブスクライバーが終了したときに実行を停止します。</span>

以下に例を示します:

```ts
import { interval, Subject } from 'rxjs';
import { multicast, refCount } from 'rxjs/operators';

const source = interval(500);
const subject = new Subject();
const refCounted = source.pipe(multicast(subject), refCount());
let subscription1, subscription2;

// This calls `connect()`, because
// it is the first subscriber to `refCounted`
console.log('observerA subscribed');
subscription1 = refCounted.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});

setTimeout(() => {
  console.log('observerB subscribed');
  subscription2 = refCounted.subscribe({
    next: (v) => console.log(`observerB: ${v}`)
  });
}, 600);

setTimeout(() => {
  console.log('observerA unsubscribed');
  subscription1.unsubscribe();
}, 1200);

// This is when the shared Observable execution will stop, because
// `refCounted` would have no more subscribers after this
setTimeout(() => {
  console.log('observerB unsubscribed');
  subscription2.unsubscribe();
}, 2000);

// Logs
// observerA subscribed
// observerA: 0
// observerB subscribed
// observerA: 1
// observerB: 1
// observerA unsubscribed
// observerB: 2
// observerB unsubscribed
```

`refCount()` メソッドは ConnectableObservable にのみ存在し、別の ConnectableObservable ではなく `Observable` を返します。

## BehaviorSubject

Subject の変異体の 1 つは、 "現在の値" の概念を持つ `BehaviorSubject` です。 これは、コンシューマに発行された最新の値を格納し、新しい Observer がサブスクライブするたびに、 `BehaviorSubject` から "現在の値" をすぐに受け取ります。

<span class="informal">BehaviorSubjects は、 "時間の経過に伴う値" を表すのに役立ちます。 たとえば、誕生日のイベントストリームは Subject ですが、個人の年齢のストリームは BehaviorSubject になります。</span>

次の例では、 BehaviorSubject は、最初の Observer がサブスクライブするときに受け取る値 `0` で初期化されています。 2 番目の Observer は、値 `2` が送信された後にサブスクライブしましたが、値 `2` を受信します。

```ts
import { BehaviorSubject } from 'rxjs';
const subject = new BehaviorSubject(0); // 0 is the initial value

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});

subject.next(1);
subject.next(2);

subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

subject.next(3);

// Logs
// observerA: 0
// observerA: 1
// observerA: 2
// observerB: 2
// observerA: 3
// observerB: 3
```

## ReplaySubject

`ReplaySubject` は `BehaviorSubject` に似ていますが、古い値を新しいサブスクライバーに送信できますが、 Observable 実行の一部を *記録* することもできます。

<span class="informal">`ReplaySubject` は、観察可能な実行からの複数の値を記録し、それらを新しいサブスクライバーに再生します。</span>

`ReplaySubject` を作成するときに、再生する値の数を指定できます:

```ts
import { ReplaySubject } from 'rxjs';
const subject = new ReplaySubject(3); // buffer 3 values for new subscribers

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

subject.next(5);

// Logs:
// observerA: 1
// observerA: 2
// observerA: 3
// observerA: 4
// observerB: 2
// observerB: 3
// observerB: 4
// observerA: 5
// observerB: 5
```

バッファーサイズのほかに、 *ウィンドウ時間* をミリ秒単位で指定して、記録された値の古さを判断することもできます。 次の例では、 `100` の大きなバッファサイズを使用していますが、ウィンドウタイムパラメータはわずか `500` ミリ秒です。

<!-- skip-example -->
```ts
import { ReplaySubject } from 'rxjs';
const subject = new ReplaySubject(100, 500 /* windowTime */);

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});

let i = 1;
setInterval(() => subject.next(i++), 200);

setTimeout(() => {
  subject.subscribe({
    next: (v) => console.log(`observerB: ${v}`)
  });
}, 1000);

// Logs
// observerA: 1
// observerA: 2
// observerA: 3
// observerA: 4
// observerA: 5
// observerB: 3
// observerB: 4
// observerB: 5
// observerA: 6
// observerB: 6
// ...
```

## AsyncSubject

AsyncSubject は、観察可能な実行の最後の値のみがオブザーバーに送信され、実行が完了した場合にのみ使用される変異体です。

```js
import { AsyncSubject } from 'rxjs';
const subject = new AsyncSubject();

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

subject.next(5);
subject.complete();

// Logs:
// observerA: 5
// observerB: 5
```

AsyncSubject は、単一の値を配信するために `complete` 通知を待機するという点で、 [`last()`](/api/operators/last) Operator に似ています。


## Void subject

ときどき、放出された値は、値が放出されたという事実ほど重要ではありません。

たとえば、次のコードは、 1 秒が経過したことを通知します。

```ts
const subject = new Subject<string>();
setTimeout(() => subject.next('dummy'), 1000);
```

この方法でダミー値を渡すのは扱いにくく、ユーザーを混乱させる可能性があります。

_無効なサブジェクト_ を宣言すると、値が無関係であることを通知します。 イベント自体のみが重要です。

```ts
const subject = new Subject<void>();
setTimeout(() => subject.next(), 1000);
```

コンテキストの完全な例を以下に示します:

```ts
import { Subject } from 'rxjs';

const subject = new Subject(); // Shorthand for Subject<void>

subject.subscribe({
  next: () => console.log('One second has passed')
});

setTimeout(() => subject.next(), 1000);
```

<span class="informal">バージョン 7 より前は、 Subject の値のデフォルトのタイプは `any` でした。 `Subject<any>` は放出された値の型チェックを無効にしますが、 `Subject<void>` は放出された値への偶発的なアクセスを防ぎます。 古い動作が必要な場合は、 `Subject`を `Subject<any>` に置き換えます。</span>

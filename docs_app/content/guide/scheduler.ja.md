# Scheduler

**スケジューラとは？** スケジューラは、サブスクリプションの開始時と通知の配信時を制御します。 3 つのコンポーネントで構成されています。

- **スケジューラはデータ構造です。** 優先度やその他の基準に基づいてタスクを保存してキューに入れる方法を知っています。
- **スケジューラは実行コンテキストです。** これは、タスクが実行される場所とタイミングを示します（すぐに、または setTimeout や process.nextTick などの別のコールバックメカニズム、またはアニメーションフレームなど）。
- **スケジューラーには（仮想）クロックがあります。** これは、スケジューラーのゲッターメソッド `now()` によって "時間" の概念を提供します。 特定のスケジューラーでスケジュールされているタスクは、そのクロックで示される時間のみに従います。

<span class="informal">スケジューラを使用すると、 Observable が Observer に通知を配信する実行コンテキストを定義できます。</span>

以下の例では、値 `1`, `2`, `3` を同期的に発行する通常の単純な Observable を使用し、 `observeOn` 演算子を使用して、これらの値の配信に使用する `async` スケジューラを指定します。

```ts
import { Observable, asyncScheduler } from 'rxjs';
import { observeOn } from 'rxjs/operators';

const observable = new Observable((observer) => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  observer.complete();
}).pipe(
  observeOn(asyncScheduler)
);

console.log('just before subscribe');
observable.subscribe({
  next(x) {
    console.log('got value ' + x)
  },
  error(err) {
    console.error('something wrong occurred: ' + err);
  },
  complete() {
     console.log('done');
  }
});
console.log('just after subscribe');
```

これは出力で実行されます:

```none
just before subscribe
just after subscribe
got value 1
got value 2
got value 3
done
```

通知 `got value...` が `サブスクライブ直後` に配信されたことに注意してください。 これは、これまでに見たデフォルトの動作とは異なります。 これは、 `observeOn(asyncScheduler)` が `new Observable` と最後の Observer の間にプロキシ Observer を導入するためです。 いくつかの識別子の名前を変更して、サンプルコードでその区別を明確にします:

```ts
import { Observable, asyncScheduler } from 'rxjs';
import { observeOn } from 'rxjs/operators';

var observable = new Observable((proxyObserver) => {
  proxyObserver.next(1);
  proxyObserver.next(2);
  proxyObserver.next(3);
  proxyObserver.complete();
}).pipe(
  observeOn(asyncScheduler)
);

var finalObserver = {
  next(x) {
    console.log('got value ' + x)
  },
  error(err) {
    console.error('something wrong occurred: ' + err);
  },
  complete() {
     console.log('done');
  }
};

console.log('just before subscribe');
observable.subscribe(finalObserver);
console.log('just after subscribe');
```

The `proxyObserver` is created in `observeOn(asyncScheduler)`, and its `next(val)` function is approximately the following:

```ts
const proxyObserver = {
  next(val) {
    asyncScheduler.schedule(
      (x) => finalObserver.next(x),
      0 /* delay */,
      val /* will be the x for the function above */
    );
  },

  // ...
}
```

`async` スケジューラは、指定された `delay` がゼロであっても、 `setTimeout` または `setInterval` で動作します。 いつものように、 JavaScript では、 `setTimeout(fn, 0)` は次のイベントループの反復で最も早く関数 `fn` を実行することがわかっています。 これは、 `サブスクライブが発生した直後` に、 `got value 1` が `finalObserver` に配信される理由を説明しています。

スケジューラの `schedule()` メソッドは、 `delay` 引数を受け取ります。 これは、スケジューラ自体の内部クロックを基準とした時間量を参照します。 スケジューラーのクロックは、実際の実時間とは無関係である必要はありません。 これは、 `delay` などの一時的な演算子が実際の時間ではなく、スケジューラのクロックによって指示された時間で動作する方法です。 これはテストで特に役立ちます。 実際には、スケジュールされたタスクを同期的に実行しながら、 *仮想時間スケジューラ* を使用して実時間を偽ることができます。

## Scheduler Types

`async` スケジューラーは、 RxJS が提供する組み込みスケジューラーの 1 つです。 これらはそれぞれ、 `Scheduler` オブジェクトの静的プロパティを使用して作成および返すことができます。

| Scheduler | Purpose |
| --- | --- |
| `null` | スケジューラを渡さないことにより、通知は同期的かつ再帰的に配信されます。 これは、一定時間演算または末尾再帰演算に使用します。 |
| `queueScheduler` | 現在のイベントフレームのキューでスケジュールします（トランポリンスケジューラー）。 これを反復操作に使用します。 |
| `asapScheduler` | マイクロタスクキューのスケジュール。 これは、 promise に使用されるキューと同じです。 基本的には現在のジョブの後、次のジョブの前。 これは非同期変換に使用します。 |
| `asyncScheduler` | スケジュールは `setInterval` で機能します。 これは時間ベースの操作に使用します。 |
| `animationFrameScheduler` | 次のブラウザーコンテンツが再描画される直前に発生するタスクをスケジュールします。 スムーズなブラウザアニメーションの作成に使用できます。 |


## Using Schedulers

使用するスケジューラのタイプを明示的に指定せずに、 RxJS コードですでにスケジューラを使用している場合があります。 これは、同時実行性を処理するすべての Observable 演算子にオプションのスケジューラがあるためです。 スケジューラーを指定しない場合、 RxJS は最小並行性の原則を使用してデフォルトのスケジューラーを選択します。 これは、オペレーターのニーズを満たす最小量の並行性を導入するスケジューラーが選択されることを意味します。 たとえば、メッセージが有限で少数のオブザーバブルを返すオペレーターの場合、 RxJS はスケジューラーを使用しない、つまり `null` または `undefined` です。 オペレーターが潜在的に大きいまたは無限の数のメッセージを返す場合、 `queue` スケジューラーが使用されます。 タイマーを使用するオペレーターには、 `async` が使用されます。

RxJS は最小並行性スケジューラーを使用するため、パフォーマンス目的で並行性を導入したい場合は、別のスケジューラーを選択できます。 特定のスケジューラーを指定するには、 `from([10, 20, 30], asyncScheduler)` など、スケジューラーを取るオペレーターメソッドを使用できます。

**静的作成演算子は通常、スケジューラを引数として取ります。** たとえば、 `from(array, scheduler)` では、 `array` から変換された各通知を配信するときに使用するスケジューラーを指定できます。 これは通常、演算子の最後の引数です。 次の静的作成演算子は、Scheduler引数を取ります:

- `bindCallback`
- `bindNodeCallback`
- `combineLatest`
- `concat`
- `empty`
- `from`
- `fromPromise`
- `interval`
- `merge`
- `of`
- `range`
- `throw`
- `timer`

**`subscribeOn` を使用して、 `subscribe()` コールが発生するコンテキストをスケジュールします。** デフォルトでは、Observable の `subscribe()` 呼び出しは同期してすぐに実行されます。 ただし、インスタンスオペレーター `subscribeOn(scheduler)` を使用して、特定のスケジューラーで実際のサブスクリプションが発生するように遅延またはスケジュールすることができます。 `スケジューラー` は、あなたが指定する引数です。

**`observeOn` を使用して、通知が配信されるコンテキストをスケジュールします。** 上記の例で見たように、インスタンスオペレーター `observeOn(scheduler)` は、ソース Observable と宛先 Observer の間に調停者 Observer を導入します。 調停者は、指定されたスケジューラーを使用して宛先 Observer への呼び出しをスケジュールします。

**インスタンスオペレータは、スケジューラを引数として使用できます。**

`bufferTime`, `debounceTime`, `delay`, `auditTime`, `sampleTime`, `throttleTime`, `timeInterval`, `timeout`, `timeoutWith`, `windowTime` などの時間関連の演算子はすべて、最後の引数としてスケジューラーを受け取り、それ以外の場合はデフォルトで `asyncScheduler` で動作します。

引数としてスケジューラを使用する他のインスタンスオペレーター: `cache`, `combineLatest`, `concat`, `expand`, `merge`, `publishReplay`, `startWith` 。

`cache` と `publishReplay` は両方とも ReplaySubject を利用するため、スケジューラを受け入れることに注意してください。 ReplaySubject は時間を処理する可能性があるため、 ReplaySubject のコンストラクターはオプションのスケジューラーを最後の引数として受け取りますが、これはスケジューラーのコンテキストでのみ意味があります。 デフォルトでは、 ReplaySubject は `queue` スケジューラを使用してクロックを提供します。

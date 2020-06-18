# Subscription

**サブスクリプションとは何ですか？** Subscription は、使い捨てのリソース、通常は Observable の実行を表すオブジェクトです。 Subscription には 1 つの重要なメソッドである `unsubscribe` があり、引数をとらず、サブスクリプションによって保持されているリソースを破棄するだけです。 RxJS の以前のバージョンでは、サブスクリプションは "Disposable (使い捨て)" と呼ばれていました。

```ts
import { interval } from 'rxjs';

const observable = interval(1000);
const subscription = observable.subscribe(x => console.log(x));
// Later:
// This cancels the ongoing Observable execution which
// was started by calling subscribe with an Observer.
subscription.unsubscribe();
```

<span class="informal">Subscription には、基本的に、リソースを解放したり、監視可能な実行をキャンセルしたりするための `unsubscribe()` 関数があります。</span>

 Subscription をまとめることもできるため、 1 つの Subscription の `unsubscribe()` を呼び出すと、複数の Subscription の Subscription が解除される場合があります。 これを行うには、 Subscription を別の Subscription に "adding (追加)" します:

```ts
import { interval } from 'rxjs';

const observable1 = interval(400);
const observable2 = interval(300);

const subscription = observable1.subscribe(x => console.log('first: ' + x));
const childSubscription = observable2.subscribe(x => console.log('second: ' + x));

subscription.add(childSubscription);

setTimeout(() => {
  // Unsubscribes BOTH subscription and childSubscription
  subscription.unsubscribe();
}, 1000);
```

実行すると、コンソールに次のように表示されます:
```none
second: 0
first: 0
second: 1
first: 1
second: 2
```

Subscription には、子 Subscription の追加を元に戻すための `remove(otherSubscription)` メソッドもあります。

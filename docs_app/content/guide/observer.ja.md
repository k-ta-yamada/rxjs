# Observer

**Observer とは？** Observer は、 Observable によって提供される値のコンシューマーです。 Observer は、Observable によって配信される通知のタイプごとに 1 つずつ、一連のコールバック: `next`, `error`, そして `complete` のセットです。 以下は、典型的な Observer オブジェクトの例です。

```ts
const observer = {
  next: x => console.log('Observer got a next value: ' + x),
  error: err => console.error('Observer got an error: ' + err),
  complete: () => console.log('Observer got a complete notification'),
};
```

Observer を使用するには、 Observable の `subscribe` に提供します:

```ts
observable.subscribe(observer);
```

<span class="informal">Observer は、 Observable が配信する可能性がある通知のタイプごとに 1 つずつ、 3 つのコールバックを持つオブジェクトです。</span>

RxJS の Observer も *部分的* である可能性があります。 コールバックの 1 つを指定しなくても、 Observable の実行は通常どおり行われますが、 Observer に対応するコールバックがないため、一部のタイプの通知は無視されます。

以下の例は、 `complete` コールバックのない Observer です:

```ts
const observer = {
  next: x => console.log('Observer got a next value: ' + x),
  error: err => console.error('Observer got an error: ' + err),
};
```

Observable にサブスクライブする場合、たとえば次のように、 Observer オブジェクトにアタッチせずに、コールバックを引数として提供することもできます:

```ts
observable.subscribe(x => console.log('Observer got a next value: ' + x));
```

内部的には `observable.subscribe` で、最初のコールバック引数を `next` のハンドラーとして使用して Observer オブジェクトを作成します。 3 つのタイプのコールバックすべてを引数として提供できます:

```ts
observable.subscribe(
  x => console.log('Observer got a next value: ' + x),
  err => console.error('Observer got an error: ' + err),
  () => console.log('Observer got a complete notification')
);
```

# JavaScriptのダックタイピング

## ダックタイピングとは
[ウィキペディア](https://ja.wikipedia.org/wiki/%E3%83%80%E3%83%83%E3%82%AF%E3%83%BB%E3%82%BF%E3%82%A4%E3%83%94%E3%83%B3%E3%82%B0)によりますと:
> ダック・タイピング（英: duck typing）とは、Smalltalk、Perl、Python、Rubyなどのいくつかの動的型付けオブジェクト指向プログラミング言語に特徴的な型付けの作法のことである。それらの言語ではオブジェクト（変数の値）に何ができるかはオブジェクトそのものが決定する。これによりポリモーフィズム（多態性）を実現することができる。つまり、静的型付け言語であるJavaやC#の概念で例えると、オブジェクトがあるインタフェースのすべてのメソッドを持っているならば、たとえそのクラスがそのインタフェースを宣言的に実装していなくとも、オブジェクトはそのインタフェースを実行時に実装しているとみなせる、ということである。それはまた、同じインタフェースを実装するオブジェクト同士が、それぞれがどのような継承階層を持っているのかということと無関係に、相互に交換可能であるという意味でもある。

簡単に言えば、オブジェクトに対して特定のプロパティまたはメソッドがある限り、そのオブジェクトのタイプを推定できます。

## ArrayLike 类数组对象
文字列、引数などのオブジェクトは、配列のような`length`プロパティを持ち、アクセスの添え字として数値を使用しますが、それらのオブジェクトは配列ではありません。このようなオブジェクトをArrayLikeオブジェクトと呼び、タイプは次のように表されます
```typescript
interfact ArrayLike<T> {
    [key: string]?: T,
    readonly length: number
}
```
判定方法
```typescript
const isArrayLike = array => array && typeof array.length === 'number'
```
JavaScriptのダック・タイピング特性を利用し、配列プロトタイプメソッドにcall()、 apply()を呼び出すことで配列プロトタイプメソッドがこれらのデータを処理できるようになります。
```typescript
const arrLike = {
    '0': 1,
    '1': 2,
    '2': 3,
    length: 3
}

// [].slice === Array.prototype.slice ，以下同じ
[].slice.call(arrLike) // [1, 2, 3]
[].map.call(arrLike, item => item + 1) // [2, 3, 4]
[].filter.call(arrLike, item => item !== 2) // [1, 3]
[].reduce.call(arrLike, (prev, curr) => prev + curr, 0) // 6
[].map.call('123', Number) // [1, 2, 3]
```

## Iterable 反復可能（イテラブル）なオブジェクト
オブジェクトまたはそのプロトタイプにSymbol.iteratorメソッドがある場合、反復可能（イテラブル）なオブジェクトとなります。以下のメソッドはイテレータメソッドと呼ばれます
```typescript
const iterable = {
    *[Symbol.iterator] () {
        yield 1;
        yield 2;
        yield 3;
    }
};

[...iterable] // [1, 2, 3]
```
オブジェクトはイテレータメソッドを呼び出すことにより、[スプレッド構文](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Spread_syntax)または`for...of`のイテレーションを実現できます。このオブジェクトは[反復可能プロトコル](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols)を実現し呼びこのオブジェクトは反復可能（イテラブル）なオブジェクトと呼ばれます。

ES6では、Array、String、arguments、Set、Map、FormData、およびその他のコンストラクターのプロトタイプには、それぞれ独自のSymbol.iteratorイテレータメソッドがあります。ただし、下記の`arrLike`は配列として扱うことはできますが、イテレーションはできません。これは、反復可能プロトコルを実装しておらず、イテレータメソッドを`arrLike`に追加する必要があるためです。
```typescript
arrLike[Symbol.iterator] = function* () {
    let i = 0;
    while (i < this.length) {
        yield this[i];
        i++;
    }
}

[...arrLike] // [1, 2, 3]
```
もちろん、関数によって返される反復可能（イテラブル）なオブジェクトは以下のイテレータータイプに準拠すると、通常の関数を使用してそれを実装することもできます。詳細は[反復処理プロトコル](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols)を参照してください。
```typescript
interface Iterable<T> {
    [Symbol.iterator](): Iterator<T>;
}
```
その中で、イテレーターは、ジェネレーター関数を呼び出したときの戻り値と同じnext、return、throwメソッドを提供する必要があります。
```typescript
interface Iterator<T> {
    next(value?: any): IteratorResult<T>;
    return?(value?: any): IteratorResult<T>;
    throw?(e?: any): IteratorResult<T>;
}
```
反復可能（イテラブル）なオブジェクトの判断
```typescript
const iterable = data =>
    typeof Symbol !== 'undefined'
    && typeof data[Symbol.iterator] === 'function'
```

## Thenableオブジェクト
`new Promise(()=>{})`を呼び出す時、then、catch、finally、およびその他のメソッドを含むオブジェクトが返されます、thenメソッドが含まれているオブジェクトをThenableオブジェクトと称します(TypeScriptでは`PromiseLike`タイプを指します)。
```typescript
const thenable = {
    then(res) {
        setTimeout(res, 1000)
    }
}

// 1
Promise.resolve()
    .then(()=>thenable)
    .then(()=>console.log('1秒経った'));

// 2
!async function() {
    const sleep = () => thenable

    await sleep();
    console.log('1秒経った');
}();
```
両方のステートメントは期待どおりに実行できます（1秒待ってからプリントアウト）、Promiseは、オブジェクトがthenメソッドの存在によってresolved状態になるのを待つ必要があるかどうかを判断できます。
Thenable タイピング
```typescript
interface Thenable<T> {
    then<T, N = never> (
        resolve: (value: T) => T | Thenable<T> | void,
        reject: (reason: any) => N | Thenable<N> | void
    ): Thenable<T | N>
}
```
判定方法
```typescript
const thenable = fn => fn.then && typeof fn.then === 'function'
```

## Entries
`Entries`は反復可能（イテラブル）なオブジェクトであり、反復処理プロトコルを実装する必要があります、また、プリミティブ型（文字列など）にすることはできません。例えば、オブジェクト{a: 1, b: 2, c: 3}に対して、[key, value]を要素として2次元配列を使用します
```typescript
[
    ['a', 1],
    ['b', 2],
    ['c', 3]
]
```

Entries タイピング
```typescript
interface Entries<K,V> {
    [key: number]: [K, V],
    [Symbol.iterator](): Iterator<T>;
}
```
判定方法
```typescript
const isEntries = data => {
    if (typeof data[Symbol.iterator] !== 'function') {
        return false;
    }
    return Object.values(data).every(d => Array.isArray(d) && d.length >= 2)
}
```

### Object.entries変換
Object.entriesの呼び出しでキー値ペアを持つオブジェクトをEntriesに変換できます
```typescript
const entry = Object.entries({ a: 1, b: 2, c: 3 }) // [['a', 1], ['b', 2], ['c', 3]]

const map = new Map()
map.set('a', 1)
map.set('b', 2)
map.set('c', 3)

Object.entries(map) // [['a', 1], ['b', 2], ['c', 3]]

const fd = new FormData()
fd.set('a', 1)
fd.set('b', 2)
fd.set('c', 3)

Object.entries(fd) // [['a', 1], ['b', 2], ['c', 3]]
Object.entries('abc') // [['0','a'],['1','b'],['2','c']]
```
その中で、Array、Map、Set、FormDataなどの参照型のプロトタイプには独自のentriesメソッドがあります、呼び出し後、ジェネレータオブジェクトを返します、`...`スプレッド構文で展開もできます
```typescript
const arrIterator = ['a', 'b', 'c'].entries();
[...arrIterator] // [['0','a'],['1','b'],['2','c']]

const setIterator = new Set(['a', 'b', 'c']).entries();

[...setIterator] // [['a', 'a'], ['b', 'b'], ['c', 'c']]

const map = new Map();
map.set('a', 1)
map.set('b', 2)
map.set('c', 3)

const mapIterator = map.entries();
[...mapIterator] // [['a', 1], ['b', 2], ['c', 3]]

const fd = new FormData();
fd.set('a', 1)
fd.set('b', 2)
fd.set('c', 3)

const fdIterator = fd.entries(fd);
[...fdIterator] // [['a', 1], ['b', 2], ['c', 3]]
```

### Object.fromEntriesとMapコンストラクター
Object.fromEntriesはECMAScript 2019で定義された構文です（古いバージョンのブラウザーは対応していません）、Object.entriesとは逆となり、EntriesオブジェクトをObjectに変換します
```typescript
Object.fromEntries( [['a', 1], ['b', 2], ['c', 3]] ) // { a:1, b:2, c: 3 }
```
FormDataオブジェクトまたはMapオブジェクトの場合、暗黙的にEntriesに変換されます
```typescript
const fd = new FormData()
fd.set('a', 1)
fd.set('b', 2)
fd.set('c', 3)

Object.fromEntries(fd) // [['a', 1], ['b', 2], ['c', 3]]

const map = new Map()
map.set('a', 1)
map.set('b', 2)
map.set('c', 3)

Object.fromEntries(map) // [['a', 1], ['b', 2], ['c', 3]]
```
キーのタイプが数値の場合、キーは変換後に文字列になります。Mapオブジェクトに参照タイプ（つまり、Objectが受け入れないキータイプ）がある場合、このキー値ペアは無視されます。
配列にたいしては、引数タイプの推定にはあいまいさが存在するため、EntriesまたはEntriesジェネレータオブジェクトにならないと渡すことができません。
```typescript
Object.fromEntries([1,2,3]) // エラー：要素タイプが[key, value]の形式ではありません

// [['0', 1], ['1', 2], ['2', 3]] の形式はEntriesの特性に満たしている
Object.fromEntries([1,2,3].entries()) // [['a', 1], ['b', 2], ['c', 3]]
Object.fromEntries([...[1,2,3].entries()]) // [['a', 1], ['b', 2], ['c', 3]]
```
Mapコンストラクター引数タイプについては、Object.fromEntries引数と同じタイプのEntriesオブジェクトを受け入れます
```typescript
const entries = [['a', 1], ['b', 2], ['c', 3]];

new Map(entries) // Map(3) {"a" => 1, "b" => 2, "c" => 3}

const fd = new FormData()
fd.set('a', 1)
fd.set('b', 2)
fd.set('c', 3)

new Map(fd) // Map(3) {"a" => 1, "b" => 2, "c" => 3}

new Map([1,2,3].entries()) // Map(3) {0 => 1, 1 => 2, 2 => 3}
new Map([...[1,2,3].entries()]) // Map(3) {0 => 1, 1 => 2, 2 => 3}
```
EntriesはMapにカプセル化されると、イテレーション、クエリ、要素数の取得及び削除などの操作ができます。
多くのキー値ペア形式のオブジェクトはEntriesを介して、キー値ペア間の相互変換を実現できます。

# まとめ
- ダックタイピングはオブジェクトの動作から推定されたタイプです。
- `ArrayLike`を判断するには、オブジェクトに`length`プロパティがあり、`length`の値が数値であれば判断できます。
- 反復可能（イテラブル）なオブジェクトを判断するには、オブジェクトのSymbol.iteratorプロパティの値が関数であるかどうかを確認する必要があります。
- Thenableオブジェクトを判断するには、オブジェクトのthenプロパティの値が関数であるかどうかを確認する必要があります。
- Entriesは[key、value]を要素とする2次元配列であり、Entriesの特性を利用してObject、Map、FormDataなどのデータ構造を相互変換することができます。

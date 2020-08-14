# はじめに
localStorageは、HTML5仕様でクライアントデータを永続化するためのソリューションです。LocalStorageは、データキャッシュ、ログストレージと他のアプリケーションシナリオに使用できます。 localStorageのいくつかの特性によって：
- 同一生成元ポリシーによる制限
- ストレージ容量は通常約5MB
- キー値ペアは最終的に文字列の形式で保存される

localStorageをうまく使用するのはそれほど簡単ではありません。この記事では主に、その使用に関するいくつかのベストプラクティスを紹介します。

# 互換性
ユーザーブラウザーのバージョンによって新機能に対するが異なるため、localStorageを使用する前に、スニッフィング操作で現在の環境がlocalStorageをサポートしているかどうかを判別する必要があります。
```javascript
function isLocalStorageUsable() {
  const localStorageTestKey = '__localStorage_support_test';
  const localStorageTestValue = 'test';
  let isSupport = false;
  try {
    localStorage.setItem(localStorageTestKey, localStorageTestValue);
    if (localStorage.getItem(localStorageTestKey) === localStorageTestValue) {
      isSupport = true;
    }
    localStorage.removeItem(localStorageTestKey);
    return isSupport;
  } catch(e) {
    return isSupport;
  }
}
```
読み取りおよび書き込み操作で現在のブラウザーがlocalStorage機能をサポートしているかどうかを確認できますが、一部のlocalStorageをサポートしているブラウザーは必ず書き込み操作を実行できるわけではありません。前述したように「ブラウザーによってlocalStorageに割り当てられるストレージ容量は制限されています」、保存されたコンテンツが割り当てられるストレージ容量の上限に達すると、それ以上の書き込み操作は実行できなくなります
```javascript
try {
  localStorage.setItem(localStorageTestKey, localStorageTestValue);
  if (localStorage.getItem(localStorageTestKey) === localStorageTestValue) {
    isSupport = true;
  }
  localStorage.removeItem(localStorageTestKey);
  return isSupport;
} catch(e) {
  if (e.name === 'QuotaExceededError' || e.name === 'NS_ERROR_DOM_QUOTA_REACHED') {
    console.warn('localStorage reached storage limit!')
  } else {
    console.warn('current browser does not support localStorage!');
  }
  return isSupport;
}
```

localStorage関連のメソッドを呼び出すときは、現在のブラウザがlocalStorage機能をサポートしていることを確認するべきです。ここでは、値キャッシュを使用してこのメソッドへの複数の呼び出しによるパフォーマンスの低下を回避できます。
```javascript
// クラスのインスタンスメソッド
ready() {
  if (this.isSupport === null) {
    this.isSupport = isLocalStorageUsable();
  }

  if (this.isSupport) {
    return Promise.resolve();
  }
  return Promise.reject();
}
```
上記の準備メソッドを定義することにより、スニッフィングメソッドは「遅延実行」の特性を持つようになりました。

# キー値ペア
オブジェクトがキー値ペアとしてlocalStorageに直接渡されると、toStringメソッドが暗黙的に呼び出されます。
```javascript
  // 最終的に保存されるキー値ペアは key：[object Object] value: [object Object]
  localStorage.setItem({}, {});
```
キー名のタイプに注意しないと、キー名の重複によりデータが失われる可能性があります。
localStorageのキーがオブジェクトである場合、ミスで引き起こされるバグをある程度回避するために、適切な警告を与える必要があります。
```javascript
function normalizeKey(key) {
  if (typeof key !== 'string') {
    console.warn(`${key} used as a key, but it is not a string.`);
    key = String(key);
  }
  return key;
}
```
値については、このように処理された場合、保存された値は無意味となったため、データのタイプに従ってシリアル化およびデシリアライズする必要があります。

# シリアル化
```javascript
// クラスのインスタンスメソッド
setItem(key, value) {
  key = normalizeKey(key);
  return this.ready().then(() => {
    if (value === undefined) {
      value = null;
    }
    serialize(value, (error, valueString) => {
      if (error) {
        return Promise.reject(error);
      }
      try {
        // 最大ストレージ容量を超えるため、ストレージが失敗する場合があります。
        localStorage.setItem(key, valueString);
        return Promise.resolve();
      } catch(e) {
        return Promise.reject(e);
      }
    })
  })
}
```
通常、JSON形式で保存されたデータは、JSON.stringifyを使用してシリアル化をします。
```javascript
function serialize(value, callback) {
  try {
    const valueString = JSON.stringify(value);
    callback(null, valueString);
  } catch(e) {
    callback(e);
  }
}
```
ここでは、JSON.stringifyメソッドの例外をキャプチャする必要があります、「シリアライズされたオブジェクトに循環参照がある場合、このメソッドは例外をスローします」、デシリアル化する場合、JSON.parseで例外をキャプチャすることも必要です。以下はよくあるエラーです：
```javascript
  JSON.parse('undefined');
  // VM20179:1 Uncaught SyntaxError: Unexpected token u in JSON at position 0
```
この場合、setItemメソッドでフィルタリングを実行してこのエラーを処理します。
```javascript
if (value === undefined) {
  value = null;
}
```
この簡単な判断で不正なJSON文字列を完全に回避することはできません、例外をキャッチするにはtry / catchが必要です。
ただし、要件が複雑な場合、保存された値の特定のタイプを判断して特定のシリアル化処理を実行する必要もあります。
```javascript
const toString = Object.prototype.toString;
function serialize(value, callback) {
  const valueType = toString.call(value).replace(/^\[object\s(\w+?)\]$/g, '$1');
  switch(valueType) {
    case 'Blob':
      const fileReader = new FileReader();

      fileReader.onload = function() {
        // 値のタイプをマークする
        var str =
          BLOB_TYPE_PREFIX +
          value.type +
          '~' +
          bufferToString(this.result);

        callback(null, SERIALIZED_MARKER + TYPE_BLOB + str);
      }

      fileReader.readAsArrayBuffer(value);
      break;
    default:
      try {
        const valueString = JSON.stringify(value);
        callback(null, valueString);
      } catch(e) {
        callback(e);
      }
  }
}
```
Blobタイプのストレージの対応をここに追加しました。FileReader +　ArrayBufferを使用してシリアル化する必要があり、デシリアル化プロセス中に値のタイプを通知するために識別子も要ります。

# JSON.stringifyで最適化
JSON.stringifyメソッドは実行時にオブジェクトの構造とキー値ペアのタイプを分析するため、複雑なネストされたオブジェクトを処理する時に非常に時間がかかります。
最適化の方法はこの時間のかかる作業をコンパイル段階に持ち込むことです。たとえば：
```javascript
const testObj = {
  firstName: 'Matteo',
  lastName: 'Collina',
  age: 32
}
function stringify({ firstName, lastName, age }) {
  return `"{"firstName":"${firstName}","lastName":"${lastName}","age":${age}}"`
}
```
上記の例の場合、キー値ペアとオブジェクトのタイプは実行前に決定できます。benchmark.jsからベンチマークテストデータを取得することも可能です。
```javascript
const benchmark = require('benchmark');
const fastjson = require('fast-json-stringify');
const suite = new benchmark.Suite();

const testObj = {
  firstName: 'Matteo',
  lastName: 'Collina',
  age: 32
}

function stringify({ firstName, lastName, age }) {
  return `"{"firstName":"${firstName}","lastName":"${lastName}","age":${age}}"`
}

suite.add('JSON.stringify obj', function () {
  JSON.stringify(testObj)
})

suite.add('fast-json-stringify obj', function () {
  stringify(testObj)
})

suite.on('cycle', (e) => console.log(e.target.toString()))
  .on('complete', function() {
    console.log(`Fastest is ${this.filter('fastest').map('name')}`);
  })

suite.run()
```
得られたテスト結果により、1秒間にテストコードを実行する回数と相対的に最も速い速度の統計誤差に関わらず、最適化方案のパフォーマンスが優れています。
```bash
  # テスト結果
  JSON.stringify obj x 1,695,618 ops/sec ±0.62% (90 runs sampled)
  fast-json-stringify obj x 787,253,287 ops/sec ±0.36% (92 runs sampled)
  Fastest is fast-json-stringify obj
```
上記の例は汎用性が足りなく、実際の業務開発では,カスタムJSON Schemaを利用して特定のstringifyメソッドを生成することがおすすめです、そのため選択肢としてのオープンソースフレームワークがあります：
- fast-json-stringify

# 名前空間
localStorageは同一生成元ポリシーによって制限されています、この分離レベルはアプリケーションレベルと同等ですが、実際の開発プロセスでは、この分離レベルの一部のシナリオをカバーできません。
キー値ペアをlocalStorageに保存する際には,主に以下の特徴があります：
- キー値ペアの数は、lengthプロパティで取得できます
- キー値ペアはキー名でインデックスされます
- キー値ペアは追加された時間の逆順にソートされます

では、もし現在のアプリケーションの各モジュールがlocalStorageを使用してデータを保存する必要がある場合モジュールレベルで分離するにはどうすればよいでしょうか？

キー値ペアはキー名で索引付けされますので、キー名に名前空間を付けることで区別できます。

```javascript
const keyPrefix = name + '/';
localStorage.setItem(keyPrefix + key, value);
```

複数のストレージモジュールが存在しているの場合、clearメソッドを直接呼び出すと、他のストレージモジュールのデータが誤って削除される可能性があります。名前空間を導入すると、このミスを回避できます。
```javascript
function clear(keyPrefix) {
  const keys = Object.keys(localStorage);
  keys.forEach(key => {
    if (key.indexOf(keyPrefix) === 0) {
      localStorage.removeItem(key);
    }
  })
}
```

# まとめ
おわりに、この記事で述べたベストプラクティスをまとめます。
- try / catchを使用してブラウザーの互換性をスニッフィングできますが、ストレージの制限を超えないように注意しなければならないです。
- localStorageのキーと値が文字列であるという特性に対しては、統一のシリアル化およびデシリアライズ方法がで処理すべきです。
- JSON Schemaを使用してオブジェクト構造の分析をコンパイル段階に繰り上げ、実行効率を最適化できます。
- 名前空間の導入によって複数のモジュールの管理を強化することができます

> 参照: [localForage.js](https://github.com/localForage/localForage)

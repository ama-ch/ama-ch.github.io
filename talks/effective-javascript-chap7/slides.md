# Effective JavaScript
## 第7章 並行処理
[@ama_ch](https://twitter.com/ama_ch)

2013/05/20 サイボウズ社内勉強会



### JavaScriptはスタンドアローンアプリとしてではなく、もっと大きなアプリケーションのコンテキストで実行される(例. ブラウザ)


## ブラウザでは、どんな時でも各種イベントが発生する


JavaScriptが複数の同時的なイベントに応答するためのアプローチ

* イベントキュー(イベントループ)による並行処理
* 非同期API



## イベントキュー


![event loop](img/default.svg)
https://developer.mozilla.org/en-US/docs/JavaScript/Guide/EventLoop


<iframe src="http://www.slideshare.net/slideshow/embed_code/13464412?startSlide=14" width="512" height="421" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC;border-width:1px 1px 0;margin-bottom:5px" allowfullscreen webkitallowfullscreen mozallowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="http://www.slideshare.net/shigeki_ohtsu/processnext-tick-nodejs" title="そうだったのか！ よくわかる process.nextTick() node.jsのイベントループを理解する" target="_blank">そうだったのか！ よくわかる process.nextTick() node.jsのイベントループを理解する</a> </strong> from <strong><a href="http://www.slideshare.net/shigeki_ohtsu" target="_blank">shigeki_ohtsu</a></strong> </div>



### 項目61
### イベントキューを入出力待ちでブロックしてはならない


```javascript
// ファイルをダウンロードする
var text = downloadSync('file.txt');
console.log(text);
```

* 同期API
* ダウンロードが完了するまで処理をブロックする


```javascript
// ファイルをダウンロードする(非同期バージョン)
downloadAsync('file.txt', function(text) {
  console.log(text);
});
```

* 非同期API
* ダウンロード中に他の処理が進められる
* ダウンロードが完了したらコールバック関数を呼び出す
* コールバック関数は次のイベントループで実行される


### 憶えておくべき事項

* 非同期APIはメインアプリケーションをブロックしないようにできる
* JavaScriptはイベントを同時的に受け付けるが、イベントハンドラの処理は、イベントキューを使って順次に行われる
* アプリケーションのイベントキューの中で、ブロックする入出力を使ってはならない
<aside class="notes">
コストが本当に高い処理は非同期APIを使っても処理自体はブロックしちゃうから注意<br>
普通に使ってるとブロックするAPIはほとんどないから、コスト高の処理に気をつける
</aside>



### 項目62
### 非同期シーケンスには、ネストした（または名前付きの）コールバックを使う


非同期データベースにあるURLをルックアップしてから、そのURLコンテンツをダウンロードしたい

```javascript
db.lookupAsync('url', function(url) {
  // ?
});
downloadAsync(url, function(text) {
  // urlが欲しいけど、ここでは参照できない
  console.log('contents of' + url + ': ' + text);
});
```


ネストで解決する

```javascript
db.lookupAsync('url', function(url) {
  downloadAsync(url, function(text) {
    console.log('contents of' + url + ': ' + text);
  });
});
```


シーケンスが長くなると、コールバック地獄

```javascript
db.lookupAsync('url', function(url) {
  downloadAsync(url, function(text) {
    downloadAsync('a.txt', function(a) {
      downloadAsync('b.txt', function(b) {
        downloadAsync('c.txt', function(c) {
          // ...
        });
      });
    });
  });
});
```


コールバック地獄対策: 関数に名前をつける

```javascript
// Before
db.lookupAsync('url', function(url) {
  downloadAsync(url, function(text) {
    console.log('contents of' + url + ': ' + text);
  });
});
```
```javascript
// After
db.lookupAsync('url', downloadURL);

function downloadURL(url) {
  downloadAsync(url, funcion(text) {
    showContents(url, text);
  });
}

function showContents(url, text) {
  console.log('contents of' + url + ': ' + text);
}
```

ネストは残る


コールバック地獄対策: bindでネストを減らす

```javascript
db.lookupAsync('url', downloadURL);

function downloadURL(url) {
  downloadAsync(url, showContents.bind(null, url));
}

function showContents(url, text) {
  console.log('contents of' + url + ': ' + text);
}
```

* ぱっと見改善してる？
* それぞれの処理に名前を付けるのが面倒


さっきの長い例に適用

```javascript
db.lookupAsync('url', downloadURLAndFiles);

function downloadURLAndFiles(url) {
  downloadAsync(url, downloadABC.bind(null, url));
}

function downloadABC(url, file) {
  downloadAsync('a.txt', downloadBC.bind(null, url, file));
}

function downloadBC(url, file, a) {
  downloadAsync('b.txt', downloadC.bind(null, url, file, a));
}

function downloadC(url, file, a, b) {
  downloadAsync('c.txt', finish.bind(null, url, file, a, b));
}

function finish(url, file, a, b, c) {
  // ...
}
```

関数の名前がだらしない


名前付けとネストを組み合わせる

```javascript
db.lookupAsync('url', downloadURLAndFiles);

function downloadURLAndFiles(url) {
  downloadAsync(url, downloadFiles.bind(null, url));
}

function downloadFiles(url, file) {
  downloadAsync('a.txt', function(a) {
    downloadAsync('b.txt', function(b) {
      downloadAsync('c.txt', function(c) {
        // ...
      });
    });
  });
}
```

少しスッキリしたけどまだネストが残る


最後のステップに抽象を導入

```javascript
db.lookupAsync('url', downloadURLAndFiles);

function downloadURLAndFiles(url) {
  downloadAsync(url, downloadFiles.bind(null, url));
}

function downloadFiles(url, file) {
  downloadAllAsync(['a.txt', 'b.txt', 'c.txt'], function(all) {
    var a = all[0], b = all[1], c = all[2];
    // ...
  });
}
```

* downloadAllAsyncの実装は後述
* ダウンロード処理をひとつにまとめる
* 複数ダウンロードが同時にできるようにもなる
* そもそもシーケンシャルに実行すべきではなかったというオチ


### 憶えておくべき事項

* 少数の非同期処理をシーケンシャルに実行するには、ネストされた（または名前付きの）コールバックを使う
* コールバックのネストが深すぎるのと、外に出したコールバックに「だらしない名前」をつけることのバランスを考慮しよう
* 並列処理できるものをシーケンス化するのは避けよう



### 項目63
### エラーの取りこぼしに注意


非同期処理はtryブロックで包むことができない

```javascript
try {
  setTimeout(function() {
    throw new Error();
  }, 0);
} catch(e) {
  console.log('Error!'); // 実行されない
}
```


非同期APIでエラーコールバックを受け取る

```javascript
downloadAsync('file.txt', function(text) {
  console.log(text);
}, function(error) {
  console.log('Error: ', error);
});
```


エラーコールバックの重複

```javascript
downloadAsync('a.txt', function(a) {
  downloadAsync('b.txt', function(b) {
    downloadAsync('c.txt', function(c) {
      // ...
    }, function(error) {
      console.log('Error: ', error);
    });
  }, function(error) { // 重複
    console.log('Error: ', error);
  );
}, function(error) { // 重複
  console.log('Error: ', error);
});
```


エラー処理の共通化

```javascript
function onError(error) {
  console.log('Error: ', error);
}

downloadAsync('a.txt', function(a) {
  downloadAsync('b.txt', function(b) {
    downloadAsync('c.txt', function(c) {
      // ...
    }, onError);
  }, onError);
}, onError);
```


### 憶えておくべき事項

* エラー処理コードのコピペを避けるために、共有エラー処理関数を書こう
* エラーの取りこぼしを防ぐために、すべてのエラー状況を必ず明示的に処理しよう



### 項目64
### 非同期ループには再帰関数を使おう


URLの配列を受け取って、どれかが成功するまでひとつずつダウンロードする関数がほしい

```javascript
function downloadOneAsync(urls, onsuccess, onerror) {
  for (var i = 0, len = urls.length; i < len; i++) {
    downloadAsync(urls[i], onsuccess, function(error) {
      // ?
    });
    // ループ続行
  }
  throw new Error('all downloads failed');
}
```

* これだとすべてのダウンロードを始動してしまう


ループを関数として実装する

```javascript
function downloadOneAsync(urls, onsuccess, onerror) {
  var n = urls.length;

  function tryNextURL(i) {
    if (i >= n) {
      onerror('all downloads failed');
      return;
    }
    downloadAsync(urls[i], onsuccess, function() {
      tryNextURL(i + 1);
    }
  }
  tryNextURL(0);
}
```

* tryNextURLを再帰的に実行する


再帰呼び出しはスタックオーバーフローに注意
```javascript
function countdown(n) {
  if (n === 0) {
    return;
  }
  countdown(n - 1);
}
countdown(100000); // RangeError: Maximum call stack size exceeded
```


再帰呼び出しを次のイベントループにもっていくと、コールスタックがクリアされるので大丈夫

```javascript
function countdown(n) {
  if (n === 0) {
    return;
  }
  setTimeout(function() {
    countdown(n - 1);
  }, 0);
}
countdown(100000);
```

だからさっきのtryNextURLも大丈夫！


### 憶えておくべき事項

* ループを非同期にすることはできない
* イベントループの別々の回で繰り返しを実行するために、再帰関数を使おう
* イベントループの別々の回で実行される再帰は、コールスタックのオーバーフローを起こさない



### 項目65
### イベントキューを計算でブロックさせない


* JavaScriptでアプリケーションを止めるのは簡単
```javascript
while (true) {}
```
* 応答性を保つために、イベントループの毎回の実行をできるだけ短くすることが重要
* コストの高い計算はユーザーの経験に悪影響を与える


### 本当にコストの高い計算を実行する必要があるときは？


### 1. Web Workers

http://caniuse.com/webworkers

```javascript
// ワーカを作成
var ai = new Worker('ai.js');

// ワーカにメッセージを送信
ai.postMessage(JSON.stringify({
  userMove: userMove
}));

// ワーカからの応答を処理
ai.onmessage = function(event) {
  executeMove(JSON.parse(event.data).computerMove);
};
```

* 新しいスレッドを作成して処理を実行する
* 非同期APIで結果を受け取りアプリケーションが処理する


### 2. 非同期ループに分割

ソーシャルネットワークグラフをサーチするアルゴリズム

```javascript
Member.prototype.inNetwork = function(other) {
  var visited = {};
  var worklist = [this];
  while (worklist.length > 0) {
    var memter = worklist.pop();
    // ...
    if (member === other) { // 見つかった？
      return true;
    }
    // ...
  }
  return false;
};
```

* whileループの実行に長時間かかる
* Workerを使うのは面倒


非同期な再帰関数に置き換え

```javascript
Member.prototype.inNetwork = function(other, callback) {
  var visited = {};
  var worklist = [this];
  function next() {
    if (worklist.length === 0) {
      callback(false);
      return;
    }
    var memter = worklist.pop();
    if (member === other) { // 見つかった？
      callback(true);
      return;
    }
    setTimeout(next, 0); // 次の繰り返しをスケジューリング
  }
  setTimeout(next, 0); // 最初の繰り返しをスケジューリング
};
```


ループの繰り返しを毎回違うイベントキューで実行するまでもないときは、繰り返し回数を調整する

```javascript
Member.prototype.inNetwork = function(other, callback) {
  // ...
  function next() {
    // 10回ずつまとめる
    for (var i = 0; i < 10; i++) {
      // ...
    }
    setTimeout(next, 0);
  }
  setTimeout(next, 0);
};
```


### 憶えておくべき事項

* メインのイベントキューでは、計算コストの高いアルゴリズムを避ける
* Worker APIをサポートするプラットフォームでは、ワーカを使って長い計算を別のイベントキューで実行できる
* Worker APIが利用できないか、コストが高すぎるときは、計算をイベントループの複数の回に分けて実行することを考慮しよう


* ループの非同期化はそんなに万能ではないという認識です。
* 応答可能な中途半端なUIを描画しても、あまり意味がない
* 上限を仕様として設計するべきでは
* イベントキューの合間に依存するコンポーネントの状態が変わって失敗するかも
* まったく応答がないよりはマシという程度



### 項目66
### カウンタを使って並行処理を管理する


URLの配列を受け取ってダウンロードする関数

```javascript
fundtion downloadAllAsync(urls, onsuccess, onerror) {
  var result = [], length = urls.length;

  if (length === 0) {
    setTimeout(onsuccess.bind(null, result), 0);
    return;
  }

  urls.forEach(function(url) {
    downloadAsync(url, function(text) {
      if (result) {
        // ダウンロードが完了した順に詰められるため、urlsの順でresultに入る保証がない
        result.push(text);
        if (result.length === urls.length) {
          onsuccess(result);
        }
      }
    }, function(error) {
      if (result) {
        reseult = null;
        onerror(error);
      }
    });
  });
}
```


```javascript
downloadAllAsync(['a.txt', 'b.txt', 'c.txt'], function(files) {
  // filesの中身がa,b,cの順になっている保証がない
  // ファイルサイズが c.txt < a.txt < b.txt なら、
  // filesの並びもそうなっている可能性が高いが、その保証もない
});
```


ダウンロードしたファイルを元のインデックスの位置に格納する

```javascript
fundtion downloadAllAsync(urls, onsuccess, onerror) {
  var result = [], pending = urls.length;

  if (pending === 0) {
    setTimeout(onsuccess.bind(null, result), 0);
    return;
  }

  urls.forEach(function(url, i) {
    downloadAsync(url, function(text) {
      if (result) {
        result[i] = text; // 固定インデックスで格納する
        pending --;
        if (pending === 0) {
          onsuccess(result);
        }
      }
    }, function(error) {
      if (result) {
        reseult = null;
        onerror(error);
      }
    });
  });
}
```

こうすれば、完了した結果は常に正しい順序で返される


### 憶えておくべき事項

* JavaScriptアプリケーションのイベントは、偶発的に（予想できない順序で）発生する
* 並行処理のデータ競合を避けるために、カウンタを使おう



### 項目67
### 非同期コールバックを同期的に呼び出してはいけない


ダウンロード済みのURLが渡されたらキャッシュを返す

```javascript
var cache = new Dict();

function downloadCachingAsync(url, onsuccess, onerror) {
  if (cache.has(url)) {
    onsuccess(cache.get(url)); // 同期呼び出し
    return;
  }
  return downloadAsync(url, function(file) {
    // ...
  }, onerror);
}
```


呼び出し

```javascript
downloadCachingAsync('file.txt', function(file) {
  console.log('finished');
});
console.log('starting');
```

実行結果（1度目）

```
starting
finished
```

実行結果（2度目）

```
finished
starting
```

同じ関数呼び出しなのに実行の順序が変わった！


### 憶えておくべき事項

* 非同期コールバックは（たとえデータが即座に利用できても）決して同期的に使ってはならない
* 非同期コールバックを同期的に呼び出すと、処理の期待されたシーケンスが乱され、コードの実行順序に予期しない変動が生じるかも知れない
* 非同期コールバックを同期的に呼び出すと、スタックオーバーフローや例外処理の間違いが発生するかも知れない
* 非同期コールバックを次回に実行されるようスケジューリングするには、setTimeoutのような非同期APIを使う


余談

https://code.google.com/p/closure-library/source/browse/closure/goog/async/nexttick.js

setTimeoutより良いやつ



### 項目68
### プロミスを使って非同期ロジックを明快にする


### Promise

* 非同期APIを構築する、もうひとつ人気のある方法
* Deferred, Futureとも呼ばれる
* 仕様としては大体[CommonJS Promises/A](http://wiki.commonjs.org/wiki/Promises/A)を指す
* jQueryなどのライブラリで実装されている


コールバック

```javascript
downloadAsync('file.txt', function(file) {
  console.log(file);
});
```

プロミス

```javascript
var p = downloadP('file.txt');
p.then(function(file) {
  console.log(file);
});
```


Chaining

```javascript
var p = downloadP('file.txt');
p.then(function(file) {
  return file.length;
}).then(function(length) {
  console.log('lenght: ' + length);
});
```


### join, when

複数のプロミスの結果を接合する

```javascript
var filesP = join(downloadP('file1.txt'),
                  downloadP('file2.txt'),
                  downloadP('file3.txt'));

filesP.then(function(files) {
  // ...
});
```

```javascript
var fileP1 = downloadP('file1.txt');
var fileP2 = downloadP('file2.txt');
var fileP3 = downloadP('file3.txt');

when([fileP1, fileP2, fileP3], function(files) {
  // ...
});
```


データ競合を未然に防ぐ

```javascript
var file1, file2;

downloadAsync('file1.txt', function(file) {
  file1 = file;
});

downloadAsync('file2.txt', function(file) {
  file1 = file; // 間違った変数
});
```


シーケンス全体で1つのエラーコールバック

```javascript
function onError(error) {
  console.log('Error: ', error);
}

downloadAsync('a.txt', function(a) {
  downloadAsync('b.txt', function(b) {
    downloadAsync('c.txt', function(c) {
      // ...
    }, onError);
  }, onError);
}, onError);
```

```javascript
var fileP1 = downloadP('file1.txt');
var fileP2 = downloadP('file2.txt');
var fileP3 = downloadP('file3.txt');

when(fileP1, fileP2, fileP3).then(function(files) {
  // ...
}, function(error) {
  console.log('Error: ', error);
});
```


### select
同じファイルを複数のサーバから同時にダウンロードして、最初に完了したものを使う

```javascript
var fileP = select(downloadP('http://example1.com/file.txt'),
                   downloadP('http://example2.com/file.txt'),
                   downloadP('http://example3.com/file.txt'));

fileP.then(function(file) {
  console.log(file);
});
```


selectにタイムアウトエラーを追加

```javascript
var fileP = select(downloadP('file.txt'), timeoutErrorP(2000));

fileP.then(function(file) {
  console.log(file);
}, function(error) {
  console.log('I/O error or timeout: ' + error);
});
```


### 憶えておくべき事項

* プロミスは「結果となる値」、すなわち最終的に結果を生成する平行処理を表現する
* プロミスは、さまざまな並行処理を生成するのに使える
* プロミスAPIを使えばデータ競合を防げる
* 意図的なデータ競合が必要な状況では、select（chooseとも呼ばれる）を使う



### コールバックとプロミス

[命令型のコールバック、関数型のプロミス: Node が逸した最大の機会](https://gist.github.com/okapies/5354929)


* コールバック・ベースの関数は何も返さないので、関数同士を組み立てにくい
* コールバック・ベースの関数を呼び出すと、関数を呼び出した後にそのコールバックが呼び出されるまでの間、 結果を表すものがプログラムのどこにも出てこない
* コールバックやイベント・ベースの関数から結果を得るには、基本的に”適切なタイミングで適切な場所に”いる必要がある


```javascript
fs.readFile('file1.txt',
  // しばらく時間が経つと...
  function(error, buffer) {
    // 結果が飛び出して出現する
  }
);
```


* 一方で、プロミスはタイミングや順序を気にしない
* リスナーを取り付けるのはプロミスの解決前でも後でも良く、値を取り逃すことはない
* プロミスを返す関数は、結果を表現する値を直ちに返す。この結果を表す値は第一級 (first-class) のデータとして使うことができ、他の関数へ渡すことができる。
* プロミスは、構文レベルでインデントのピラミッドを避けるためだけの方法じゃない。問題をより高いレベルでモデル化する抽象化を提供することで、もっとツールに仕事を任せられる。


```javascript
var p1 = new Promise();
// プロミスの解決前にリスナーを登録
p1.then(console.log);
p1.resolve(42);

var p2 = new Promise();
p2.resolve(2013);
// プロミスの解決後にリスナーを登録
p2.then(console.log);

// prints:
// 42
// 2013
```



### おわり

[![Effective JavaScript](img/effectivejs.jpg)](http://www.amazon.co.jp/dp/4798131113)
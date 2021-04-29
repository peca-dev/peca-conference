# .NETの非同期処理 第2回

by あれくま

---

# 前回のおさらい

.NET にはこんな非同期処理機能がありました。

* Process
  * 簡単。わかりやすい。
  * プロセス間のやりとりにはなんらかの通信が必要でめんどい。
* Thread
  * スレッド間のやりとりは簡単。
  * 一箇所のメモリに同時アクセスできるのでタイミングの問題によるバグが発生しがち。
* ThreadPool
  * スレッド数が制御しやすい。
  * スレッド間でのやりとり方法については何も解決していない。
* Task
  * 処理間のやりとりが簡単になる。
  * 継続を連鎖させるのはめんどくさい。

今回はこの続きから。

---

# イベント

前回は紹介を忘れていたのですが、Taskに似たものでイベントという機能がありました。

イベントはなんらかの通知が発生した時に実行する処理を指定しておけるだけのもので、これ自体も非同期処理を行う仕組みではありません。
GUIではボタンを押された時にイベントが発火するといった具合でよく使われますが、他に非同期処理の完了通知にも使われます。

## 利点

* 処理を渡すだけなので簡単に使える。
* データが引数で渡ってくるだけなのでメモリ読み書きのタイミング問題が発生しない。

イベント自体はC#の言語仕様に古くから組み込まれており、簡単な文法で扱うことができます。

イベントには引数が設定できるので、登録した処理の中では渡されてきた引数を見てデータを処理できます。
処理中に引数の中身が変わることは普通ないので、安心して処理することができます。

## 欠点

* 実際どこで実行されるかはわかりづらい。
* 非同期処理を待つ機能はない。

イベントの処理は通知する側が呼び出したスレッドで実行されます。
単純な動作なのでわかりやすいのですが、使う側からしてみると通知する側がどこのスレッドから通知してくるかよく知らないと思わぬことが起きたりします。
処理によってはどのスレッドで実行されてもかまわないこともあるのですが、GUIではGUIのコントロールを操作できるスレッドが限られていたりします。
またうっかり長時間かかる処理を実行すると思わぬスレッドが固まって動かなくなったりします。

イベントができるのは、通知が発生した時に登録された処理を実行するだけです。
通知が発生するまで待つ、という機能は直接はありません。
これを実現するには、イベントにフラグを立たせる処理を登録しておき、待つ方はフラグが立つのを待つということをする必要があります。
そんなに大変ではないんですが、慣れないと地味にめんどうです。

# イベント の例

イベントで非同期処理をする例を見てみましょう。

イベント自体に非同期処理を開始する機能はないので、非同期処理の結果などをイベントで通知してくれるクラスを使います。
Process にはプロセスが終了した時にイベントで通知してくれる機能があるので、それを使ってみます。

``` cs
//結果を受け取る変数を用意しておく
//結果を受け取ったら数値が入ることにしよう
int? result = null;

//ffmpeg でなんか処理したい
var proc = new Process();
proc.StartInfo.FileName = "ffmpeg";
proc.StartInfo.Arguments = "～";
//プロセス終了時にイベントが呼ばれるように設定しておく
proc.EnableRaisingEvents = true;
//プロセス終了時に呼ばれるイベントに処理を登録する
proc.Exited += (sender, args) => {
  //プロセス終了時に結果を変数に入れておくよ
  result = proc.ExitCode;
};
//プロセスを起動する
proc.Start();

//結果が設定されるまで待つ
while (!result.HasValue) {
  Console.WriteLine("処理中。ちょっと待ってね");
}

//結果を表示する
if (result.Value==0) {
  Console.WriteLine("完了しました！");
}
else {
  Console.WriteLine("なんか失敗しました！");
}
```

`Process.Exited` イベントに結果を設定する処理を登録しておき、待つ側はそれを見て待っています。

これくらいであれば簡単なんでそんなめんどくさくもないように見えますが、実際これで正しく動くかというと怪しいところがあって、ちゃんといつでも正しく動くコードを書こうとすると気をつけるところが多くあります。

# なんでも Task で待つ

Task の話に戻ります。

以前の例ではTaskを作るには `Task.Run()` を作っていました。これは渡した処理をスレッドプールで実行して、その結果を受け取れるTaskを返すメソッドです。
しかしTaskはスレッドプールでの実行を待つだけのものではありません。
単に結果が設定されるまで待つのと、設定された結果を取得するだけのものですので、なにも結果を設定するのがスレッドプールである必要はなく、もっと汎用的に使えます。

Task に結果を設定するのが TaskCompletionSource です。

``` cs
//タスクに結果を設定するTaskCompletionSourceを作っておく
//ここでは結果の型をintにしている
var taskSource = new TaskCompletionSource<int>();

//TaskCompletionSourceで設定された結果を受け取るタスクを取得する
var task = taskSource.Task;

//taskで取得できる結果を設定しておく
taskSource.SetResult(42);

//TaskCompletionSourceに結果が設定されるまで待つ
//すぐ上で既に設定されてるので実際は待たないけど
while (!task.IsCompleted) {
  Console.WriteLine("処理中。ちょっと待ってね");
}

//結果を表示する
//SetResult()に渡した値(ここでは42)が表示されるぞ
Console.WriteLine(task.Result);
```

この TaskCompletionSource は新しくスレッドを作ったりスレッドプールで何か処理したりは一切しません。
なので上の例は何の並行処理にもなっていなくて大した意味はありません。
じゃあ何に使うのかっていうと、`Task.Run()`なんかとは別の方法で非同期処理をした時の結果をTaskとして受け取りたい時に使います。

たとえばイベントの例であったプロセスの終了待ちをTaskで行ってみましょう。

``` cs
//結果を設定する TaskCompletionSource を作っておく
var taskSource = new TaskCompletionSource<int>();

//ffmpeg でなんか処理したい
var proc = new Process();
proc.StartInfo.FileName = "ffmpeg";
proc.StartInfo.Arguments = "～";
//プロセス終了時にイベントが呼ばれるように設定しておく
proc.EnableRaisingEvents = true;
proc.Exited += (sender, args) => {
  //プロセス終了時に TaskCompletionSource の結果を設定するよ
  taskSource.SetResult(proc.ExitCode);
};
//プロセスを起動する
proc.Start();

//TaskCompletionSourceで設定された結果を受け取るタスクを取得する
var task = taskSource.Task;

//TaskCompletionSourceに結果が設定されるまで待つ
while (!task.IsCompleted) {
  Console.WriteLine("処理中。ちょっと待ってね");
}

//結果を表示する
if (task.Result==0) {
  Console.WriteLine("完了しました！");
}
else {
  Console.WriteLine("なんか失敗しました！");
}
```

コードとしてはそんなに変わりませんね。結果を受け取る変数が TaskCompletionSource に変わっただけです。
しかしこれだと `Task.Wait()` でループせずに終了待ちをしたり、ただの `int?` に結果を設定するより安全で汎用性のある機能を使うことができます。

また、待つ側は非同期処理がスレッドプールで実行されてようが別プロセスで実行されてようが、中身を気にせず Task を待てばいいだけになるので、使い方が単純になります。

# Task の継続

Task には継続を指定できる機能があります。
継続というのは、終わったあとに追加でやってほしい処理のことです。

まあべつに追加でやってほしい処理なんて終わるの待ってからやってもいいんですが、それが難しかったりめんどくさかったりする場合があります。

たとえば、複数の動画ファイルがあって、それをffmpegかなんかで再エンコードして、終わったら他のフォルダに移動したいとしましょう。

``` cs
string[] files = { "hoge.flv", ...ファイル名のリスト, ... };

//全ファイルに大して実行する
foreach (string file in files) {
  //なんかffmepgでエンコードしてくれるメソッド
  //結果はTaskで待てる
  var task = EncodeWithFFMPEGAsync(file);
  //終わるのを待つ
  task.Wait();
  if (task.Result==0) {
    //正常終了してたらowataフォルダに移動する
    File.Move(file, "owata/" + file);
  }
  else {
    //失敗してたらなんか表示しよう
    Console.WriteLine($"失敗しました: {file}");
  }
}
```

まあこれでもいいんですけど、一つずつエンコードが終わるのを待っているので全然うれしくないですね。

``` cs
var tasks = new List<Task<int>>();
string[] files = { "hoge.flv", ...ファイル名のリスト, ... };

//全ファイルに対して実行する
foreach (string file in files) {
  //なんかffmepgでエンコードしてくれるメソッド
  //結果はTaskで待てる
  var task = EncodeWithFFMPEGAsync(file);

  //処理中のタスクとして取っておく
  tasks.Add(task);
}

//全部のタスクが終わったら終わるタスクを作る
var allTask = Task.WhenAll(tasks);

//全部処理が終わるまで待つ
allTask.Wait();

//allTask.Resultには待ってたタスク全部のResultの配列が入ってるよ
for (int i=0; i<allTask.Result.Length; i++) {
  //ファイル名と結果は配列に同じ順番で入ってるはず……
  var file = files[i];
  var result = allTask.Result[i];
  if (result==0) {
    //正常終了してたらowataフォルダに移動する
    File.Move(file, "owata/" + file);
  }
  else {
    //失敗してたらなんか表示しよう
    Console.WriteLine($"失敗しました: {file}");
  }
}

```

全部終わるのを待って、あらためて移動処理(または失敗表示)をしています。

まあさっきよりはマシですけど、ファイル名と結果の対応をあらためてとっているのは面倒ですね。
また、ファイルの移動はエンコードが全部終わってから一つずつ順番に行われています。
エンコードは並列で実行していますので、エンコードが終わったそばから移動してくれると、移動も並行でできて良さそうです。

ファイルのエンコードが終わったらファイルの移動をする、までを一つのタスクにしたいのでここで継続を追加します。

``` cs
var tasks = new List<Task>();
string[] files = { "hoge.flv", ...ファイル名のリスト, ... };

//全ファイルに対して実行する
foreach (string file in files) {
  //なんかffmepgでエンコードしてくれるメソッド
  //結果はTaskで待てる
  var encodeTask = EncodeWithFFMPEGAsync(file);

  //エンコードが終わったあとの処理を追加する
  //追加の処理までの結果を待てるTaskを返すよ
  var withMoveTask = encodeTask.ContinueWith(prev => {
    //prevには前のタスク(ここではencodeTaskと同じ)が入ってる
    if (prev.Result==0) {
      //正常終了してたらowataフォルダに移動する
      File.Move(file, "owata/" + file);
    }
    else {
      //失敗してたらなんか表示しよう
      Console.WriteLine($"失敗しました: {file}");
    }
  });

  //処理中のタスクとして取っておく
  tasks.Add(withMoveTask);
}

//全部処理が終わるまで待つ
Task.WaitAll(tasks);

```

`task.ContinueWith(～)` で、タスクが終わったあとの処理を追加できます。
これによって全部のエンコードが終わってからでなく、エンコードが一つ終わる度にファイルの移動をする処理を追加できました。

また、`task.ContinueWith(～)` は追加した処理までまとめて待てるTaskを返してくれます。
つまり、ファイルのエンコードをして移動する処理を待つTaskが返ってきます。
これを待つことで全部の処理を待つことができます。

しかもこれらはファイル毎に並列で実行していますので、ファイルの移動まで並列に処理してくれるようになりました。

# Task の継続の連鎖

Task の継続を使うと動画をエンコードしてファイルを移動するまでを一つなぎでできて便利なのが分かりましたが、どんどん使っていくともっと複雑な状況が出てきます。

たとえば、動画ファイルをダウンロードして、再エンコードして、終わったらファイルを移動しようとしましょう。
ダウンロードするのが挟まっただけなんでまあそんなに複雑ではなさそうです。

``` cs
var tasks = new List<Task<Task>>();
string[] files = { "http://example.com/hoge.flv", ...URLのリスト, ... };

//全ファイルに対して実行する
foreach (string file in files) {
  //なんか非同期でファイルをダウンロードしてくれるメソッド
  //結果にダウンロードしたファイル名が入ってくる
  Task<string> downloadTask = DownloadFileAsync(file);

  //ダウンロードが終わったあとの処理を追加する
  Task<Task> withEncodeAndMoveTask = downloadTask.ContinueWith(prev => {
    //なんかffmepgでエンコードしてくれるメソッド
    //prevには前のタスク(ここではdownloadTaskと同じ)が入ってる
    Task<int> encodeTask = EncodeWithFFMPEGAsync(prev.Result);

    //エンコードが終わったあとの処理を追加する
    //追加の処理までの結果を待てるTaskを返すよ
    Task withMoveTask = encodeTask.ContinueWith(prev2 => {
      //prev2には前のタスク(ここではencodeTaskと同じ)が入ってる
      if (prev2.Result==0) {
        //正常終了してたらowataフォルダに移動する
        File.Move(file, "owata/" + file);
      }
      else {
        //失敗してたらなんか表示しよう
        Console.WriteLine($"失敗しました: {file}");
      }
    });

    //ファイルのエンコードと移動を待つタスクを返す
    return withMoveTask;
  });

  //withEncodeAndMoveTask の結果にはファイルのエンコードと移動を待つタスクが入ってくる
  //つまりタスクを返すタスクになっている……？

  //処理中のタスクとして取っておく
  tasks.Add(withEncodeAndMoveTask);
}

//全部処理が終わるまで待つ
Task.WaitAll(tasks);

//ここまでで終わってる保証があるのはダウンロードだけ
//エンコードとファイルの移動の処理は開始されたのは確実だけど終わってるとは限らない
//結果の中のタスクが全部終わって初めてエンコードと移動も終わるのでそれも待たないと……
Task.WaitAll(tasks.Select(task => task.Result));

```

ダウンロード処理の `downloadTask` に、さっきのエンコードとファイル移動の処理を継続として追加しました。
それだけなんですけど激しく意味わからなくなりましたね……？

なんでいきなり難しくなったのかというと、タスクの継続で新しく非同期処理を開始したからです。
タスクが終わった時に同期処理を追加するのは簡単なんですが、非同期処理を追加すると大変なことになります。

タスクが終わった時に非同期処理を開始すると、タスクの結果として<追加された非同期処理を待つタスク>を入れることになります。
難しいので意味わからないと思うんですけど、まあ非同期処理の継続の中で非同期処理を開始したので2回待たないといけないと考えておけばそんなに間違いないです。
これが2回ならまだいいんですけど、ファイルの移動が非同期処理だったりするとタスクを返すタスクを返すタスクとかになってきて3回待つ必要が出てきます。そんな馬鹿な……。

さすがにそれは馬鹿らしいので、タスクを返すタスクを上手く処理してくれる `Unwrap()` というメソッドがあります。これはタスクを返すタスクをなんかいい感じに一つのタスクにしてくれるやつです。

``` cs
var tasks = new List<Task>();
string[] files = { "http://example.com/hoge.flv", ...URLのリスト, ... };

//全ファイルに対して実行する
foreach (string file in files) {
  //なんか非同期でファイルをダウンロードしてくれるメソッド
  //結果にダウンロードしたファイル名が入ってくる
  Task<string> downloadTask = DownloadFileAsync(file);

  //ダウンロードが終わったあとの処理を追加する
  Task<Task> withEncodeAndMoveTask = downloadTask.ContinueWith(prev => {
    //なんかffmepgでエンコードしてくれるメソッド
    //prevには前のタスク(ここではdownloadTaskと同じ)が入ってる
    Task<int> encodeTask = EncodeWithFFMPEGAsync(prev.Result);

    //エンコードが終わったあとの処理を追加する
    //追加の処理までの結果を待てるTaskを返すよ
    Task withMoveTask = encodeTask.ContinueWith(prev2 => {
      //prev2には前のタスク(ここではencodeTaskと同じ)が入ってる
      if (prev2.Result==0) {
        //正常終了してたらowataフォルダに移動する
        File.Move(file, "owata/" + file);
      }
      else {
        //失敗してたらなんか表示しよう
        Console.WriteLine($"失敗しました: {file}");
      }
    });

    //ファイルのエンコードと移動を待つタスクを返す
    return withMoveTask;
  });

  //処理中のタスクとして取っておく
  //Unwrap() を呼ぶと中のTaskが終わったら終わるTaskに変換してくれる
  tasks.Add(withEncodeAndMoveTask.Unwrap());
}

//全部処理が終わるまで待つ
Task.WaitAll(tasks);
//これでダウンロードから移動まで終わっている
```

意味わからないのが少し単純になりました。よかったね。

でも `ContinueWith()` が3つも入れ子になっててしんどいです。
今は3つで済んでますが、非同期処理をいくつも連鎖させようとするとどんどん深くなっていくし `Unwrap()` も連打するはめになります。やべえ。

それでは使いづらいので `async/await` という機能が追加されました。

が、それはまたの機会に……。


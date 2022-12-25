---
title: "Dartで作るコマンドラインツール"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart"]
published: true
---

この記事は[株式会社TORICO Advent Calendar 2022](https://qiita.com/advent-calendar/2022/torico) の25日目の記事です。


最近社内で作った十数個のEPUB から特定の文字を変更することや、あるデータで使っているYaml の値を280箇所書き換えるなどのタスクがありました。
この場合に、正規表現で一括書き換えができるものはVSCode だけで完結することもありますが、ファイルが多岐に渡ったり、一括で置換できないものはスクリプトやCLIツールを作成して対応することがあります。

その際に最近はDart でツールを作ることが多いので、そのツールの作り方について解説したいと思います。
Dart を使っている理由としては書くことに慣れていて、調べなくてもある程度書き進めることができるのと、`dart compile exec` でシングルバイナリの実行ファイルにコンパイルできることが理由です。


## 1. プロジェクトの作成

プロジェクトの作成は`dart create` でできます。
ディレクトリを作ってからプロジェクトを作成する場合は`--force`オプションを付けてください。

```
dart create .
```

プロジェクトを作成すると下記の雛形が作成されます。
Flutterプロジェクトの雛形との違いといえば、bin ディレクトリがあることと、main.dart が無いことくらいで、Flutter で開発している方には馴染みのある構成になっていると思います。

```
.
├── analysis_options.yaml
├── bin
│  └── torico_advent_calendart_2022_2.dart
├── CHANGELOG.md
├── lib
│  └── torico_advent_calendart_2022_2.dart
├── pubspec.lock
├── pubspec.yaml
├── README.md
└── test
   └── torico_advent_calendart_2022_2_test.dart
```

プログラムを実行する場合は、`dart run` で実行することができます。

```
$ dart run bin/torico_advent_calendart_2022_2.dart
Hello world: 42!
```


## 2. フラグや引数を使えるようにする

`dart create --force .` のようなコマンドがあった場合に、`--force` がフラグです。

`dart creaet` をした後の /bin ディレクトリにあるファイルを見ると下記のようになっており、`arguments` からフラグや引数が取得できます。

```dart
void main(List<String> arguments) {
  print('Hello world: ${torico_advent_calendart_2022_2.calculate()}!');
}
```

arguments の値を確認するために下記のように書き換えます。

```dart
void main(List<String> arguments) {
  // argumentsの値を全て表示する
  print(arguments);
  print('Hello world: ${torico_advent_calendart_2022_2.calculate()}!');
}
```


試しに下記のコマンドを実行してarguments の値を確認してみると、入力したものが、StringListで前から順番に入っていることが確認できます。
```
$ dart run bin/torico_advent_calendart_2022_2.dart -h -v -t lib/main.dart
[-h, -v, -t, lib/main.dart]
```

ただこれでは扱いづらいので[args](https://github.com/dart-lang/args)ライブラリーを追加して、扱いやすいようにします。
上記のコマンドで書いている`-h`(ヘルプ), `-v`(バージョン), `-t`(ターゲット)をargs を使って引数を受け取れるようにしてみます。

### 2-1. args のインストール

引数の解析には`args`ライブラリーを使用します。
他にもあるのかもしれませんが、dart 公式のライブラリーとなっているので、これを採用しておけば良いと思っています。

```
dart pub add args
```

https://github.com/dart-lang/args

### 2-2. フラグ、引数を受け取れるようにする

```dart
import 'package:args/args.dart';

void main(List<String> arguments) {
  final parser = ArgParser();
  // --help, -h のオプションを登録
  parser.addFlag('help', abbr: 'h', help: 'Show usage information');
  // --version, -v のオプションを登録
  parser.addFlag('version', abbr: 'v', help: 'Show version number');
  // --target, -t の引数を登録
  parser.addOption('target', abbr: 't');
  // パーサーにコマンドライン引数を渡す
  final result = parser.parse(arguments);
  
  // result がMap になっているので、キーを指定して値を取得する
  print('help: ${result['help']}');
  print('version: ${result['version']}');
  print('target: ${result['target']}');
}
```

単機能のツールを作る分にはサブコマンドは必要ないと思いますが、args には[デフォルトのヘルプ コマンド](https://github.com/dart-lang/args#default-help-command)の追加や、Flutter のコマンドで言う`create` に当たる[サブコマンド](https://github.com/dart-lang/args#subcommands) と言われるものを追加する機能があります。

### 2-3. ヘルプオプションを実装する

例として簡易的なヘルプオプションを実装します。
処理の内容としては、help オプションがコマンドに追加されている場合に、`parser.addFlag`、`parser.addOption`で登録した各オプションと引数のヘルプテキスト表示します。
`parser.usage` を使うとオプションと引数のヘルプテキストが表示されます。
またコマンドラインツールでは`return` して終了するのではなく、`exit()`に[終了コード](https://dart.dev/tutorials/server/cmdline#setting-exit-codes)を渡して、プログラムを終了させます。

0が成功で、1〜255が失敗の終了コードになっています。
今回は、`help`がtrue だった場合は、ヘルプテキストを表示してプログラムを終了させるので、下記のコードになります。

```dart
import 'package:args/args.dart';

void main(List<String> arguments) {
  final parser = ArgParser();
  parser.addFlag(
    'help',
    abbr: 'h',
    help: 'Show usage information',
  );
  parser.addFlag(
    'version',
    abbr: 'v',
    help: 'Show version number',
  );
  parser.addOption(
    'target',
    abbr: 't',
  );
  final result = parser.parse(arguments);
  final help = result['help'] as bool;
  if (help) {
    // ヘルプテキストを表示する
    print(parser.usage);
    // 終了コード 0 でプログラムを終了させる
    exit(0);
  }
  print('version: ${result['version']}');
  print('target: ${result['target']}');
  print(parser.usage);
  exit(0);
}
```

実行結果は以下の通りです。

```
$ dart run bin/torico_advent_calendart_2022_2.dart -h
-h, --[no-]help       Show usage information
-v, --[no-]version    Show version number
-t, --target
```


## あとがき

Dart といえばFlutter のような感覚の方は多いかもしれませんが、コマンドラインツールやサーバーアプリケーションを作ることも可能なので、ぜひFlutter モバイルアプリエンジニアの方はちょっとしたツールを作る練習をしておいて、いつか来るであろう面倒くさく時間のかかる仕事をプログラムの力を使ってサクッと解決できるように準備しておいてはいかがでしょうか？？
Dart は静的型付け言語でありつつも、意外とゆるく使うことができるのでサクッと作る分にはおすすめの言語です！！

他の書き方やサンプルとなるコードを見たい方は[melos](https://github.com/invertase/melos/tree/main/packages/melos) や[mason_cli](https://github.com/felangel/mason/tree/master/packages/mason_cli)を見てみると参考になると思います。

今回書いたプログラムは下記のリポジトリで公開しています。
https://github.com/0maru/torico-advent-calendart-2022-2
---
title: "学祭ステージ用にNixとRustでアーティファクトを作った話"
emoji: "❄️"
type: "tech"
topics: [ "rust", "nix", "bevy" ]
published: true
published_at: 2025-12-15
---

# はじめに

みなさんこんにちは. 最近コミュニティイベントにもちょくちょく顔出すようになったKlehaです. 
2025年ももうすぐ終わりますが, 今年みなさんはなにか面白いことができたでしょうか. 
私はなんか新年早々わけわからない書類を作っていましたが, 何だかんだで忙しすぎてそちらの話は全然進められてません. 
今回は, そんな私が今まで全然そちらの話を進められなかった原因のお話です. 
いつもほど長くないので適当に読み流してもらえると幸いです. ~~(いつもが長すぎるだけ~~

# 今回の主題

今回は, 私が高専祭(学祭)のステージで使うために, Bevy, Axum, Nixを用いて電光掲示板をフルスクラッチで作ったという話です. 
その名も「nex-board」, 先進的な技術スタックに掛けて「next」, 5枚のモニターを並べて制御することに掛けて「nexus」, そして今回のコア技術である「nix」, この3つを掛けて命名しました. 
これはMIT Licenseで公開してるのでどうぞご自由にお使いください.

https://github.com/nex-board/nex-board

# 技術スタック

nex-boardはRustで実装しています. 具体的には[Bevy](https://bevy.org/)というゲームエンジンと[axum](https://docs.rs/axum/latest/axum/)というWebアプリケーションフレームワーク, [Serde](https://serde.rs/)というパーサーを使用しています. 

## Bevy

Bevyは, **ECS**に準拠したGUIエディタを持たないゲームエンジンです. 

ECSは, ゲームを構成する要素を**Entity**, **Component**, **System**に分けて管理するアーキテクチャです. 
従来のオブジェクト指向プログラミングでいうClassは, インスタンスという実体があって, その中でメソッドが定義されていました. 
一方で, ECSでは, 実体はEntity, そしてEntityはComponent(データ)をもち, それを制御するのはSystem, という感じになります. 

簡単な例で確認してみましょうか. 

```csharp
class Player {
  Vector3 position, velocity;
  void Update() {
    position += velocity * Time.deltaTime;
  }
}

class GameManager {
  List<Player> players = new List<Player>();
  List<Train> trains = new List<Train>();
  void Setup() {
      var p = new Player { pos = new(0,0), vel = new(1,0) };
    players.Add(p);
  }
  void Update() {
    foreach (var p in players) p.Updatete();
  }
}
```

↑ OOP (Unity)
↓ ECS (Bevy Engine)

```rust
#[derive(Component)]
struct Position { x: f32, y: f32 }

#[derive(Component)]
struct Velocity { x: f32, y: f32 }

#[derive(Component)]
struct Player;

fn move_system(mut query: Query<(&mut Position, &Velocity)>) {
  for (mut pos, vel) in &mut query {
    pos.x += vel.x
    pos.y += vel.y
  }
}

fn setup(mut cmds: Commands) {
  cmds.spawn((Player, Position { x = 0.0, y = 0.0 }))
}
```

多くの人が見慣れてるのはOOPのほうの書き方だと思います. 
この程度のコードだとOOPのほうが見やすそうですよね. 

では, 例えば電車も一緒に動かしたい, というとどうでしょうか. 

```diff csharp
class Player {
    Vector3 pos, vel;
    public void Update() { pos += vel; }
}

+ class Train {
+     Vector3 pos, vel;
+     public void Update() { pos += vel; }
+ }

class GameManager {
    List<Player> players = new List<Player>();
+   List<Train> trains = new List<Train>();
	
    void Setup() {
        var p = new Player { pos = new(0,0), vel = new(1,0) };
        players.Add(p);

+       var t = new Train  { pos = new(0,5), vel = new(2,0) };
+       trains.Add(t);
    }

    void Update() {
        foreach (var p in players) p.Update();
+       foreach (var t in trains)  t.Update();
    }
}
```

↑ OOP
↓ ECS

```rust diff
#[derive(Component)]
struct Position { x: f32, y: f32 }

#[derive(Component)]
struct Velocity { x: f32, y: f32 }

#[derive(Component)]
struct Player;

+#[derive(Component)]
+struct Train;

fn move_system(mut query: Query<(&mut Position, &Velocity)>) {
  for (mut pos, vel) in &mut query {
    pos.x += vel.x
    pos.y += vel.y
  }
}

fn setup(mut cmds: Commands) {
  cmds.spawn((Player, Position { x = 0.0, y = 0.0 }))
+ cmds.spawn((Train, Position { x = -100.0, y = -250.0}))
}
```

OPPでは, 別のモノを扱う場合は別のClassを書くか, 継承するかでしたが, ECSではとにかく「このComponentが設定されてたらEntityが何だろうと動かす」なので(というかEntityの種類そのものがComponentなので),  今回の例ではOOPではTrainクラスの実装が必要でしたがECSでは識別用のComponentとspawnまわりを書き換えるだけで済みます. 
これは, ECSでは, QueryでEntityの中からComoponentを使って絞りこんで, それに対してSystemで処理をするという形で, 今回の場合はその条件が「PositionとVelocityをもつ」ことだったわけです. なので, PlayerとTrainの双方にPositionとVelocityを付せば, 同じロジックを2度書くことなく同じ挙動をさせることができるのです. 

...とまあBevyの話をすこししすぎてしまいましたね. また今度Bevyの記事も書きます. 

残り2つはさらっと流していきます. 本題はNixなのでね. 

## Axum

Axumは, Actix webと並んでRustのWebフレームワークの代表のような立場のフレームワークです. 
Tokioという非同期ランタイムがあり, そのチームが開発したものです. (Actix webもtokioベースですが, あちらは独自のアーキテクチャを用いて高速化を図っているようです)

今回は時間があまりにもなかったので, Axumのコードはほぼ生成AIに書かせました.

## Serde

Serdeは, Rustのcrateでおそらく一位二位を争うほどに有名なcrateで, シリアライズ・デシリアライズライブラリです. 
serde-tomlやcsv, serde-jsonなどと併用することで実に様々なファイルをパースすることができます. 
ちなみに由来は"**ser**ializing and **de**serializing"なんだとか

## Nix

...は説明しなくてもいいよね？ これ**Nix** Advent Calendarだし...

とうのはさておき, Nixも一応説明しておきましょう. 

Nixは, その異常なまでの厳密さが特徴のビルドシステムです. 
どのくらい厳密なのかというと, ビルドするときは, とてつもなく厳密なビルド式(Derivation)を用いて, 入力(ビルド対象ファイルなど)のハッシュを確認して, ネットワークを遮断してderivationに明記されているパッケージ以外入っていないsandboxでderivationに則ってビルドをします. もし入力のハッシュが合わなければ, 即座にビルドは失敗します. 

これによって, Nixはとてつもなく高い再現性をもちます. 原則として, 手許でビルドが通れば, 別マシンだろうとリモート環境だろうと仮想環境だろうとCIだろうとビルドが通ることになります. 

このderivationは, Nix Expression Language, 通称Nix言語を用いて記述します. 純粋関数型言語です. 
derivationは`stdenv.mkDerivation`を用いて定義するのが一般的ですが, Rustの場合`rustPlatform.buildRustPackage`というのがあったり, 他にもいろいろと便利なラッパー関数があります. 

また, Nixはパッケージマネージャーとしての側面ももちます, 
そして, Nix Package Managerでは(というかNixも), すべてのパッケージは`/nix/store/`というディレクトリに隔離され, Nix Package Managerを用いる場合はここから必要な箇所にシンボリックリンクを貼る形となります. 
つまり, GlobalなPATH環境を汚すのではなく, 可逆的な変更を加えるのです. 

またdevShellという開発環境の構築機能もありますが, まあ簡単にいうと一時的に環境変数を書き換えて指定されたパッケージのPATHが通った環境が使える, そういうものです. 

今回はNixを純粋にビルドシステムとして使うとともに, devShellも使用しています. 

# Bevyのビルド

みなさんも薄々気づかれてるかと思いますが, 私は先程「Nixは指定したパッケージ以外存在しないsandboxでビルドを行う」といいました. 
そんな環境でゲームのビルドなんて......考えたくもないですよね. 

実際, 今回は音声は使用していないのでまだましかもしれませんが, BevyはWaylandやMesaに強く依存し, 他にも様々なものに対する依存を持っています. 
しかも, 当然ながらBevyはゲームエンジンであるがゆえに多数のcrateとの依存関係があります. 
そのcrateたちにもまた依存パッケージがあります. 
......と, まあよくある依存地獄に陥るわけですよ. 

ただ, こういうものこそNixでビルドすべきだとも思います. 
PC A,B,Cの3台に展開すると考えたとき, PC AとBではうまくできるのにCではなんかうまくいかない, ということがよくあります. もちろん大抵原因は依存関係です. 
Nixは, こうなるリスクをゼロにできるわけですね. なにせ何もない状態から依存パッケージを全部用意してビルドするので. 

また, nix runなどを使えば, 実行時依存も注入された状態でビルド成果物を動かすことができます. Nix Package Managerでインストールしても同様です. 

今回デプロイに使用したOSはNixOSです. これはその名の通り, Nixに最適化されたとてつもなく再現性の高いLinuxディストロです. 私の手許環境もNixOSです. 
NixOSでは, システム設定もConfiguration.nixというファイルにNix式で定義していきます.
参考までに私のdotfile貼っておくので気になる人はご覧ください. 

https://github.com/kleha-tc/dotfiles

ただ, このNixOS, 致命的な問題が1つありまして, FHSに準拠してないんですよね......
FHSというのは, Filesystem Hierarchy Standardといって, Linux環境の標準的なディレクトリ構成のことです. 
これに準拠していないということはすなわち, たとえばプログラム側がbashを`/usr/bin/bash`にあることを前提に組まれていたらPath Not Foundで動かないということです. 
そして, Wayland環境なんて絶対こういう依存多いに決まってますよね. 

そういうときに取れる方法は2つあり, 

1. nix-ld
2. pkgs.buildFHSEnv

となります. 

nix-ldは, NixOS環境下において, FHSに準拠するような形でシンボリックリンクを貼ってくれるツールです. 
buildFHSEnvは, Nixのビルド時やdevShell環境などで, 軽量なFHS準拠のコンテナを用意して, そこの中でビルドしたりdevShellでつないだりできるツールです. 

今回は後者でビルドを行いました. 

とりあえずflake.nixを貼っておきます. 

https://github.com/nex-board/nex-board/blob/main/flake.nix

flake.nixを見たらわかるとおり, 結構色々と試行錯誤しています. その結果として「どう考えてもletの中に放り込んだ方がいいだろ!」というものもそのままになっています. そのうち直します, というかこのシステム自体今後も拡張していくので早い段階で直すと思います. 

また, 書いてある通り, Rustの部分はfenixを用いています. このあたりも今後非Nix環境でもビルド可能にするための布石です. 
というのも, fenixではrust-toolchain.tomlが使用できます. 年末年始あたり時間があればrust-toolchain.tomlを追加してfenixにも反映させようと思っています. 

# 背景

そもそもなんでこんな尖った技術を採用したのか, その裏側の話を少しだけ......

学祭の3か月ほど前に, 3台のPCがほぼ同時期に壊れるというトラブルが発生して, その結果電光掲示板に使えるのは10年近く前のCore i3搭載PC3台, しかもストレージはHDD, メモリも4GBだったんですよ.
今まではCore i5-12500やi3-12100とかにメモリ16GB, SSD環境でNext.jsベースのシステムで運用してたんですが, それでも尚カクつくなーとか言ってました. 
そんなシステムがこの限界スペックのPCで動くわけがない, それに薄々気づきながらもシステムの作りなおしはやりたくないのでどうにか直せないかと模索していました. 
結局修理は不可能と結論づけられたのが高専祭1か月前, とりあえず前のシステムも最低限カクカクでも動くだろう, そう信じて動かそうとしたんです. 
ですが, そもそもWindows10のサポート切れ問題から, Windowsでの動作が倫理上厳しくなり, 管理性と再現性の高さからNixOSで運用しようとしました. 
結果としては......DisplayLinkで無理やり拡張してどうにか3画面出力でやってたというのが旧システムですが, DisplayLinkがうまく動作せず動かないと結論づけられました. これが高専祭3週間前です. 
そして, もうここまでくるとシステム再構築以外の手段がいよいよなくなってきたので, 3週間でこのシステムを作りあげた, というわけです. 

もちろん些細なトラブルはたくさんありましたし, その結果反省会資料は脅威の13ページ. ただ, 目立った放送事故は1件で済みましたし, それもシステムではなく流す文字列データの誤りでした. 

ちなみに, このシステムの全体像としては, モニター5枚を並べて左から1..5としますと, 1..2→PC-A, 3→PC-B, 4..5→PC-Cに接続し, 3..5は1枚の大きなモニターとして同期運用を行うものです. 流すのは文字列のみなので比較的ましです. 
同期の手法としては, オペレーター側がWebSocket同時接続で2台両方に同時に信号を送るという手法をとりました. 

ただし, このとき
PC-A: Core i3-6100/4GB RAM/HDD
PC-B: Core i3-12100/16GB RAM/SSD 
PC-C: Core i3-7100/8GB RAM/HDD
という構成でした. 要するに, モニター3と4..5の間でたまにズレが生じてしまったんですよね. 
まあ当然といえば当然です. PC-CはPC-Bよりもそもそもの性能がかなり低いですし, I/Oのボトルネックもかなり大きいので. 

この環境下において, もはや私に残された選択肢などないに等しかったわけですが, 唯一希望が見いだせたのがこのBevy×Nixのアプローチです. 
このアプローチでは, パフォーマンスはかなりいいうえに, Nixの再現性の高さとRustの堅牢性によりデプロイまわりでのトラブルを最小限に抑えられます.

その結果がこの"nex-board"です. 

もちろんまだまだ荒削りなのでこの先様々な機能を載せたりAIに書かせた部分をリファクタしたりデフォルト操作UIをElmで作ったりして実用に耐えうるようにしていきますが, まあまず高専祭ステージイベントでの実運用に耐えた記念としてv0.1.0をリリースしておきました. 

もし興味があれば色々いじってみてください!

# おわりに

私は, この開発を通して, BevyやNixのまわりはもちろん, その他いろいろなことを学びました. 
特にBevyの情報については日本語情報はまだまだ少ないので積極的に発信していきたいと思います. 

> In the middle of every difficulty lies opportunity.  - Albert Einstein

アインシュタインも, 困難の中にこそ機会がある, といっています. 私にとっては, 今回はいつか触ってみようと思っていたBevyを半強制的ではあったものの学ぶことができ, 結果としてはよかったのかなと思っています. この先, 様々な困難があるとは思いますが, この経験を胸に立ち向かっていきたいと思います. 

~~まあルロイ先生は「困難は分割せよ」と言ってましたが......今回は分割することなく1人で片付けてしまいました. ルロイ先生に右ひとさし指を立てられそう......~~

例によって結構長い記事でしたが最後まで読んでいただきありがとうございました. 
今後続編として, nex-boardの開発と並行して, nex-boardの実装からBevyを学んでいくというシリーズものの記事を書いてみようと思ってますので, よければそちらもお楽しみに!

ちなみに, タイトルのアーティファクトというのは, このシステムが少し特殊すぎて, 後世に引き継げず文字通りのアーティファクト, あるいはオーパーツとして扱われる可能性が高そうなことから来ています. まあこの記事とBevy解説記事, Nix解説記事を書けばまあ大丈夫だと思います......が. 

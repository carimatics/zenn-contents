---
title: "Angularにおける組み込み制御フローの導入とその背景"
emoji: "✍️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Angular"]
published: true
published_at: 2023-12-02 00:00
---

本稿は[Angular Advent Calendar 2023](https://qiita.com/advent-calendar/2023/angular)の2日目の記事です。

# 概要

最近のAngularにとって最もインパクトのあるニュースといえば、今年11月にリリースされたAngular v17だと思います。
本稿では、そのv17リリースの目玉機能である組み込み制御フロー(built-in control flow)について紹介します。

組み込み制御フロー自体はシンプルな機能ですが、その背景にはAngularのロードマップに掲げられた目標があり、Angularの将来や開発者にとって大きな意味を持つものとなっています。
そのため、本稿では組み込み制御フローの簡単な機能紹介に加えて、その背景となる目標についても詳しく見ていきます。
本稿からAngularが将来どのような方向に向かっていくのかを少しでも感じていただければ幸いです。

# 組み込み制御フローとは

Angularの組み込み制御フロー(built-in control flow)は、いわゆる`if`や`for`、`switch`といった制御フローをAngularのテンプレートで利用できるようにする機能です。
組み込み制御フローを利用することで、プログラミング言語(JavaScript/TypeScript)に近い構文でコンポーネントの表示/非表示や繰り返しの構造を表現できるようになります。
例えば、以下のようにテンプレート内で`@if`や`@for`、`@switch`の構文を利用できます。

```html
@if (user.loggedIn) {
  <app-dashboard />
} else {
  <app-login />
}

@switch (user.role) {
  @case ('admin') { <app-admin-dashboard /> }
  @case ('manager') { <app-dashboard /> }
  @default { <app-user-dashboard /> }
}

@for (item of items; track item) {
  {{ item }}
} @empty {
  <p>No items found</p>
}
```

v16までのAngularでも、`*ngIf`や`*ngFor`、`*ngSwitch`の構造ディレクティブを利用することにより、組み込み制御フローと同等の機能を実現できました。
つまり、組み込み制御フローが導入されたとはいえ、(アプリケーションの機能レベルでは)表現能力が向上したわけではありません。

では、なぜ組み込み制御フローが導入されたのでしょうか。

## なぜ組み込み制御フローが導入されたか

そもそもの話になりますが、現在の[Angularのロードマップ](https://angular.dev/roadmap)には以下の2つの項目が掲げられています。

- Angular開発者のDX(Developer Experience)改善
- Angularフレームワークのパフォーマンス改善

組み込み制御フローの導入も例に漏れず、これら2つの目標を達成するための取り組みの一つとなっています。

組み込み制御フローをAngularに導入するモチベーションに関しては、[RFC](https://github.com/angular/angular/discussions/50719)に詳しく書かれています。
私の理解では、以下の2点が主なモチベーションとなっています。

- Angular開発者のDXの改善
- ZonelessなAngularアプリケーションの実現

順に詳しく見ていきましょう。

### Angular開発者のDXの向上

これはAngularのロードマップに直結する項目です。

Angularチームの調査によると、構造ディレクティブによるマイクロ構文ベースの制御フローは、他のフレームワークの構文と比較してDXが低くなってしまっているようです。
RFCでは上記程度の言及に留まっていますが、具体的にどのようにDXが改善されるのでしょうか。

わかりやすいものとしては、表現がシンプルになります。
例えば、条件によってコンポーネントの出し分けをしたい場合、`*ngIf`と`@if`を比較すると以下のようになります。

```html
<!-- *ngIf -->
<app-dashboard *ngIf="user.loggedIn; else loginBlock" />
<ng-template #loginBlock><app-login /></ng-template>

<!-- @if -->
@if (user.loggedIn) {
  <app-dashboard />
} else {
  <app-login />
}
```

`@if`については行数こそ多くなっているものの、JavaScriptの構文に近いため、ほとんどの開発者にとっては直感的に理解できると思います。
対して`*ngIf`では要求に対してノイズが多く、本質的ではない部分の読み書きが必要になります。

条件によってコンポーネントを出し分けたいというのは、動的なWebアプリケーションを作成する上で非常に一般的な要求です。
にも関わらず、Angularではこのようなシンプルな機能を実現するために、構造ディレクティブやテンプレート構文といった概念を理解する必要があります。
これは`*ngIf`に限らず、`*ngFor`や`*ngSwitch`などの構造ディレクティブにも言えることです。

この問題は初学者にとってはより顕著になっているように感じます。
Angularを学び始めてテンプレートを記述しようとした時、初学者はテンプレートの多様な機能によって構成される複雑な記述に圧倒されることでしょう。
例を挙げると、プロパティバインディングやイベントバインディング、属性バインディング、双方向バインディング、パイプ、テンプレート参照変数、そして構造ディレクティブなどです。
かなりシンプルな機能のみを持つアプリケーションを作成する場合ですら、上記の機能をほぼ確実に利用することになります。
さらに、Angularで学ぶべきはテンプレートの記述方法だけではなく、TypeScript、コンポーネントのライフサイクル、依存性注入、RxJSなどのライブラリ、Angular CLIなどのツールについても学ぶ必要があります。
これらの点を踏まえると、組み込み制御フローによりもたらされるシンプルな構文によって、初期段階で越えるべき大きな壁が一つなくなったと考えられるのではないでしょうか。

また、`@switch`は渡した条件によって各`@case`ブランチでコンパイル時の型チェックができます。
例えば、以下のようなテンプレートをエラーにできます。

```
@switch (user.role) {
  @case ('admin') { <app-admin-dashboard /> }
  @case ('manager') { <app-dashboard /> }
  <!-- user.role と 1 は型が一致しないためエラー -->
  @case (1) { <invalid /> }
  @default { <app-user-dashboard /> }
}
```

このように、組み込み制御フローを利用することで、Angularのテンプレートでより型安全なコードを記述できるようになります。

組み込み制御フローと構造ディレクティブの比較については、以下の文献も参考にしてみてください。

- [Built-in control flow](https://angular.io/guide/control_flow#comparing-built-in-control-flow-to-ngif-ngswitch-and-ngfor)
- [Introducing Angular v17](https://blog.angular.io/introducing-angular-v17-4d7033312e4b#088e)

### ZonelessなAngularアプリケーションの実現

これはAngularのロードマップにおけるDXおよびフレームワークのパフォーマンス改善に関わる項目です。

現在のAngularではリアクティビティを実現する(コンポーネントの状態変更を検知して画面に再描画する)ために[`zone.js`](https://www.npmjs.com/package/zone.js)を利用しています。
Angularにとって重要な存在である`zone.js`ですが、DXやパフォーマンスの面で多くの課題があり、Angularチームはzoneless(`zone.js`を利用しない)なAngularアプリケーションを実現するための取り組みを行っています。
これらについてはv16から導入された[Angular SignalsのRFC](https://github.com/angular/angular/discussions/49685)の中で詳しく説明されています。

組み込み制御フローのRFCでも言及されているように、構造ディレクティブによる制御フローは`zone.js`の仕組みに依存しており、zonelessなAngularアプリケーションの実現のためにはそれらを別の手段で置き換える必要があります。
そのために組み込み制御フローが提案され、v17で実装されたというわけです。

# まとめ

Angular v17から追加された組み込み制御フローとその背景について紹介しました。
シンプルな構文で制御フローを記述できるようになったことで、Angular開発者のDXが向上するとともに、zonelessなAngularアプリケーションの実現にも近づいているということがわかりました。

記事に間違いや補足、感想などがありましたらコメントしていただけるととても嬉しいです😋

今年も残り短くなってきましたが、健康に気をつけてお過ごしください。
それでは良いAngularライフを👋

# 参考

- [Introducing Angular v17](https://goo.gle/angular-v17)
- [RFC: Built-in Control Flow](https://github.com/angular/angular/discussions/50719)
- [Angular Roadmap](https://angular.dev/roadmap)
- [RFC: Angular Signals](https://github.com/angular/angular/discussions/49685)
- [Angularの最新情報がわかる！Monthly Angular 10月号 【ng-japan OnAir #70】](https://www.youtube.com/watch?v=O-d0_WLQgJg)

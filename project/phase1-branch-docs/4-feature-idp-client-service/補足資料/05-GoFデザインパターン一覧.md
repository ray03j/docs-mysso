# GoF デザインパターン一覧

## 目次

- [背景](#背景)
- [分類の軸](#分類の軸)
- [生成に関するパターン(5)](#生成に関するパターン5)
- [構造に関するパターン(7)](#構造に関するパターン7)
- [振る舞いに関するパターン(11)](#振る舞いに関するパターン11)
- [Template Method の位置づけ](#template-method-の位置づけ)
- [GoF 以外のパターン体系](#gof-以外のパターン体系)

---

## 背景

`04-認証フィルタの適用範囲を制御する設計パターン.md` で Template Method を「採用例」として扱ったが、これは Rails 固有の考えではなく、**GoF(Gang of Four)の書籍『Design Patterns』(Gamma ら、1994)で定義された23の汎用デザインパターンの1つ**である。

本ドキュメントでは GoF の23パターンをすべて列挙する。各パターンの「一言での目的」と、Ruby/Rails での対応例(分かるもののみ)を併記する。各パターンの詳細な解説は専門書に譲り、ここでは「名前を聞いたときに何かを思い出せる」索引を目指す。

## 分類の軸

GoF は23パターンを2つの軸で分類している。

**目的による分類**(本書の章立て):

- **生成(Creational)**: オブジェクトの生成方法に関するパターン(5個)
- **構造(Structural)**: クラスやオブジェクトの組み合わせ方に関するパターン(7個)
- **振る舞い(Behavioral)**: オブジェクト間の責務分担・通信に関するパターン(11個)

**スコープによる分類**:

- **クラスパターン**: 継承で振る舞いを決める(実行時に固定)。Factory Method、Adapter(クラス版)、Interpreter、Template Method の4個
- **オブジェクトパターン**: オブジェクトの合成で振る舞いを決める(実行時に差し替え可能)。残り19個

## 生成に関するパターン(5)

| パターン | 目的(一言) | Ruby/Rails での例 |
|---|---|---|
| Abstract Factory | 関連するオブジェクト群をまとめて生成する窓口 | — |
| Builder | 複雑なオブジェクトを段階的に構築する | `Nokogiri::XML::Builder` |
| Factory Method | 生成処理をサブクラスに委譲する(クラススコープ) | — |
| Prototype | 既存オブジェクトの複製から生成する | `Object#dup` / `#clone` が言語組込み |
| Singleton | インスタンスを1つに限定する | 標準ライブラリ `Singleton` モジュール |

## 構造に関するパターン(7)

| パターン | 目的(一言) | Ruby/Rails での例 |
|---|---|---|
| Adapter | 合わないインターフェースを変換して接続する | ActiveRecord の DB アダプタ、ActiveJob のキューアダプタ |
| Bridge | 抽象と実装を分離し、それぞれ独立に拡張可能にする | — |
| Composite | 部分と全体を同じインターフェースで扱う(木構造) | — |
| Decorator | オブジェクトに動的に機能を付け足す | Rack ミドルウェアの入れ子構造、Draper gem |
| Facade | 複雑なサブシステムに単純な窓口を用意する | — |
| Flyweight | 細かいオブジェクトを共有してメモリを節約する | Ruby の Symbol、`frozen_string_literal` |
| Proxy | 本物の代理としてアクセスを制御する | `ActiveRecord::Associations::CollectionProxy`(遅延ロード) |

## 振る舞いに関するパターン(11)

| パターン | 目的(一言) | Ruby/Rails での例 |
|---|---|---|
| Chain of Responsibility | 要求を処理者の鎖に沿って渡す | Rack ミドルウェアスタック |
| Command | 要求をオブジェクトとしてカプセル化する | ActiveJob のジョブ、`Proc` / `lambda` |
| Interpreter | 言語の文法を表現し評価する(クラススコープ) | — |
| Iterator | 内部構造を隠して要素を順にたどる | `Enumerable` / `each`、`Enumerator` |
| Mediator | オブジェクト間の相互作用を仲介役に集約する | — |
| Memento | 状態を保存し、後で復元できるようにする | — |
| Observer | 状態変化を依存オブジェクトへ通知する | `ActiveSupport::Notifications`、モデルコールバック |
| State | 状態に応じて振る舞いを切り替える | AASM / `state_machine` 系 gem |
| Strategy | アルゴリズムを差し替え可能にする | Warden の strategies(Devise の土台)、`Proc` を渡す方式全般 |
| Template Method | 処理の骨格を親が決め、変動部分をサブクラスに委ねる(クラススコープ) | 本プロジェクトの `internal_api?`、コールバックのフック |
| Visitor | データ構造と操作を分離し、操作を後から追加可能にする | — |

## Template Method の位置づけ

- **GoF の「振る舞い」パターンの1つ**で、スコープは「クラスパターン」(継承を使う点が特徴)
- 定義: メソッドの処理手順(骨格)を親クラスで定義し、手順の一部をフックメソッドとして切り出す。サブクラスはフックのオーバーライドだけで振る舞いを変えられる
- Rails 固有の概念ではなく、Rails がこのパターンの**利用者**である。`03-内部APIキー認証の仕組み.md` の `before_action ... if: :internal_api?` は、Rails のフィルタ機構(骨格)に対し、`internal_api?` をフックとして差し込んだ適用例
- 同じ目的(振る舞いの差し替え)を継承ではなく合成で実現するのが **Strategy** であり、両者はしばしば対比される。`04-` のパターン6(ポリシーレジストリ)は「条件判定をデータ化して注入する」点で Strategy に近い発想と言える

## GoF 以外のパターン体系

GoF(1994)は最も有名なカタログだが、パターンの体系は他にも存在する。本書では列挙の対象外とし、存在の紹介に留める。

- **PoEAA(Patterns of Enterprise Application Architecture)**: Martin Fowler による企業アプリケーション向けパターン集。Rails の命名に直結するものが多い — **Active Record** パターン、**MVC**、**Front Controller**(`config.ru` / dispatcher)など。Rails を「GoF より先に PoEAA の観点で読む」と理解が早い
- **マルチスレッド/並行処理のパターン**: 『Java言語で学ぶデザインパターン入門 マルチスレッド編』など。Producer-Consumer、Read-Write Lock など
- **マイクロサービス関連のパターン**: Circuit Breaker、Saga、API Gateway など。本プロジェクトの「内部APIキー認証」自体は、マイクロサービス間認証の文脈では最も単純な共有シークレット方式に相当する

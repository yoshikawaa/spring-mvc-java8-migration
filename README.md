# Spring MVC の Java 8 マイグレーション

独断と偏見によるマイグレーション基準

ここで扱うのは、以下の3点

* [`Optional`](#optionalの適用)
* [`@FunctionalInterface`](#functionalinterfaceの適用)
* [`Stream`](#streamの適用)

Date and Time APIなんかは考慮しない

## `Optional`の適用

値が任意のもの、`Null`チェックが必要なものは、`Optional`に置き換えたほうが良い  

基本的な考え方としては、

* フィールド -> `Optional`には適していない
* ローカル変数 -> `Optional`には適していない
* メソッド引数 -> 基本的には`Optional`には適していないが、Controllerでは活用できる
* メソッド戻り値　-> `Optional`に適している

> メソッド戻り値を`Optional`にすることで、戻り値が`Null`となる場合の挙動を呼び出し元に委ねることが容易となる  
> それ以外の使い方は、基本的に想定されていない
> [Java SE 8 Optional, a pragmatic approach](http://blog.joda.org/2015/08/java-se-8-optional-pragmatic-approach.html)

### Controller

#### 引数（`@PathVariable`、`@RequestParam`、`@RequestAttribute`、`@SessionAttribute`、  `@RequestHeader`、`@CookieValue`）

（１）必須の場合、`Optional`に置き換えない（置き換えると、必須でなくなってしまう）

```java
public String sample(@RequestParam("param") String param) {...}
```

（２）任意の場合、`Optional`に置き換える

```java
public String sample(@RequestParam(name = "param", required = false) String param) {...}
```
to
```java
public String sample(@RequestParam(name = "param") Optioanl<String> param) {...}
```

※`Null`をそのまま扱いたい場合は、`required = false`のままで良い（`Optional#orElse(null)`はナンセンス）

（３）デフォルト値が指定されている場合、`Optional`に置き換えない（`Null`になることがない）

```java
public String sample(@RequestParam(name = "param", defaultValue = "hoge") String param) {...}
```
to
```java
public String sample(@RequestParam(name = "param", defaultValue = "hoge") String param) {...}
```
※`required = false`は削除する（`defaultValue`を指定すると、自動的に`required = false`になる）

#### 引数（フォームオブジェクト）

リクエストパラメータが任意の場合も、`Optional`に置き換えない

> フォームオブジェクトのプロパティは、リクエストパラメータが存在しない場合はバインドされない（`Null`のまま）ため、プロパティが`Optional`型であっても`Null`のまま（`Optional#ifPresent(...)`を実行するとNPE）

#### 戻り値

`Optional`に置き換えない

### Repository

#### MyBatis 3

MyBatis 3.4系は`Optional`をサポートしていないため、置き換えない

> MyBatis 3.5から`Optional`がサポートされる

#### JPA

JPAを利用する場合は、戻り値が`Null`となる可能性があるものは、`Optional`に置き換える

> Spring Data 5から`Optional`がサポートされ、`CrudRepository`の一部メソッドの戻り値が`Optional`となった  
> 戻り値が`Null`となる可能性があるが`Optional`とそうでないものが混在するのは設計としてNGなので、Repository全体的に置き換えたほうが良い

```java
public Todo findById(String id) {...}
```
to
```java
public Optional<Todo> findById(String id) {...}
```

### Service

戻り値が`Null`となる可能性があるものは、`Optional`に置き換える

```java
public Todo findOne(String id) {
    return repository.findOne(id);
}
```
to
```java
public Optional<Todo> findOne(String id) {
    return Optional.ofNullable(repository.findOne(id));
}
```

> **要議論** TERASOLUNAの思想「App層とDomain層は独立」を尊重すれば、`Optional`に置き換えるのが妥当だが、実際の開発ではApp層とDomain層は同じ開発者が開発しており、必ずしも`Null`の挙動を呼び出し元に委ねる必要性は大きくない  
> `Optional`に置き換えるコストメリットある？

### その他

#### `@Value`

値が任意の場合も、`Optional`に置き換えない

> * `@Value("${prop:default-value}")`でデフォルト値をセットできる
> * `@Value("${prop:}")`とすると、デフォルト値は空文字となる
> * `@Value("${prop:#{null}}")`とすると、デフォルト値は`Null`となるが、ここまでして`Null`を扱いたいケースは稀

### `Optional`適用における注意点

#### `Optional`の生成

* 必ずある値（`Null`でない）を元にする場合は、`Optioanl#of()`
* `Null`になる可能性のある値を元にする場合は、`Optional#ofNullable()`
* `Null`を返却する場合は、`Optional#empty()`（戻り値が`Optional`の場合は、`Null`を返却してはならない）

#### `Null`の判定

* `Null`でない場合に処理するなら、`Optional#ifPresent(...)`
* `Null`かどうか判断するなら、`Optional#isPresent()`（できる限り、`ifPresent`で実装できないか検討する）

#### 値の取り出し

* `Null`ではいけない場面なら、`Optional#get()`（`Null`なら`NoSuchElementException`がスローされる）
* `Null`のときにスローする例外を指定するなら、`Optional#orElseThrow(...)`
* `Null`のときに代替値を利用するなら、`Optional#orElse("代替値")`
* `Null`のときに代替値をロジックで生成するなら、`Optional#orElseGet(...)`
* `Optional#orElse(null)`は極力避ける（通常、`Optional`のまま透過すれば良い）

----

## `@FunctionalInterface`の適用

フレームワーク部品、共通部品のインターフェイスは、`@FunctionalInterface`を付与すると良い

> メソッドが一つのみ定義されたインターフェイスを関数インターフェイスとして利用することができ、慣習として`@FunctionalInterface`を付与する
> （付与しなくても、関数インターフェイスとして利用することは可能）

```java
@FunctionalInterface
public interface Callback {
    void execute();
}
```

### Service、Repository

インターフェイスに`@FunctionalInterface`を付与しない

> インターフェイス＋実装クラスのセット実装が基本で、関数インターフェイスとして利用することはないと思われるため、関数インターフェイスとしての利用を推奨する`@FunctionalInterface`を付与するのは妥当ではない

### `@FunctionalInterface`の利用

無名クラス（匿名クラス）を実装しているところは、関数インターフェイスを利用したラムダ式に置き換える（置き換えられるか検討する）

```java
xxx.setCallback(new Callback() {
    @Override
    public void execute() {
        LOGGER.info("executed");
    }
});
```
to
```java
xxx.setCallback(() -> LOGGER.info("executed"));
```

----

## `Stream`の適用

フレームワーク部品では積極的に`Stream`を適用すると良い

> **要議論** 既に実装済みのアプリケーションやテストコードまで`Stream`を適用するコストメリットある？

### `Stream`の基本

既存のループ処理を、3つの要素に分解できる

```java
List<Admin> admins = new ArrayList<>();
for (User user : users) {
    if ("Admin".equals(user.getRole()) {
        Admin admin = new Admin(user.getId(), user.getName());
        admins.add(admin);
    }
}
```
to
```java
List<Admin> admins = users.stream()
                          .filter(user -> "Admin".equals(user.getRole()))
                          .map(user -> new Admin(user.getId(), user.getName()))
                          .collect(toList());
```

* `Stream`の作成 -> コレクションや配列から`Stream`を取得する
* 中間操作 -> `Stream`の絞込み（`filter(...)`）や加工（`map(...)`）を行う
* 終端操作 -> `Stream`から新しいコレクションの取得（`collect(...)`）や、順次処理（`forEach(...)`）を行う

これにより、データの選定と処理が分離でき、プログラムの可読性が向上する

また、`Collection#parallelStream()`や`Stream#parallel()`で容易に並列実行を実現できる

### `Stream`が適さない場合

基本的に、拡張for文で実現するのが難しいところは、`Stream`に置き換えるのも難しい

* 複数のコレクションを同時にループする場合
* ループのインデックスが欲しい場合
* 要素のグルーピングなど、複数要素を同時に扱いたい場合

また、並列実行や順序制御には注意が必要

* コレクションの順序を保持して処理を行うなら、`forEachOrdered(...)`
* スレッドアンセーフなオブジェクトを扱うなら、`Collection#parallelStream()`や`Stream#parallel()`は使用不可

---
layout: article
title: Scala+PlayFlameworkを最初から
tags: scala PlayFramework
aside:
  toc: true
---

## 概要

PlayFrameworkを使ってScalaのプロジェクトを作ってみる

## プロジェクトセットアップ

```
sbt new playframework/play-scala-seed.g8
```

プロジェクト名とオーガニゼーションは以下を設定
```
name [play-scala-seed]: todo
organization [com.example]: com.my.play
```


## 起動確認
```
cd todo
sbt run
```

しばらく待ってから以下にアクセス
```
localhost:9000
```

![image](https://user-images.githubusercontent.com/44778704/118975843-587e0000-b9af-11eb-9164-e492b0b7766f.png)

## ディレクトリ解説

- .g8: Playのスクリプトファイルが詰まっている。編集することは多分ない
- .gradle: ビルドで生成されるファイル類を管理
- app: プログラム本体。ここをメインに触る
- conf: 設定やルーティングのファイルを置くところ
- gradle: Gradleのファイル群
- project: SBTの関連ファイル群
- public: 静的ファイル置き場
- target: ビルドの中間ファイル
- test: テストコード置き場

## router

conf/routeに以下を書く
```java
GET / controllers.HomeController.index(id:Int, name:Option[String])
```

## controller
app/controllers/HomeController.scalaに以下を書く

```scala
package controllers
import javax.inject._
import play.api.mvc._

@Singleton
class HomeController @Inject()(val controllerComponents: ControllerComponents) extends BaseController {

  def index(id:Int, name:Option[String]) = Action { implicit request: Request[AnyContent] =>
    println("id: " + id)
    println("name: "+name.getOrElse("no-name"))
    Ok("id: " + id + ", name: "+name.getOrElse("no-name"))
  }
}
```

`  def index(id:Int, name:Option[String]) ` で、nameをOption型で指定している。こうすると、例えば以下のようなリクエストが来た時にエラーにならない。

```
http://localhost:9000/?id=1
```

このように、Optionにしたパラメータはリクエストに情報がなくても問題ないが、逆にOptionにしなかった場合はエラーになる。例えばidの方はOptionにしていないので、
```
http://localhost:9000/
```
↑このようにリクエストするとエラーになる。


</br>
</br>
つづく...


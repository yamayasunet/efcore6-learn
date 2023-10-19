# efcore6-learn
Entity Framework (EF) Core 6 の謎

## 問題

テーブル　Information

| 物理名 | タイプ| 備考 |
|--|--|--|
|id|num|プライマリキー(not null)|
|title|text|タイトル(null許可)|
|message|text|内容(null許可)|


### No.1 Entityを追加する
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Add(info1);
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー：データ変更なし  
  追加要求しているのに、保存メソッドを呼んでいないので、エラーになるはず！  

|id|title|message|
|-|-|-|

2. システムエラー：データ追加あり  
  追加には成功し、自動コミットなのでDBに反映されるが、保存メソッドを呼んでいないので、エラーになるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

3. 正常終了：データ変更なし   
  保存メソッドを呼んでいないので、DBに反映されない。保存メソッドを呼び出していないが、DB操作についてはなかったことに（無視）されるはず！  

|id|title|message|
|-|-|-|

4. 正常終了：データ追加あり  
  保存メソッドを呼び出していないが、DB操作は行っているので自動でDBに反映されるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||


<details>
<summary>こたえ</summary>

3. 正常終了：データ変更なし   

> SaveChanges()を呼び出していないので、すべての操作は無視されて更新されない。
</details>


---
### No.2 Entityを追加した後にSaveを呼び出す
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Add(info1);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー：データ変更なし  
  データベースのすべての項目を設定していないので、エラーになるはず！  

|id|title|message|
|-|-|-|

2. システムエラー：データ追加あり  
  データベースに反映されるが、明示的なトランザクションの開始を行っていないので、エラーになるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

3. 正常終了：データ変更なし   
  明示的なトランザクションの開始を行っていないので反映されないで終了するはず！  

|id|title|message|
|-|-|-|

4. 正常終了：データ追加あり  
  お任せしているので自動的にDBに反映されるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||


<details>
<summary>こたえ</summary>

4. 正常終了（データ追加あり）
 
> コードの通り、データベースに格納される
</details>


---
### No.3 データ追加した後でEntityをいじる
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Add(info1);
    // -- (中略) --
    info1.title = "EF Core の謎";
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```


1. システムエラー：データ変更なし  
  データを追加を依頼した後に、データの更新を行おうとしているので、エラーになるはず！  

|id|title|message|
|-|-|-|

2. システムエラー：データ追加あり  
  データの追加は成功し自動コミットされるが、データの更新操作を行おうとして、エラーになるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

3. 正常終了：データ変更なし   
  データの追加は成功し自動コミットされる。
  タイトルの更新はUpdateメソッドを呼び出していないので、データベースには反映されないはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

4. 正常終了：データ追加・更新あり  
  EF Coreがうまいことやってくれるので、データ追加も、タイトル更新も実行されるはず！  

|id|title|message|
|-|-|-|
|101|EF Core の謎||


<details>
<summary>こたえ</summary>

4. 正常終了：データ追加・更新あり  
 
> EF Coreの管理下にあるEntityは変更追跡される。
</details>


---
### No.4 データ追跡管理化にない状態でUpdateする/更新後にEntityをいじる
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが１件 （DB {id = 101 , title = "タイトル", message = null }）
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "更新後" };
    context.Informations.Update(info1);
    // -- (中略) --
    info1.title = "EF Core の謎";
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー：データ変更なし  
  データを読み込んでいない(Selectしていない)のでデータ追跡ができず、エラーになるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

2. システムエラー：データ変更あり  
  Updateまでの操作は自動コミットにより反映され、titleの変数操作が追跡管理外になるのでエラーになるはず！  

|id|title|message|
|-|-|-|
|101|更新後||

3. 正常終了：データ変更あり  
  Update呼び出し直前までが反映される。  
  その後の操作はUpdateメソッドを呼び出していないので、データベースには反映されないはず！  

|id|title|message|
|-|-|-|
|101|更新後||

4. 正常終了：データ追加・更新あり  
  EF Coreがうまいことやってくれるので、タイトルも更新されるはず！  

|id|title|message|
|-|-|-|
|101|EF Core の謎||


<details>
<summary>こたえ</summary>

4. 正常終了：データ追加・更新あり  
 
> Update命令で管理下になり、Entityは変更追跡され、最終的な結果が永続化される。
</details>


---
### No.5 更新対象のデータがない
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Update(info1);
    // -- (中略) --
    info1.title = "EF Core の謎";
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー：データ変更なし  
  データがデータベースにないので、エラーになるはず！  

|id|title|message|
|-|-|-|

2. 正常終了：データ変更なし   
  update information set title="タイトル" where id = id、update information set title="EF Core の謎" where id = id というSQLが流れるはずなので、一致するデータがなく、何もしなかった時と同じになるはず！  

|id|title|message|
|-|-|-|

3. 正常終了：データ追加あり  
  データベースにデータが存在しないため自動で作成してくれるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

4. システムエラー：データ追加・更新あり  
  データベースにデータが存在しないため自動で作成して、さらに更新もやってくれるはず！  

|id|title|message|
|-|-|-|
|101|EF Core の謎||


<details>
<summary>こたえ</summary>

1. システムエラー：データ変更なし  
 
> 更新対象のデータが見つからないので、エラーになる。
</details>


---
### No.6 更新して追加
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Update(info1);
    // -- (中略) --
    context.Informations.Add(info1);
    info1.title = "EF Core の謎";
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー：データ変更なし  
  Updateの時点でデータがデータベースにないので、エラーになるはず！  

|id|title|message|
|-|-|-|

2. システムエラー：データ変更なし   
  Addする際に、Update命令が先に実行されているため、エラーにしてくれるはず！ 

|id|title|message|
|-|-|-|

3. システムエラー：データ追加あり  
  Update命令は空振りし、Addが実行されるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

4. 正常終了：データ追加・更新あり  
  Update命令は空振りし、Addが実行され、さらに title も更新されるはず！  

|id|title|message|
|-|-|-|
|101|EF Core の謎||


<details>
<summary>こたえ</summary>

4. 正常終了：データ追加・更新あり  
 
> 一番最初のUpdateはなかったことにされる。
</details>









---
### No.7 追加して更新
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Add(info1);
    // -- (中略) --
    info1.title = "EF Core の謎";
    context.Informations.Update(info1);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. 正常終了：データ追加・更新あり  
   ソースに書いた通り、AddしてUpdateされるはず！  
   
|id|title|message|
|-|-|-|
|101|EF Core の謎||

2. システムエラー：データ変更なし









1. システムエラー：データ変更なし  
  Updateの時点でデータがデータベースにないので、エラーになるはず！  

|id|title|message|
|-|-|-|

2. システムエラー：データ追加あり   
  Addは成功して自動コミットされ、Update命令が先に実行されているため、エラーにしてくれるはず！ 

|id|title|message|
|-|-|-|

3. システムエラー：データ追加あり  
  Update命令は空振りし、Addが実行されるはず！  

|id|title|message|
|-|-|-|
|101|タイトル||

4. 正常終了：データ追加・更新あり  
  Update命令は空振りし、Addが実行され、さらに title も更新されるはず！  

|id|title|message|
|-|-|-|
|101|EF Core の謎||


<details>
<summary>こたえ</summary>

4. 正常終了：データ追加・更新あり  
 
> 一番最初のUpdateはなかったことにされる。
</details>







































































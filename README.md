# efcore6-learn
Entity Framework (EF) Coreの謎

## 問題

テーブル　Information

| 物理名 | タイプ| 備考 |
|--|--|--|
|id|num|プライマリキー|
|title|text|タイトル|
|message|text|内容|


### No.1
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

1. システムエラー（データ変更なし）
2. システムエラー（DB {id = 101 , title = "タイトル", message = null }）
3. 正常終了（データ変更なし）
4. 正常終了（DB {id = 101 , title = "タイトル", message = null }）

<details>
<summary>こたえ</summary>

3. 正常終了（データ変更なし）

> SaveChanges()を呼び出していないので、更新されない。
</details>


### No.2
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

1. システムエラー（データ変更なし）
2. システムエラー（DB {id = 101 , title = "タイトル", message = null }）
3. 正常終了（データ変更なし）
4. 正常終了（DB {id = 101 , title = "タイトル", message = null }）

<details>
<summary>こたえ</summary>

4. 正常終了（DB {id = 101 , title = "タイトル", message = null }）
 
> コードの通り、データベースに格納される
</details>


### No.3
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Add(info1);
    info1.title = "EF Core の謎";
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー（データ変更なし）
2. システムエラー（DB {id = 101 , title = "タイトル", message = null }）
3. 正常終了（DB {id = 101 , title = "タイトル", message = null }）
4. 正常終了（DB {id = 101 , title = "EF Core の謎", message = null }）

<details>
<summary>こたえ</summary>

4. 正常終了（DB {id = 101 , title = "EF Core の謎", message = null }）
 
> Addにて管理下にしたEntityは変更追跡されるので、最終的な結果が永続化される。
</details>

### No.4
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Update(info1);
    info1.title = "EF Core の謎";
    context.Informations.Add(info1);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー（データ変更なし）
2. システムエラー（DB {id = 101 , title = "タイトル", message = null }）
3. 正常終了（DB {id = 101 , title = "タイトル", message = null }）
4. 正常終了（DB {id = 101 , title = "EF Core の謎", message = null }）

<details>
<summary>こたえ</summary>

4. 正常終了（DB {id = 101 , title = "EF Core の謎", message = null }）
 
> 管理していないEntityの操作が無視され、タイトル編集後のデータがDBに追加される。
</details>

### No.5
以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
> - データベース内にはデータが存在しない  
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 101, title = "タイトル" };
    context.Informations.Update(info1);
    info1.title = "EF Core の謎";
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. システムエラー（データ変更なし）
2. システムエラー（DB {id = 101 , title = "タイトル", message = null }）
3. 正常終了（DB {id = 101 , title = "タイトル", message = null }）
4. 正常終了（DB {id = 101 , title = "EF Core の謎", message = null }）

<details>
<summary>こたえ</summary>

1. システムエラー（データ変更なし）
 
> 管理していないEntityの操作をしようとして、データベースにデータがないのでエラーになる。
</details>
























































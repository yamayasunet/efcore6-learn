# efcore6-learn

Entity Framework (EF) Core 6 の謎

## 前提条件

使用するテーブルは以下の通り

- テーブル　 Information

| 物理名  | タイプ | NOT NULL | Default | 備考                              |
| ------- | ------ | -------- | ------- | --------------------------------- |
| id      | num    | YES      |         | プライマリキー(ユニーク/自動採番) |
| title   | text   | NO       |         | タイトル                          |
| message | text   | NO       | 未設定  | 内容                              |

## 基礎知識

EF Core における基本的なデータベース操作命令は以下の通り

- 追加

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 1, title = "タイトル", message = "内容" };
    context.Informations.Add(info);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```
実行されるSQL
```SQL
INSERT INTO information (id, title, message)
      VALUES (@p0, @p1, @p2)
```

- 更新

```csharp
DBContext context = GetDbContext();
try {
    Information info = context.Informations.Find(1);
    info.title = "タイトル(更新後)";
    context.Informations.Update(info);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```
実行されるSQL
```SQL
UPDATE information
SET title
      VALUES (@p0, @p1, @p2)
```

- 削除

```csharp
DBContext context = GetDbContext();
try {
    Information info = context.Informations.Find(1);
    context.Informations.Remove(info);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

## 問題

### No.1 SaveChanges を省略

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 1, title = "タイトル", message = "内容" };
    context.Informations.Add(info);
} catch (Exception e){
   throw e;
}
return OK;
```

1. 正常終了：データが追加される  
   セーブ命令を忘れても、EF Core が気を利かせて反映してくれる。

| id  | title    | message |
| --- | -------- | ------- |
| 1   | タイトル | 内容    |

2. 正常終了：データは追加されない  
   セーブ命令を実行していないので、編集中の内容はすべて破棄される。  
   （データベースに反映されない）

| id  | title | message |
| --- | ----- | ------- |

3. エラー：警告してくれる  
   値を変更(追加)しているのに、セーブ命令を実行していないので教えてくれる。

| id  | title | message |
| --- | ----- | ------- |

4. エラー：データが追加される  
   セーブ命令を忘れても、EF Core が気を利かせて反映してくれ、自動コミットモードなので DB に反映される。  
   しかし、セーブ命令を実行していないので最後に教えてくれる。

| id  | title    | message |
| --- | -------- | ------- |
| 1   | タイトル | 内容    |

<details><summary>こたえ</summary>

2. 正常終了：データは追加されない

> SaveChanges()を呼び出していないので、すべての操作は無視されて更新されない。

</details>

---

### No.2 項目の省略

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info = new() { id = 1, title = "タイトル" };
    context.Informations.Add(info);
    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. エラー：データは追加されない
   message 項目が設定されていないのでエラーになる。

| id  | title | message |
| --- | ----- | ------- |

2. 正常終了：１件追加される ①
   message 項目は DB デフォルト値になる。

| id  | title    | message |
| --- | -------- | ------- |
| 1   | タイトル | 未設定  |

3. 正常終了：１件追加される ②
   message 項目は設定していないので null になる。

| id  | title    | message |
| --- | -------- | ------- |
| 1   | タイトル | [NULL]  |

<details><summary>こたえ</summary>

3. 正常終了：１件追加される ②

> プログラムで設定した値になる(設定していない場合はプログラム側の項目のデフォルト(null))

</details>

---

### No.3 連続 Insert①

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info = new() { id = 1, title = "タイトル", message = "内容" };
    context.Informations.Add(info);

    info.id = 2;
    context.Informations.Add(info);

    info.id = 3;
    context.Informations.Add(info);

    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. 正常終了：３件追加される  
   実装通り、１～３が保存される。

| id  | title    | message |
| --- | -------- | ------- |
| 1   | タイトル | 内容    |
| 2   | タイトル | 内容    |
| 3   | タイトル | 内容    |

2. エラー：１件追加される
   ２件目でインスタンスを使いまわしていることを怒られる。  
   ただし、トランザクションは自動のためその部分までは保存される。

| id  | title    | message |
| --- | -------- | ------- |
| 1   | タイトル | 内容    |

3. 正常終了：１件追加される  
   一番最後に追加したものだけ保存される。

| id  | title    | message |
| --- | -------- | ------- |
| 3   | タイトル | 内容    |

4. エラー：なにも追加されない  
   ２件目でインスタンスを使いまわしていることを怒られる。  
   怒られたことにより、自動でロールバックされる。

| id  | title | message |
| --- | ----- | ------- |

<details><summary>こたえ</summary>

3. 正常終了：１件追加される

> インスタンスの最後の状態が保管される。

</details>

---

### No.4 連続 Insert②

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 1, title = "タイトル1", message = "内容" };
    Information info2 = new() { id = 2, title = "タイトル2", message = "内容" };
    Information info3 = new() { id = 3, title = "タイトル3", message = "内容" };
    Information info5 = new() { id = 5, title = "タイトル4", message = "内容" };

    context.Informations.Add(info1);
    context.Informations.Add(info2);
    context.Informations.Add(info3);
    context.Informations.Add(info1); // ← 注目！
    context.Informations.Add(info5);

    context.SaveChanges();
} catch (Exception e){
   throw e;
}
return OK;
```

1. エラー：一意制約違反 ①
   ３件目まで正常に追加・コミットされる。  
   ４件目で一意制約違反エラー。

| id  | title      | message |
| --- | ---------- | ------- |
| 1   | タイトル 1 | 内容    |
| 2   | タイトル 2 | 内容    |
| 3   | タイトル 3 | 内容    |

2. エラー：一意制約違反 ②  
   ４件目で一意制約違反エラー。自動でロールバックされる。

| id  | title | message |
| --- | ----- | ------- |

3. 正常終了：４件追加  
   ４件目の一意制約違反は無視され、５件目も追加される。  
   結果、重複行を除いたすべてが DB に保存される。

| id  | title      | message |
| --- | ---------- | ------- |
| 1   | タイトル 1 | 内容    |
| 2   | タイトル 2 | 内容    |
| 3   | タイトル 3 | 内容    |
| 5   | タイトル 5 | 内容    |

4. 正常終了：５件追加  
   ４件目で一意制約違反になるが、自動採番を設定しているので自動で追加してくれる。  
   後続処理も同様に実行される。

| id  | title      | message |
| --- | ---------- | ------- |
| 1   | タイトル 1 | 内容    |
| 2   | タイトル 2 | 内容    |
| 3   | タイトル 3 | 内容    |
| 4   | タイトル 1 | 内容    |
| 5   | タイトル 5 | 内容    |

<details><summary>こたえ</summary>

3. 正常終了：４件追加

> 1 件目のデータが 4 件目で上書きされ、保存される。

</details>

--- 更新系

### No.2 Add 後の操作

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
    Information info1 = new() { id = 1, title = "タイトル", message = "内容" };
    context.Informations.Add(info);
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

| id  | title | message |
| --- | ----- | ------- |

2. システムエラー：データ追加あり  
   データの追加は成功し自動コミットされるが、データの更新操作を行おうとして、エラーになるはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

3. 正常終了：データ変更なし  
   データの追加は成功し自動コミットされる。
   タイトルの更新は Update メソッドを呼び出していないので、データベースには反映されないはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

4. 正常終了：データ追加・更新あり  
   EF Core がうまいことやってくれるので、データ追加も、タイトル更新も実行されるはず！

| id  | title        | message |
| --- | ------------ | ------- |
| 101 | EF Core の謎 |         |

<details>
<summary>こたえ</summary>

4. 正常終了：データ追加・更新あり

> EF Core の管理下にある Entity は変更追跡される。

</details>

### No.3 データ追加した後で Entity をいじる

---

### No.4 データ追跡管理化にない状態で Update する/更新後に Entity をいじる

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
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
   データを読み込んでいない(Select していない)のでデータ追跡ができず、エラーになるはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

2. システムエラー：データ更新あり  
   Update までの操作は自動コミットにより反映され、title の変数操作が追跡管理外になるのでエラーになるはず！

| id  | title  | message |
| --- | ------ | ------- |
| 101 | 更新後 |         |

3. 正常終了：データ更新あり  
   Update 呼び出し直前までが反映される。  
   その後の操作は Update メソッドを呼び出していないので、データベースには反映されないはず！

| id  | title  | message |
| --- | ------ | ------- |
| 101 | 更新後 |         |

4. 正常終了：データ更新 × ２あり  
   EF Core がうまいことやってくれるので、最後のタイトルも更新されるはず！

| id  | title        | message |
| --- | ------------ | ------- |
| 101 | EF Core の謎 |         |

<details>
<summary>こたえ</summary>

4. 正常終了：データ更新 × ２あり

> Update 命令で管理下になり、Entity は変更追跡され、最終的な結果が永続化される。

</details>

---

### No.5 更新対象のデータがない

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
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

| id  | title | message |
| --- | ----- | ------- |

2. 正常終了：データ変更なし  
   update information set title="タイトル" where id = id、update information set title="EF Core の謎" where id = id という SQL が流れるはずなので、一致するデータがなく、何もしなかった時と同じになるはず！

| id  | title | message |
| --- | ----- | ------- |

3. 正常終了：データ追加あり  
   データベースにデータが存在しないため自動で作成してくれるはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

4. システムエラー：データ追加・更新あり  
   データベースにデータが存在しないため自動で作成して、さらに更新もやってくれるはず！

| id  | title        | message |
| --- | ------------ | ------- |
| 101 | EF Core の謎 |         |

<details>
<summary>こたえ</summary>

1. システムエラー：データ変更なし

> 更新対象のデータが見つからないので、エラーになる。

</details>

---

### No.6 更新して追加

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
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
   Update の時点でデータがデータベースにないので、エラーになるはず！

| id  | title | message |
| --- | ----- | ------- |

2. システムエラー：データ変更なし  
   Add する際に、Update 命令が先に実行されているため、エラーにしてくれるはず！

| id  | title | message |
| --- | ----- | ------- |

3. システムエラー：データ追加あり  
   Update 命令は空振りし、Add が実行されるはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

4. 正常終了：データ追加・更新あり  
   Update 命令は空振りし、Add が実行され、さらに title も更新されるはず！

| id  | title        | message |
| --- | ------------ | ------- |
| 101 | EF Core の謎 |         |

<details>
<summary>こたえ</summary>

4. 正常終了：データ追加・更新あり

> 一番最初の Update はなかったことにされる。

</details>

---

### No.7 追加して更新

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
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

1. システムエラー：データ変更なし  
   データを追加を依頼した後に、データの更新を行おうとしているので（２つの異なる操作）、エラーになるはず！

| id  | title | message |
| --- | ----- | ------- |

2. システムエラー：データ追加あり  
   データの追加は成功し自動コミットされるが、データの更新操作を行おうとして、エラーになるはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

3. 正常終了：データ変更なし  
   データの追加は成功し自動コミットされる。
   タイトルの更新は Update メソッドを呼び出していないので、データベースには反映されないはず！

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

4. 正常終了：データ追加・更新あり  
   EF Core がうまいことやってくれるので、データを追加したあとで更新してくれるはず！

| id  | title        | message |
| --- | ------------ | ------- |
| 101 | EF Core の謎 |         |

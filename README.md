# Entity Framework (EF) Core 6 の謎

## 前提条件

使用するテーブルは以下の通り

- テーブル　 Information

| 物理名  | タイプ | NOT NULL | Default  | 備考                              |
| ------- | ------ | -------- | -------- | --------------------------------- |
| id      | num    | YES      | -        | プライマリキー(ユニーク/自動採番) |
| title   | text   | NO       | -        | タイトル                          |
| message | text   | NO       | '未設定' | 内容                              |

## 基礎知識

EF Core における基本的なデータベース操作命令の実装と動きは以下の通り

- 追加

```csharp
DBContext context = GetDbContext();
try {
    var info = new Information { id = 1, title = "タイトル", message = "内容" };
    context.Informations.Add(info);
    context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

実行される SQL

```SQL
INSERT INTO information (id, title, message)
      VALUES (@p0, @p1, @p2)
```

- 更新

```csharp
DBContext context = GetDbContext();
try {
   var info = context.Informations.Find(1);
   info.title = "タイトル(更新後)";
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

実行される SQL

```SQL
UPDATE information SET title = @p0
WHERE id = @p1;
```

- 削除

```csharp
DBContext context = GetDbContext();
try {
    var info = context.Informations.Find(1);
    context.Informations.Remove(info);
    context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

実行される SQL

```SQL
DELETE FROM information
WHERE id = @p0;
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
    var info = new Information { id = 1, title = "タイトル", message = "内容" };
    context.Informations.Add(info);
} catch (Exception e) {
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
   値を変更する操作(追加)しているのに、セーブ命令を実行していないので教えてくれる。

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
    var info = new Information { id = 1, title = "タイトル" };
    context.Informations.Add(info);
    context.SaveChanges();
} catch (Exception e) {
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
| 1   | タイトル |         |

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
   var info = new Information { id = 1, title = "タイトル", message = "内容" };
   context.Informations.Add(info);

   info.id = 2;
   context.Informations.Add(info);

   info.id = 3;
   context.Informations.Add(info);

   context.SaveChanges();
} catch (Exception e) {
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
    var info1 = new Information { id = 1, title = "タイトル1", message = "内容" };
    var info2 = new Information { id = 2, title = "タイトル2", message = "内容" };
    var info3 = new Information { id = 3, title = "タイトル3", message = "内容" };
    var info5 = new Information { id = 5, title = "タイトル5", message = "内容" };

    context.Informations.Add(info1);
    context.Informations.Add(info2);
    context.Informations.Add(info3);
    context.Informations.Add(info1); // ← 注目！
    context.Informations.Add(info5);

    context.SaveChanges();
} catch (Exception e) {
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

---

### No.5 Add 後の操作

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "タイトル", message = "内容" };
   context.Informations.Add(info);
   // -- (中略) --
   info1.title = "EF Core の謎";
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：データ追加  
   データの追加は成功し自動コミットされる。  
   タイトルの更新は変数の書き換えのみ。  
   Update メソッドを呼び出していないので、データベースには反映されない。

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

2. 正常終了：データ追加・更新  
   データが追加され、さらにタイトルの変更も意図を読んで行ってくれる。

| id  | title        | message |
| --- | ------------ | ------- |
| 101 | EF Core の謎 |         |

3. エラー：データ追加  
   データの追加は成功し自動コミットされるが、その後の更新操作がエラーになる。

| id  | title    | message |
| --- | -------- | ------- |
| 101 | タイトル |         |

4.エラー：データ変更なし  
データの追加は成功し、その後の変更操作がエラーになり、ロールバックされる。

| id  | title | message |
| --- | ----- | ------- |

<details>
<summary>こたえ</summary>

2. 正常終了：データ追加・更新

> Update メソッドを呼び出していなくても、変更される。

</details>

---

### No.6 Select しないで Update

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが１件
>
>   | id  | title  | message  |
>   | --- | ------ | -------- |
>   | 1   | 本日は | 晴天なり |
>
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "明日は" , message = "台風です"};
   context.Informations.Update(info);
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. エラー：データ変更なし  
   更新対象を select していないので、更新に失敗する。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 晴天なり |

2. 正常終了：データ更新あり  
   データが更新される。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 明日は | 台風です |

<details>
<summary>こたえ</summary>

2. 正常終了：データ更新あり

> select していなくても更新は可能。

</details>

---

### No.7 Select しないで Update(対象データなし)

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "明日は" , message = "台風です"};
   context.Informations.Update(info);
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：なにもおこらない  
   Update が実行されるだけなので、エラーは起きない。

| id  | title | message |
| --- | ----- | ------- |

2. エラー：更新対象がないエラー  
   更新対象がないので、更新に失敗する。

| id  | title | message |
| --- | ----- | ------- |

<details>
<summary>こたえ</summary>

2. エラー：更新対象がないエラー

> データが無ければ更新が失敗扱いになる。

</details>

---

### No.8 Select しないで Update 後に項目変更

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが１件
>
>   | id  | title  | message  |
>   | --- | ------ | -------- |
>   | 1   | 本日は | 晴天なり |
>
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "明日は" , message = "台風です"};
   context.Informations.Update(info);
   // -- (中略) --
   info.title = "来年は";
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：Update のみ  
   Update は正常に行われ、その後は追加したわけではないので反映されない。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 明日は | 台風です |

2. 正常終了：タイトルも反映される  
   Update もタイトルの変更も成功する。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 来年は | 台風です |

3. エラー：Update のみ  
   Update は正常に行われてコミットし、管理外の操作はエラーになる。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 明日は | 台風です |

4. エラー：データ変更なし  
   管理外の操作はエラーになる。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 晴天なり |

<details>
<summary>こたえ</summary>

2. 正常終了：タイトルも反映される

> Update 後にも管理対象になってしまうため、title の変更も行われる。

</details>

---

### No.9 明示的な Update

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが１件
>
>   | id  | title  | message  |
>   | --- | ------ | -------- |
>   | 1   | 本日は | 晴天なり |
>
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = context.Informations.Find(1);
   info.title = "明日は";
   context.Informations.Update(info); // 明示的にUpdate
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. システムエラー：データ変更なし  
   余計な操作なので、更新に失敗する。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 晴天なり |

2. 正常終了：データ更新あり ①  
   通常時と同じく、タイトルが更新される。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 明日は | 晴天なり |

3. 正常終了：データ更新あり ②  
   タイトルとメッセージ（未指定なので null）が更新される。

| id  | title  | message |
| --- | ------ | ------- |
| 1   | 明日は |         |

4. 正常終了：データ更新あり ③  
   タイトルとメッセージ(未指定なので DB デフォルト値の'未設定')が更新される。

| id  | title  | message |
| --- | ------ | ------- |
| 1   | 明日は | 未設定  |

<details>
<summary>こたえ</summary>

3. 正常終了：データ更新あり ②

> Update が明示的に呼ばれた場合は、フルアップデートになる。

</details>

---

### No.10 更新後に追加

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "明日は" , message = "台風です"};
   context.Informations.Update(info);
   // -- (中略) --
   info.title = "本日は";
   context.Informations.Add(info);
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. エラー：更新対象がないエラー  
   Update 実行時に更新対象がないので、更新に失敗する。

| id  | title | message |
| --- | ----- | ------- |

2. 正常終了：データ追加  
   データが追加される。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 台風です |

<details>
<summary>こたえ</summary>

2. 正常終了：データ追加

> Update 文は無視される。

</details>

---

### No.11 データ追加後に検索して更新

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "本日は" , message = "晴天なり"};
   context.Informations.Add(info);
   // -- (中略) --
   var infoT = context.Informations.Find(1);
   infoT.title = "明日は";
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：データ追加
   データが追加されるが、更新はされない。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 晴天なり |

2. 正常終了：データ追加・更新  
   データが追加されたのち、message が更新される。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 明日は | 晴天なり |

3. エラー：処理に失敗  
   連続操作しようとすると Update 処理がうまくできず、エラーになってしまう。

| id  | title | message |
| --- | ----- | ------- |

<details>
<summary>こたえ</summary>

2. 正常終了：データ追加・更新

> 追加して、更新される。

</details>

---

### No.12 データ追加後に更新(Select しない Update)

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { id = 1, title = "本日は" , message = "晴天なり"};
   context.Informations.Add(info);
   // -- (中略) --
   var infoUpdate = new Information { id = 1, message = "台風です"};
   context.Informations.Update(infoUpdate);
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：データ追加・更新  
   データが追加されたのち、message が更新される。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 台風です |

2. 正常終了：データ追加・更新  
   データが追加されたのち、更新される。  
   message は更新されるが、title が指定されていないので、title が消える。

| id  | title | message  |
| --- | ----- | -------- |
| 1   |       | 台風です |

3. エラー：処理に失敗  
   Update 処理がうまくできず、エラーになってしまう。

| id  | title | message |
| --- | ----- | ------- |

<details>
<summary>こたえ</summary>

3. エラー：処理に失敗

> DB に格納されていない状態で、Select しないで Update しようとするとエラーになる。

</details>

---

### No.12 Select しないで Remove

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが１件
>
>   | id  | title  | message  |
>   | --- | ------ | -------- |
>   | 1   | 本日は | 晴天なり |
>
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { Id = 1 };
   context.Informations.Remove(info);
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. エラー：削除されない  
   更新対象を select していないので、削除に失敗する。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 晴天なり |

2. 正常終了：削除される  
   データが削除される。

| id  | title | message |
| --- | ----- | ------- |

<details>
<summary>こたえ</summary>

2. 正常終了：削除される

> select していなくても削除は可能。

</details>

---

### No.13 Select しないで Remove(対象データなし)

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが存在しない
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = new Information { Id = 1 };
   context.Informations.Remove(info);
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：なにもおこらない  
   Delete が実行されるだけなので、エラーは起きない。

| id  | title | message |
| --- | ----- | ------- |

2. エラー：削除対象がないエラー  
   削除対象がないので、削除に失敗する。

| id  | title | message |
| --- | ----- | ------- |

<details>
<summary>こたえ</summary>

2. エラー：削除対象がないエラー

> 削除対象がない場合にはエラーになる。

</details>

---

### No.14 削除 × ２

以下のコードを実行した結果、どうなるでしょうか？

> [!NOTE]
>
> - データベース内にはデータが１件
>
>   | id  | title  | message  |
>   | --- | ------ | -------- |
>   | 1   | 本日は | 晴天なり |
>
> - 任意でトランザクション開始しない(お任せ自動コミットモード)

```csharp
DBContext context = GetDbContext();
try {
   var info = context.Informations.Find(1);
   context.Informations.Remove(info);
   // -- (中略) --
   context.Informations.Remove(info); // 大事なことなので２回実行する
   context.SaveChanges();
} catch (Exception e) {
   throw e;
}
return OK;
```

1. 正常終了：データは削除される  
   1 回目で削除され、2 回目はデータがないのでスルーされる。

| id  | title | message |
| --- | ----- | ------- |

2. エラー：削除対象がないエラー  
   2 回目の削除でエラーになる。  
   1 回目の削除後にコミットされているので、データは消えている。

| id  | title | message |
| --- | ----- | ------- |

3. エラー：削除対象がないエラー(ロールバック)  
   2 回目の削除でエラーになる。  
   1 回目の処理もロールバックされる。

| id  | title  | message  |
| --- | ------ | -------- |
| 1   | 本日は | 晴天なり |

<details>
<summary>こたえ</summary>

1. 正常終了：データは削除される

> 2 回呼んでもエラーにはならない。

</details>

---

# データモデリング3-1

## 要件
[Confluence](https://www.atlassian.com/ja/software/confluence) や [esa](https://esa.io/) や [Kibela](https://kibe.la/) のようなドキュメント管理システムのデータベースを設計してスケッチを作成してください。

以下の機能を備えているものとします

- ドキュメント
    - いつ、誰が、どんなテキスト情報を保存したのか管理する
        過去ログはどう残す？
        ドキュメントにバージョンを残す？
    - ドキュメントは必ず何らかのディレクトリに属する
- ディレクトリ
    - 一つ以上のドキュメントを含む階層構造
    - ディレクトリは無制限にサブディレクトリを持つことができる
    - ディレクトリ構造は柔軟に変更可能。ディレクトリが移動してサブディレクトリになることもあり得る
- ユーザ
    - ドキュメントをCRUD（作成、参照、更新、削除）できる
    - ディレクトリをCRUDできる

<br>
<br>

## 実装内容
### テーブル作成
- ユーザー
- ディレクトリ
- ドキュメント

### 懸念点として
- ディレクトリテーブルを**自己結合**する構成としているが、その場合ディレクトリの階層が深くなるにつれてJOINの回数も多くなるのでパフォーマンス的に悪い
    - 自己結合せずにParentChildrenDirectoryのような中間テーブルを作成する方針もあり？？
    - もしくは閉包テーブル

        https://qiita.com/kondo0602/items/de92eaf6d2bd7f57a74c

        https://note.com/standenglish/n/n0f11205f154e

        https://www.youtube.com/watch?v=wwDITz08bCQ

- 閉包テーブルが良さそう
    https://zenn.dev/rescuenow/articles/c7d7291f2deed8

### DDL
```sql
CREATE TABLE user (
    id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE directory (
    id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE document (
    id INT PRIMARY KEY,
    directory_id INT,
    user_id INT,
    text TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (directory_id) REFERENCES directory(id),
    FOREIGN KEY (user_id) REFERENCES user(id)
);

CREATE TABLE directory_closure (
    ancestor INT,
    descendant INT,
    depth INT,
    PRIMARY KEY (ancestor, descendant),
    FOREIGN KEY (ancestor) REFERENCES directory(id),
    FOREIGN KEY (descendant) REFERENCES directory(id)
);
```


### DML
```sql
-- user
INSERT INTO user (id, name) VALUES (1, 'user1');
INSERT INTO user (id, name) VALUES (2, 'user2');
INSERT INTO user (id, name) VALUES (3, 'user3');

-- directory
INSERT INTO directory (id, name) VALUES (1, 'root');
INSERT INTO directory (id, name) VALUES (2, 'sub1');
INSERT INTO directory (id, name) VALUES (3, 'sub2');
INSERT INTO directory (id, name) VALUES (4, 'sub1-1');

-- document
INSERT INTO document (id, directory_id, user_id, text, created_at) VALUES (1, 2, 1, 'text1', '2022-01-01 00:00:00');
INSERT INTO document (id, directory_id, user_id, text, created_at) VALUES (2, 3, 2, 'text2', '2022-01-02 00:00:00');
INSERT INTO document (id, directory_id, user_id, text, created_at) VALUES (3, 4, 3, 'text3', '2022-01-03 00:00:00');

-- directory_closure
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (1, 1, 0);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (1, 2, 1);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (1, 3, 1);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (1, 4, 2);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (2, 2, 0);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (2, 4, 1);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (3, 3, 0);
INSERT INTO directory_closure (ancestor, descendant, depth) VALUES (4, 4, 0);

```

### ユースケースに対するクエリー
- ユーザ1がいてドキュメントをCRUD（作成、参照、更新、削除）する場合
    ```sql
    -- ドキュメントを作成する
    INSERT INTO document (id, directory_id, user_id, text, created_at) VALUES (1, 1, 1, 'text1', '2022-01-01 00:00:00');

    -- ドキュメントを参照する
    SELECT * FROM document WHERE user_id = 1;

    -- ドキュメント1を更新する
    UPDATE document SET text = 'text1-1' WHERE id = 1 AND user_id = 1;

    -- ドキュメント1を削除する
    DELETE FROM document WHERE id = 1 AND user_id = 1;
    ```

- ユーザ1がいてディレクトリをCRUDする場合
    ```sql
    -- ディレクトリを作成する
    INSERT INTO directory (id, parent_id, name) VALUES (5, 1, 'sub3');

    -- ディレクトリを参照する
    SELECT * FROM directory WHERE parent_id = 1;

    -- ディレクトリを更新する
    UPDATE directory SET name = 'sub3-1' WHERE id = 5;

    -- ディレクトリを削除する
    DELETE FROM directory WHERE id = 5;
    ```


- ユーザー1がいて、そのユーザーがディレクトリ1を作成して、そのディレクトリにドキュメント1を作成した場合
    ```sql
    INSERT INTO user (id, name) VALUES (1, 'user1');
    INSERT INTO directory (id, parent_id, name) VALUES (1, NULL, 'root');
    INSERT INTO document (id, directory_id, user_id, text, created_at) VALUES (1, 1, 1, 'text1', '2022-01-01 00:00:00');
    ```

- ドキュメントを作成したユーザーを取得する
    ```sql
    SELECT
        u.name
    FROM
        user AS u
    JOIN
        document AS d
    ON
        u.id = d.user_id;
    ```

- ドキュメントを作成したユーザーとそのドキュメントを取得する
    ```sql
    SELECT
        u.name,
        d.text
    FROM
        user AS u
    JOIN
        document AS d
    ON
        u.id = d.user_id;
    ```


あるディレクトリーの子孫をすべて取得する場合
ディレクトリID : 1 の子孫を取得
ディレクトリに紐づくドキュメントも取得
  https://www.lyricrime.com/posts/todo-01/
  https://zenn.dev/rescuenow/articles/c7d7291f2deed8
```sql

SELECT * 
FROM directory AS dir
INNER JOIN directory_closure AS dc
	ON dir.id = dc.descendant
LEFT JOIN document AS doc
    ON dc.descendant = doc.directory_id
WHERE dc.ancestor = 1;
```

あるディレクトリーの祖先をすべて取得する場合
ディレクトリID : 2 の祖先を取得
ディレクトリに紐づくドキュメントも取得
```sql

SELECT * 
FROM directory AS dir 
INNER JOIN directory_closure AS dc
	ON dir.id = dc.ancestor
LEFT JOIN document AS doc
    ON dc.ancestor = doc.directory_id
WHERE dc.descendant = 2;
```

- ディレクトリーの移動
例えば、ディレクトリID : 2 をディレクトリID : 3 に移動する場合
```sql
UPDATE directory_closure
SET ancestor = 3
WHERE ancestor = 2;
```






```sql
SELECT
    dir.id AS "ルートディレクトリID",
    dir.name AS "ルートディレクトリ名",
    dir2.id AS "ドキュメントが属するディレクトリID",
    dir2.name AS "ドキュメントが属するディレクトリ名",
    depth AS "階層",
    doc.id AS "ドキュメントID",
    doc.text AS "ドキュメント内容",
    doc.user_id AS "ユーザーID",
    doc.created_at AS "ドキュメント作成日時"
FROM
    directory AS dir
LEFT JOIN
    directory_closure AS dc
ON
    dir.id = dc.ancestor
LEFT JOIN
    document AS doc
ON
    dc.descendant = doc.directory_id
LEFT JOIN
    directory AS dir2
ON
    doc.directory_id = dir2.id
WHERE
    dir.id = 1;
```




---

### 閉包テーブルを使用したディレクトリ移動の説明

#### 初期の閉包テーブル

| Ancestor | Descendant |
|----------|------------|
| 1        | 1          |
| 1        | 3          |
| 1        | 4          |
| 1        | 7          |
| 1        | 8          |
| 2        | 2          |
| 2        | 5          |
| 2        | 6          |
| 3        | 3          |
| 3        | 7          |
| 3        | 8          |
| 4        | 4          |
| 5        | 5          |
| 6        | 6          |
| 7        | 7          |
| 7        | 8          |

#### 子孫を自分自身も含める理由
閉包テーブルで自分自身を子孫として含めることは、クエリの単純化とデータの一貫性保持に役立ちます。具体的には：
- **クエリの単純化**: 任意のノードの全子孫を取得する際、そのノード自身も結果に含める必要がある場合、追加のクエリを実行する必要がなくなります。
- **データ整合性**: 閉包テーブルには、自己参照を含むすべてのノードが少なくとも一つのエントリ（自己参照）を持っているため、データの整合性が保たれます。



#### 自分自身を含めない設計で、ノード自身とその子孫を取得する場合

例として、ノードIDが1のすべての子孫を取得するシナリオを考えます。

- **自分自身を含める場合**のクエリ：
  ```sql
  SELECT descendant
  FROM ClosureTable
  WHERE ancestor = 1;
  ```

- **自分自身を含めない場合**のクエリ：
  ```sql
  SELECT descendant
  FROM ClosureTable
  WHERE ancestor = 1 AND descendant != 1;
  ```

  さらに、ノード自身が必要な場合は、以下のように追加のクエリが必要になります：
  ```sql
  SELECT 1;
  ```

  これは単純な例ですが、より複雑なデータ操作では、自分自身を含めないことによるクエリの複雑化が顕著になります。


#### ディレクトリ移動のプロセス

1. **子孫の取得**:
   ```sql
   SELECT descendant
   FROM directory_closure
   WHERE ancestor = 7;
   ```
   結果：7, 8

2. **子孫の関係を削除**:
   ```sql
   DELETE FROM directory_closure
   WHERE descendant IN (7, 8)
   AND ancestor != 7;
   ```

3. **移動先の祖先を取得**:
   ```sql
   SELECT ancestor
   FROM directory_closure
   WHERE descendant = 5;
   ```
   結果：2, 5

4. **新しい関係の追加**:
   ```sql
   INSERT INTO directory_closure
   VALUES (2, 7), (2, 8), (5, 7), (5, 8);
   ```

#### 新しい関係の追加をプログラム的に表現

新しい関係の追加を繰り返し処理で表現することは可能です。以下は、プログラミング言語（例えばPythonを想定）を使用した擬似コードです。

```python
# 新しい親ノードの祖先リスト
new_ancestors = [2, 5]
# 移動する子孫ノード
descendants_to_move = [7, 8]

# 各祖先に対して、すべての子孫ノードに新しい関係を追加
for ancestor in new_ancestors:
    for descendant in descendants_to_move:
        cursor.execute("INSERT INTO directory_closure VALUES (%s, %s)", (ancestor, descendant))
```

### まとめ
自分自身を子孫として含めない閉包テーブル設計は、特定の場合にはデータベースのパフォーマンスを向上させるかもしれませんが、一般的にはクエリの複雑化、再帰的処理の困難さ、データ整合性の問題といったデメリットが伴います。特に階層データを頻繁に操作するアプリケーションでは、自己を含める設計の方が扱いやすくなることが多いです。




## 課題1-1

リマインダーアプリ、[Penpen](https://penpen.netlify.app/)のデータベースを設計して、スケッチを作成してください。

<aside>
💡 その昔プラハの代表が作ったアプリです。現在はサービス停止中の為動きません。
作った[経緯はこちら](https://qiita.com/dowanna6/items/b5d1d0245985a26abf8e)です。
</aside>

以下の機能を備えているものとします
- リマインダー
  - Slackに登録している他のユーザ（複数可）宛にリマインダーを設定できる
  - リマインダーには送信相手、文面、頻度を指定可能
  - 1時間ごとにバッチが動き、配信が必要なリマインダーを指定されたSlackユーザに配信する
- リマインダーの周期
  - 設定可能な周期は、現時点では以下の4種類（もしかすると今後増えるかもしれませんが、未定です！）
    - 毎日
    - X日おき
    - 毎週X曜日
    - 毎月X日

<br>
<br>

## 考察
> Slackに登録している他のユーザ（複数可）宛にリマインダーを設定できる
- どの**ユーザー(users)**対して、どの**リマインダー(reminds)**を割り当てるかを管理するため**remid_userテーブル**の作成

<br>

> リマインダーには送信相手、文面、頻度を指定可能
- **remindsテーブル**でリマインダーに関する情報を管理

<br>

> 1時間ごとにバッチが動き、配信が必要なリマインダーを指定されたSlackユーザに配信する
- リマインドの周期に関しては**frequenciesテーブル**で管理し、remindsテーブルにfrequency_idを持たせることで、そのリマインドの配信周期を決める

<br>

> 設定可能な周期は、現時点では以下の4種類（もしかすると今後増えるかもしれませんが、未定です！）
- **frequenciesテーブル**に**how_oftenカラム**を設けて周期に関するレコードを登録する。
- 例えば「Xヶ月おき」の様な周期を追加したい場合は、新たにそのレコードをinsertすれば良い

<br>
<br>

## DDL
- remindsテーブル : リマインドの内容を管理
- frequencies : リマインドの頻度を管理
- remind_books : 次回リマインドする時刻を管理
- remind_user : どのユーザーに対して、どのリマインダーを紐づけるのかまた、そのタスクは完了 or 未完了なのかを管理

```sql
-- テーブル: frequencies
-- 毎日リマインドだと frequency_value にNULLが入る
-- 「X日おき」「毎月X日」リマインドだと frequency_value に任意の整数値が入る（例 : ２日おきの場合, 値は 2 ）
-- 「毎週X曜日」リマインドだとfrequency_valueに1〜7（月〜日曜日に対応）の整数値が入る。
CREATE TABLE frequencies (
    id INT PRIMARY KEY AUTO_INCREMENT,
    frequency_type INT NOT NULL,
    frequency_value INT
    -- how_often VARCHAR(255)
);

-- テーブル: reminds
-- ここでの　slack_user_id はリマインドの作成者IDを表す
CREATE TABLE reminds (
    id INT PRIMARY KEY AUTO_INCREMENT,
    frequency_id INT,
    slack_user_id INT,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (frequency_id) REFERENCES frequencies(id)
    -- FOREIGN KEY (slack_user_id) REFERENCES slack_users(id)
);

-- テーブル: remind_user
CREATE TABLE remind_user (
    remid_id INT,
    slack_user_id INT,
    status VARCHAR(10) NOT NULL CHECK (status IN ('完了', '未完')),
    PRIMARY KEY (remid_id, slack_user_id),
    FOREIGN KEY (remid_id) REFERENCES reminds(id)
    -- FOREIGN KEY (slack_user_id) REFERENCES slack_users(id)
);

-- テーブル: next_reminders
CREATE TABLE remind_books (
    id INT PRIMARY KEY AUTO_INCREMENT,
    remind_id INT,
    next_run TIMESTAMP,
    FOREIGN KEY (remind_id) REFERENCES reminds(id)
);

```

<br>
<br>

### DML
```sql
-- テーブル: frequencies にサンプルデータを挿入
-- frequency_typeで頻度の種類を決める
INSERT INTO frequencies (frequency_type, frequency_value) VALUES (1, NULL); -- 毎日
INSERT INTO frequencies (frequency_type, frequency_value) VALUES (2, 2);    -- 2日おき
INSERT INTO frequencies (frequency_type, frequency_value) VALUES (3, 7);    -- 毎週日曜日

-- テーブル: reminds にサンプルデータを挿入
-- slack_user_id はリマインド作成者のID
INSERT INTO reminds (frequency_id, slack_user_id, content) VALUES (1, 101, 'タスクAを完了してください');
INSERT INTO reminds (frequency_id, slack_user_id, content) VALUES (2, 102, 'タスクBを完了してください');
INSERT INTO reminds (frequency_id, slack_user_id, content) VALUES (3, 103, 'タスクCを完了してください');

-- テーブル: remind_user にサンプルデータを挿入
-- リマインダーとユーザーの関連付け、およびタスクのステータス（完了/未完了）
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (1, 201, '未完');
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (1, 202, '完了');
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (2, 201, '未完');

-- テーブル: remind_books にサンプルデータを挿入
-- 次回のリマインド時刻を設定
INSERT INTO remind_books (remind_id, next_run) VALUES (1, '2024-08-15 09:00:00');
INSERT INTO remind_books (remind_id, next_run) VALUES (2, '2024-08-16 09:00:00');
INSERT INTO remind_books (remind_id, next_run) VALUES (3, '2024-08-17 09:00:00');

```

<br>
<br>

## ユースケースを想定したクエリ
```sql
-- ## ユースケース1: 週次報告の準備リマインダー
-- 1. 毎週月曜日にリマインドするための頻度を frequencies テーブルに追加
INSERT INTO frequencies (frequency_type, frequency_value) VALUES (3, 1); -- 毎週月曜日
SET @frequency_id1 = LAST_INSERT_ID();
-- 2. reminds テーブルにリマインドの内容を追加
INSERT INTO reminds (frequency_id, slack_user_id, content) VALUES (@frequency_id1, 101, '週次報告の準備をしてください');
SET @remind_id1 = LAST_INSERT_ID();
-- 3. remind_user テーブルでリマインドをチームメンバー (slack_user_id = 201, 202, 203) に紐付ける
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (@remind_id1, 201, '未完');
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (@remind_id1, 202, '未完');
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (@remind_id1, 203, '未完');
-- 4. 次回リマインド時刻を remind_books テーブルに設定
INSERT INTO remind_books (remind_id, next_run) VALUES (@remind_id1, '2024-08-19 09:00:00'); -- 次の月曜日


-- ## ユースケース2: 特定タスク「テストケースの作成」を2日おきにリマインド
-- 1. 「2日おき」にリマインドするための頻度を frequencies テーブルに追加
INSERT INTO frequencies (frequency_type, frequency_value) VALUES (2, 2); -- 2日おき
SET @frequency_id2 = LAST_INSERT_ID();
-- 2. reminds テーブルにリマインドの内容を追加
INSERT INTO reminds (frequency_id, slack_user_id, content) VALUES (@frequency_id2, 101, 'テストケースの作成を進めてください');
SET @remind_id2 = LAST_INSERT_ID();
-- 3. remind_user テーブルでリマインドを特定のメンバー (slack_user_id = 202) に紐付ける
INSERT INTO remind_user (remid_id, slack_user_id, status) VALUES (@remind_id2, 202, '未完');
-- 4. 次回リマインド時刻を remind_books テーブルに設定
INSERT INTO remind_books (remind_id, next_run) VALUES (@remind_id2, '2024-08-14 09:00:00'); -- 2日後


-- ## ユースケース3: リマインド時刻の確認と更新
-- 次回のリマインド時刻を確認
SELECT rb.next_run
FROM reminds r
JOIN remind_books rb ON r.id = rb.remind_id
WHERE r.slack_user_id = 202;
-- 次回のリマインド時刻を更新（例: 1日後にリマインドを変更）
UPDATE remind_books SET next_run = '2024-08-15 09:00:00' WHERE remind_id = $remind_id;

-- ユーザー202が「テストケースの作成」を完了
UPDATE remind_user SET status = '完了' WHERE remid_id = @remind_id2 AND slack_user_id = 202;
```
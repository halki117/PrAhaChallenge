#　データベースモデリング課題3　
## 課題3-1
サンプルデータを投入してみましょう！
ユースケースをひとつ想定してクエリを書いてみてください。例えば今回の課題であれば「お客さんが会計をする際の最終的な支払金額を計算する」など。

以降「DBモデリングN」といった命名規則の課題は「実際にテーブルを作成する」「サンプルデータを投入する」「ユースケースを想定したクエリを書く」の3点を必ず実施してください。

つまり課題レビューを依頼する際は、以下4つの成果物が存在しなければいけません

- ER図
- 設計したテーブルのDDL
- サンプルデータを投入するDML
- ユースケースを想定したクエリ

| エンティティ   | 属性         | 値の例　|
|-------------|------------|------------|
| 価格        | 価格値(円)   | 100<br>150<br>180 |
| カテゴリ      | カテゴリ名  | 盛り込み<br>地元に生まれた味<br> 好みすし |
| 顧客        | 顧客名       | 山田 太郎<br>山田 花子 |
|             | 電話番号      | 090xxxxxxxx |
| 店舗        | 店舗名       | xxx店 |
| 商品        |  FK 価格ID     | |
|             | FK カテゴリーID  |　 |
|             | 商品名      | はな<br>鮨八方巻<br>えび<br>いくら |
| 注文        | FK 顧客ID      |  |
|             | FK 店舗ID      |  |
|             | 注文日時     | 2023-12-30 |
|             | 会計済み     | true<br>false<br> |
| 注文詳細    | FK 注文ID      | |
|             | FK 商品ID      | |
|             | 個数        | 2<br>4<br> |
|             | 小計金額(円)     | 1000<br>2500<br> |
| オプション    | オプション名      | ワサビぬき |
| 注文詳細_オプション | FK 注文詳細ID      | |
|             | FK オプションID      | |
|             | オプション付与数<br>※注文詳細の個数（皿数）の内⚪︎皿分へのオプション付与数| 5<br>8<br> |
| 注文合計（VIEW） | 注文ID      | |
|             | 合計皿数     | 10<br>20<br> |
|             | 合計金額(円)     | 5000<br>7000<br> |
| 注文合計（VIEW） | 年月別売上個数      | |
|             | 注文年月     | |
|             | 商品名     |  |
|             | 月間売上個数     | |


### テーブル作成クエリ
```SQL
-- Prices Table
CREATE TABLE Prices (
    PriceID INT PRIMARY KEY COMMENT '価格ID',
    PriceValue INT COMMENT '価格値(円)'
) COMMENT '価格テーブル';

-- Categories Table
CREATE TABLE Categories (
    CategoryID INT PRIMARY KEY COMMENT 'カテゴリID',
    CategoryName VARCHAR(255) COMMENT 'カテゴリ名'
) COMMENT 'カテゴリテーブル';

-- Customers Table
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY COMMENT '顧客ID',
    CustomerName VARCHAR(255) COMMENT '顧客名',
    PhoneNumber VARCHAR(20) COMMENT '電話番号'
) COMMENT '顧客テーブル';

-- Stores Table
CREATE TABLE Stores (
    StoreID INT PRIMARY KEY COMMENT '店舗ID',
    StoreName VARCHAR(255) COMMENT '店舗名'
) COMMENT '店舗テーブル';

-- Products Table
CREATE TABLE Products (
    ProductID INT PRIMARY KEY COMMENT '商品ID',
    PriceID INT COMMENT 'FK 価格ID',
    CategoryID INT COMMENT 'FK カテゴリID',
    ProductName VARCHAR(255) COMMENT '商品名',
    FOREIGN KEY (PriceID) REFERENCES Prices(PriceID),
    FOREIGN KEY (CategoryID) REFERENCES Categories(CategoryID)
) COMMENT '商品テーブル';

-- Orders Table
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY COMMENT '注文ID',
    CustomerID INT COMMENT 'FK 顧客ID',
    StoreID INT COMMENT 'FK 店舗ID',
    OrderDateTime DATETIME COMMENT '注文日時',
    IsPaid BOOLEAN COMMENT '会計済み',
    FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID),
    FOREIGN KEY (StoreID) REFERENCES Stores(StoreID)
) COMMENT '注文テーブル';

-- OrderDetails Table
CREATE TABLE OrderDetails (
    OrderDetailID INT PRIMARY KEY COMMENT '注文詳細ID',
    OrderID INT COMMENT 'FK 注文ID',
    ProductID INT COMMENT 'FK 商品ID',
    Quantity INT COMMENT '個数',
    Subtotal INT COMMENT '小計金額(円)',
    FOREIGN KEY (OrderID) REFERENCES Orders(OrderID),
    FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
) COMMENT '注文詳細テーブル';

-- Options Table
CREATE TABLE Options (
    OptionID INT PRIMARY KEY COMMENT 'オプションID',
    OptionName VARCHAR(255) COMMENT 'オプション名'
) COMMENT 'オプションテーブル';

-- OrderDetailOptions Table
CREATE TABLE OrderDetailOptions (
    OrderDetailOptionID INT PRIMARY KEY COMMENT '注文詳細_オプションID',
    OrderDetailID INT COMMENT 'FK 注文詳細ID',
    OptionID INT COMMENT 'FK オプションID',
    OptionCount INT COMMENT 'オプション付与数'
) COMMENT '注文詳細_オプションテーブル';

-- 注文合計VIEW
CREATE VIEW OrderTotals AS
SELECT
    od.OrderID,
    SUM(od.Quantity) AS TotalDishes,
    SUM(od.Subtotal) AS TotalAmount
FROM
    OrderDetails od
GROUP BY
    od.OrderID;

-- 月間個数
CREATE VIEW MonthlySales AS
SELECT 
    DATE_FORMAT(o.OrderDateTime, '%Y-%m') AS OrderMonth,
    p.ProductName,
    SUM(od.Quantity) AS MonthlySalesCount
FROM 
    OrderDetails od
JOIN 
    Products p ON od.ProductID = p.ProductID
JOIN 
    Orders o ON od.OrderID = o.OrderID
JOIN
    Categories c ON p.CategoryID = c.CategoryID
WHERE
    c.CategoryName = '好みすし'
GROUP BY 
    OrderMonth, p.ProductID;     
```


### サンプルデータ作成
ユースケース基づいたクエリを作成する
- ユースケース
    - 顧客「山田太郎」が「東京店」で2024年1月1日の12時に「玉子」を2皿と「えび」を1皿注文し、その注文はすでに支払い済みです。
        - 玉子 : わさび抜き
        - えび : シャリ大
    - 顧客「山田花子」が「大阪店」で2024年1月2日の15時30分に「みさき」を1皿注文しましたが、まだ支払いはされていません。
    - 2024年1月3日の18時45分に「山田太郎」が再び「東京店」で「はな」を1皿注文し、その注文は支払い済みです。

- クエリ
```SQL
-- Inserting data into Prices
INSERT INTO Prices (PriceID, PriceValue) VALUES
(1, 100),
(2, 150),
(3, 180),
(4, 220),
(5, 260),
(6, 8650),
(7, 1940),
(8, 1280);

-- Inserting data into Categories
INSERT INTO Categories (CategoryID, CategoryName) VALUES
(1, '好みすし'),
(2, '盛り込み'),
(3, 'にぎり'),
(4, '鮨やの丼&おすすめ'),
(5, '地元に生まれた味');

-- Inserting data into Customers
INSERT INTO Customers (CustomerID, CustomerName, PhoneNumber) VALUES
(1, '山田 太郎', '09012345678'),
(2, '山田 花子', '09087654321');

-- Inserting data into Stores
INSERT INTO Stores (StoreID, StoreName) VALUES
(1, '東京店'),
(2, '大阪店');

-- Inserting data into Products
-- Note: Assuming PriceID and CategoryID as foreign keys
INSERT INTO Products (ProductID, PriceID, CategoryID, ProductName) VALUES
(1, 1, 1, '玉子'),
(2, 2, 1, 'ゆでゲソ'),
(3, 3, 1, 'えび'),
(4, 4, 1, '生サーモン'),
(5, 5, 1, 'あじ'),
(6, 6, 2, 'はな'),
(7, 7, 3, 'みさき'),
(8, 8, 4, '海鮮ちらし'),
(9, 8, 5, '鮨八方巻');

-- Orders Table
INSERT INTO Orders (OrderID, CustomerID, StoreID, OrderDateTime, IsPaid) VALUES
(1, 1, 1, '2024-01-01 12:00:00', TRUE),
(2, 2, 2, '2024-01-02 15:30:00', FALSE),
(3, 1, 1, '2024-01-03 18:45:00', TRUE);

-- OrderDetails Table
INSERT INTO OrderDetails (OrderDetailID, OrderID, ProductID, Quantity, Subtotal) VALUES
(1, 1, 1, 2, 200),  -- 玉子 2皿
(2, 1, 3, 1, 180),  -- えび 1皿
(3, 2, 7, 1, 1940), -- みさき 1皿
(4, 3, 6, 1, 8650); -- はな 1皿

-- Options Table
INSERT INTO Options (OptionID, OptionName) VALUES
(1, 'ワサビ抜き'),
(2, 'シャリ大');

-- OrderDetailOptions Table
INSERT INTO OrderDetailOptions (OrderDetailOptionID, OrderDetailID, OptionID, OptionCount) VALUES
(1, 1, 1, 2),  -- 注文詳細1 (玉子) にワサビ抜き 2皿分
(2, 2, 2, 1);  -- 注文詳細2 (エビ) シャリ大 1皿分

```


<br>
<br>

## docker環境構築
  - 参考 : https://zenn.dev/peishim/articles/f7a76ae6c253e4
  - M1macの場合compose.ymlに `platform: linux/x86_64` オプションを記載しないとビルドの際エラーが発生する
    ```
    % docker-compose up -d

    no matching manifest for linux/arm64/v8 in the manifest list entries
    ```
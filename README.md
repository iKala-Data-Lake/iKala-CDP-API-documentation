iKala CDP API
===

API documentation: https://ikala-data-lake.github.io/iKala-CDP-API-documentation/

## 流程簡介
**user_profiles 與 tagging 與 orders 上傳流程簡介**：一開始需要先用 Create application token API 取得 token，才可以用這個token來操作其他 API，接著用 Create data ingestion API 建立一個「CSV上傳任務」，API的body有一個參數 type，要填 `user_profiles` 或 `tagging`或 `orders` 來決定要上傳的資料類型，建立 ingestion 後會取得 put_url，將 CSV 檔案透過 put 的方式上傳到該 put_url，當上傳完成後再呼叫  Notify that data is ready for process API 通知伺服器將 CSV 檔案的內容寫入 CDP，最後透過 Get details of data ingestion API 了解 CSV 寫入 CDP 的進度，當 status 為 `succeeded` 時，代表整個上傳作業完成。

## API 列表
|API|URL|Description|
|-|-|-|
| Create application token| POST `/v1/app/token`| 使用 client_id 與 client_secret 取得 token，夾帶在其他 API 的 header 中來通過認證|
| Create data ingestion| POST `/v1/app/user_data/ingestions` |建立一個上傳任務，API的body有一個參數 type ，要填`user_profiles`或`tagging`來決定要上傳的資料類型，接著您必須自行完成上傳 CSV 的步驟，將檔案上傳到 `pur_url`|
| Notify that data is ready for process| POST `/v1/app/user_data/ingestions/{id}/data_ready`| 當 CSV 上傳完成後，使用此API主動通知伺服器將 CSV 檔案內容傳入 CDP|
| Get details of data ingestion|GET `/v1/app/user_data/ingestions/{id}`| Ingestion 目前狀態與 meta data，status 值包含 `created`, `uploaded`, `validating`, `validated`, `processing`, `succeeded`, `failed`|

## 如何使用 put 的方式上傳 CSV

在 expired_at（即創建時間加八小時）之前上傳CSV，不然 put_url 會失效，可以使用任何 HTTP client 上傳您的 CSV 並指定 Content-Type。下面是一個 curl 範例：

`curl -X PUT -T file.csv UPLOAD_URL`

如果上傳成功，您將會收到 HTTP 200 response，在 x-goog-stored-content-length header 能知道檔案實際的byte數。

### Tagging CSV field

**⚠注意：第一列的欄位名稱，每一欄都不能省略，且欄位順序不能調換，但是在第二列以後的資料欄位，如果該資料沒該欄位，則該欄位留空**

[check tagging CSV template](./tagging_CSV_template.csv)

\* means required field

|Name|Description|Type|
|-|-|-|
|user_id*|primary key，需要與 GA 埋入的 user_id 一樣|unique-ID|
|tag_name*| 標籤名稱, ex: 3c | STRING (URF-8 encoding), ^[0-9a-z\-_]+$|

### User profiles CSV field

**⚠注意：第一列的欄位名稱，每一欄都不能省略，且欄位順序不能調換，但是在第二列以後的資料欄位，如果該資料沒該欄位，則該欄位留空**

[check user profiles CSV template](./user_profiles_CSV_template.csv)

\* means required field

|Name|Description|Type|
|-|-|-|
|user_id*|primary key，需要與 GA 埋入的 user_id 一樣|unique-ID|
|email*|電子郵件|String|
|phone_number*|電話號碼 (+886912345678)|String ([E.164](https://en.wikipedia.org/wiki/E.164), 國碼加上電話號碼)|
|gender| 性別<br> - 保留 (rather_not_say)<br> - 男 (male)<br> - 女 (female) |String { rather_not_say \| male \| female }|
|status| 帳號狀態<br> - 未驗證 (pending_activation)<br> - 註冊 (active)<br> - 註銷 (deleted)<br> - 鎖定中 (disabled) |String { pending_activation \| active \| deleted \| disabled }|
|register_timestamp|註冊時間|Timestamp(RFC3339)|
|birthday|YYYY-mm-dd|Date|
|is_dm|是否接收型錄|Boolean|
|is_edm|是否接收電子型錄|Boolean|
|is_email|是否接收電子報|Boolean|
|is_sms|是否接收簡訊通知|Boolean|
|is_call_in|是否接收外撥|Boolean|
|post_code|居住的郵遞區號(111或11163)|String|
|country|居住的國家|String|
|region|居住的地區|String|
|city|居住的城市|String|
|first_purchase_timestamp|第一次溝買的時間戳記|Timestamp(RFC3339)|
|job_industry|產業別(technology etc)|String|
|job_title|職稱(teacher etc)|String|
|register_medium|註冊媒介，例如：(facebook, brick-and-mortar etc) |String|

### Order CSV field

**⚠注意：第一列的欄位名稱，每一欄都不能省略，且欄位順序不能調換，但是在第二列以後的資料欄位，如果該資料沒該欄位，則該欄位留空**

\* means required field

|Name|Description|Type|Value Definition|
|-|-|-|-|
|order_id*|primary key|UniqueID|MUST be UTF-8 encoded|
|user_id*||UniqueID|MUST be UTF-8 encoded|
|create_timestamp|訂單建立時的時間戳記|Timestamp|https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#timestamp_type|
|status|訂單狀態 <br> - 訂單成立 <br> - 請款 <br> - 扣款 <br> - 付清 <br> - 部分換貨 <br> - 換貨 <br> - 部分退貨 <br> - 部分退貨 <br> - 訂單取消|String|MUST be UTF-8 encoded { 訂單成立 \| 請款 \| 扣款 \| 付清 \|  部分換貨 \|  換貨 \|  部分退貨 \|  部分退貨 \|  訂單取消 } <br>**⚠注意：下一個版本開始會換成英文字串**|
|order_amount|消費金額|Float||
|goods_discount_price|單獨用在商品上的折購金額總和|Float||
|order_discount_price|單獨用在訂單上的折購金額總和。從 OrdersCoupons 進行計算|Float||
|payment_amount|結帳金額|Float||
|currency|假如非跨國企業，則用當地貨幣當預設|String|https://zh.wikipedia.org/wiki/ISO_4217|
|installment|分幾期。不分期即是 0|Integer||
|is_online|是否是線上訂單|Boolean||

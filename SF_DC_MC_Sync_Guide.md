# Hướng dẫn đồng bộ Account → Data Cloud → Marketing Cloud
### Dùng Custom Field (CustomerNo) làm Subscriber Key

---

## Tổng quan kiến trúc

```
Salesforce Org (Account)
        ↓ CRM Connector
Data Cloud (Individual + Party Identification + Contact Point Email)
        ↓ Segment + Activation
Marketing Cloud (Data Extension + Subscriber)
```

**Yêu cầu tiên quyết:**
- Data Cloud license đã active trên Production org
- Marketing Cloud đã kết nối với Data Cloud qua Activation Target
- Salesforce CRM Connector đã được cài đặt trong Data Cloud

---

## PHASE 1 — Tạo Data Stream từ Salesforce CRM

### Bước 1.1 — Tạo Data Stream cho Account

1. Vào **Data Cloud → Data Streams → New**
2. Chọn source: **Salesforce CRM**
3. Chọn object: **Account**
4. Ở màn hình **Account Fields**, chọn các field cần thiết:

| Field | Mục đích |
|---|---|
| `Id` (取引先 ID) | Primary Key |
| `CustomerNo__c` | Custom field làm Subscriber Key |
| `PersonEmail` / `Email__c` | Email để gửi |
| `FirstName` | Tên |
| `LastName` | Họ |
| `LastModifiedDate` | Để sync Incremental |

5. Tick **Batch Ingestion** cho tất cả các field trên
6. **Quan trọng:** Tạo **3 Formula Fields** (xem Bước 1.2)

### Bước 1.2 — Tạo Formula Fields (bắt buộc cho Party Identification)

Click **New Formula Field** và tạo lần lượt 3 fields:

**Formula Field 1 — Party Identification Id**

| Thuộc tính | Giá trị |
|---|---|
| Field Label | `Party Identification Id` |
| Field API Name | `PartyIdentificationId` |
| Formula Return Type | Text |
| Transformation Formula | `sourceField['Id'] + "_CustomerNo"` |

> ⚠️ Dùng tab **Attributes** để chọn field `取引先 ID`, sau đó thêm `+ "_CustomerNo"` — không gõ tay `{Id}`

**Formula Field 2 — Party Identification Name**

| Thuộc tính | Giá trị |
|---|---|
| Field Label | `Party Identification Name` |
| Field API Name | `PartyIdentificationName` |
| Formula Return Type | Text |
| Transformation Formula | `"CustomerNo"` |

**Formula Field 3 — Party Identification Type**

| Thuộc tính | Giá trị |
|---|---|
| Field Label | `Party Identification Type` |
| Field API Name | `PartyIdentificationType` |
| Formula Return Type | Text |
| Transformation Formula | `"CustomerNo"` |

7. Click **Next** → Đặt tên Data Stream → **Save**

---

## PHASE 2 — Cấu hình Data Mapping (DMO)

### Bước 2.1 — Vào Data Mapper

1. Vào **Data Streams → AccountHome**
2. Nhìn panel **Data Mapping** bên phải → click **Review**

### Bước 2.2 — Map Contact Point Email DMO

> ⚠️ Đây là bước hay bị sai nhất — phải dùng Email làm Primary Key, không phải Account Id

| Source (AccountHome) | → | Contact Point Email DMO |
|---|---|---|
| `PersonEmail` (メール) | → | `*Contact Point Email Id` **(Primary Key)** |
| `PersonEmail` (メール) | → | `Email Address` |
| `取引先 ID` (Account Id) | → | `Party` |

### Bước 2.3 — Map Individual DMO

| Source (AccountHome) | → | Individual DMO |
|---|---|---|
| `取引先 ID` | → | `*Individual Id` **(Primary Key)** |
| `FirstName` (名) | → | `First Name` |
| `LastName` (姓) | → | `Last Name` |
| `PersonEmail` | → | `Email` |
| `CustomerNo__c` | → | `Customer No` (Custom field) |

### Bước 2.4 — Thêm Party Identification DMO

1. Click icon **✏️ pencil** góc trên phải panel **Data Model entities**
2. Search `Party Identification` → click **+** → click **Close**
3. Scroll xuống cuối cột phải — **Party Identification** xuất hiện

Map các fields sau:

| Source (AccountHome) | → | Party Identification DMO |
|---|---|---|
| `PartyIdentificationId` (formula) | → | `*Party Identification Id` **(Primary Key)** |
| `取引先 ID` | → | `Party` |
| `CustomerNo__c` | → | `Identification Number` |
| `PartyIdentificationName` (formula) | → | `Identification Name` |
| `PartyIdentificationType` (formula) | → | `Party Identification Type` |

4. Click **Save**

---

## PHASE 3 — Refresh Data Stream

### Bước 3.1 — Chạy Full Refresh

1. Vào **Data Streams → AccountHome**
2. Click dropdown ▼ cạnh nút **Refresh Now**
3. Chọn **Full Refresh**
4. Đợi **Last Run Status = Success** và **Last Processed Records > 0**

### Bước 3.2 — Verify data trong Profile Explorer

1. Vào **Data Cloud → Profile Explorer**
2. Chọn **Unified Individual** → chọn attribute **Email Address**
3. Search email của một Account có `CustomerNo__c`
4. Click **View** → tab **Related**
5. Kiểm tra:
   - `Unified Indv Contact Point Email` có data ✓
   - `Party Identification` có data với `CustomerNo` ✓

---

## PHASE 4 — Chạy Identity Resolution

### Bước 4.1 — Run Identity Resolution

1. Vào **Data Cloud → Identity Resolutions**
2. Click vào ruleset đang active
3. Click **Run Now** hoặc **Publish & Run**
4. Đợi job hoàn thành (5-15 phút)

### Bước 4.2 — Verify Unified Individual

Sau khi Identity Resolution chạy xong, vào **Profile Explorer** search lại — tab **Related** phải hiện:
- `Unified Indv Contact Point Email (1+)` ✓
- `Unified Link Individual (1+)` ✓

---

## PHASE 5 — Tạo Segment

### Bước 5.1 — Tạo Segment mới

1. Vào **Data Cloud → Segments → New**
2. Cấu hình:
   - **Segment Name**: `SEG_Account_CustomerNo`
   - **Segment On**: **Individual** ← bắt buộc, không chọn Account
3. Click **Save**

### Bước 5.2 — Thêm filter theo CustomerNo

1. Cột trái → **Related Attributes → Party Identification (13)**
2. Kéo field **Identification Number** vào canvas
3. Cấu hình filter:

```
Party Identification: Count At Least 1
    Identification Number  Is Equal To  '00008999'
    OR
    Identification Number  Is Equal To  '00001501'

AND

Party Identification: Count At Least 1
    Party Identification Type  Is Equal To  'CustomerNo'
```

4. Click **↻ refresh** để tính lại population
5. Verify số records đúng → click **Save**

> 💡 **Lưu ý:** Nếu muốn filter nhiều CustomerNo, dùng operator **In List** thay vì nhiều điều kiện OR

---

## PHASE 6 — Tạo Activation sang Marketing Cloud

### Bước 6.1 — Tạo Activation mới

1. Vào **Data Cloud → Activations → New Activation**
2. Chọn:
   - **Segment**: `SEG_Account_CustomerNo`
   - **Activation Target**: MC target đã có (ví dụ `SFMC_Production`)
3. Click **Next**

### Bước 6.2 — Chọn Contact Point

1. Click **+ Select** cạnh **Email**
2. **Email Address Path**: giữ nguyên mặc định
   ```
   Contact Point Email.Party > Individual.Individual Id
   ```
3. Click **Next**

### Bước 6.3 — Add Attributes

1. Click **Add Attributes** góc trên phải
2. Chọn từ **Individual DMO**:
   - `Individual Id`
   - `Customer No` ← field làm Subscriber Key
   - `First Name`
   - `Last Name`
3. Đổi **Output Name** của `Customer No` thành `CustomerNo`
4. Click **Save**

### Bước 6.4 — Set Subscriber Key = CustomerNo

> ⚠️ Đây là bước quan trọng — thực hiện được khi Contact Point Email mapping đúng (メール làm Primary Key)

Khi `Contact Point Email Id` được map từ `PersonEmail` (email value), Data Cloud tự động dùng **Party Identification → Identification Number** làm Contact Key → Subscriber Key bên MC sẽ là `CustomerNo`.

### Bước 6.5 — Publish Schedule

Chọn tần suất phù hợp:
- **Publish Now**: test ngay lập tức
- **Hourly/Daily**: production

5. Click **Save & Publish**

---

## PHASE 7 — Verify bên Marketing Cloud

### Bước 7.1 — Trigger Activation

1. Vào **Segments → SEG_Account_CustomerNo → Publish**
2. Quay lại **Activations → ACT_Account_CustomerNo → tab Insights**
3. Đợi **Publish Status = Success**

### Bước 7.2 — Kiểm tra Data Extension trong MC

1. Vào **Marketing Cloud → Email Studio → Data Extensions**
2. Tìm DE tên: `SEG_Account_CustomerNo_ACT_...`
3. Click **Records** — verify:

| SubscriberKey | CustomerNo | EmailAddress |
|---|---|---|
| 00008999 | 00008999 | email@domain.com |
| 00001501 | 00001501 | email2@domain.com |

> ✓ **SubscriberKey = CustomerNo** — đây là kết quả mục tiêu

---

## Lưu ý quan trọng khi implement Production

| # | Điều cần nhớ |
|---|---|
| 1 | **Subscriber Key không thể thay đổi** sau khi Contact đã tồn tại trong MC All Subscribers |
| 2 | Map `PersonEmail` làm `Contact Point Email Id` **(Primary Key)** — không phải Account Id |
| 3 | Party Identification cần **3 formula fields** trên source DLO |
| 4 | Luôn chọn **Segment On = Individual** khi tạo Segment cho MC Activation |
| 5 | Chạy **Identity Resolution** sau mỗi lần thay đổi Data Mapping |
| 6 | Account không có email sẽ bị **rejected** khi Activation — cần xử lý trước |
| 7 | Guest checkout (không có CustomerNo) sẽ không được activate — cần giải pháp riêng |

---

## Troubleshooting thường gặp

| Vấn đề | Nguyên nhân | Giải pháp |
|---|---|---|
| Segment ra 0 records | Data Stream chưa Refresh hoặc Party Identification chưa có data | Full Refresh → Run Identity Resolution |
| Profile Explorer không tìm thấy | Identity Resolution chưa chạy | Vào Identity Resolutions → Run Now |
| Contact Point Email = 0 trong Related tab | `Contact Point Email Id` map từ Account Id thay vì Email | Sửa mapping: `PersonEmail` → `Contact Point Email Id` |
| Activation History = No data | Segment chưa được Publish | Vào Segment → Publish |
| SubscriberKey = Individual Id thay vì CustomerNo | Contact Point Email mapping sai | Fix mapping trên Production từ đầu |
| Data Stream Failure liên tục | Auto-refresh conflict hoặc mapping thay đổi | Đợi ổn định → Full Refresh lại |
| Party Identification không xuất hiện trong Data Mapper | Chưa tạo Formula Fields | Tạo đủ 3 formula fields trước khi add DMO |

---

*Tài liệu này được tổng hợp từ quá trình implement thực tế trên Salesforce Data Cloud Sandbox.*
*Một số tính năng (Switch Mapping, Delete Stream) bị giới hạn trên Sandbox — implement trên Production sẽ không gặp các hạn chế này.*

# API Contracts — Chi tiết Request/Response

> File này gom toàn bộ contract API (body, param, response mẫu) liên quan pipeline convert.

Các request cần truyền x-client-id vào header
---

## Phần 1 — API Core HCSN cung cấp cho Tool Migrate (Input)

### 1.1. `GET tenant-convert`

Danh sách tenant HCSN đủ điều kiện convert.

**Request:**
```
GET /api/hcsn/tenant-convert?skip=0&take=50
```

**Response:**
```json
{
  "Total": 3000,
  "Items": [
    {
      "TenantID": "guid",
      "BudgetCode": "BC_ROOT_01234",
      "TenantName": "UBND xã A",
      "HCSNID": "guid-hcsn",
      "IOfficeID": "guid-ioffice",
      "BusinessType": "Comune",
      "IsComune": true,
      "ExtraData": {}
    }
  ]
}
```

---

### 1.2. `GET find?Type=...` — bản ghi lệch HCSNID/IOfficeID

**Request:**
```
GET /api/hcsn/find?TenantID={guid}&Type=User
GET /api/hcsn/find?TenantID={guid}&Type=OrganizationUnit
GET /api/hcsn/find?TenantID={guid}&Type=JobPosition
GET /api/hcsn/find?TenantID={guid}&Type=JobTitle
```

**Response:**
```json
{
  "TenantID": "guid",
  "Type": "User",
  "Items": [
    { "HCSNID": "guid-hcsn-1", "IOfficeID": "guid-ioffice-1" },
    { "HCSNID": "guid-hcsn-2", "IOfficeID": "guid-ioffice-2" }
  ]
}
```

---

### 1.3. `GET jobposition/jobtitle`

Danh mục JobPosition/JobTitle chuẩn (kể cả tenant không cài QLCB).

**Request:**
```
GET /api/hcsn/{tenantID}/jobposition-jobtitle
```

**Response:**
```json
{
  "TenantID": "guid",
  "JobPositions": [
    { "JobPositionID": "guid-jp-1", "IOfficeID": "guid-old-jp-1", "JobPositionCode": "code-1", "JobPositionName": "Chủ tịch" }
  ],
  "JobTitles": [
    { "JobTitleID": "guid-jt-1", "IOfficeID": "guid-old-jt-1", "JobPositionCode": "code-2", "JobTitleName": "Chuyên viên" }
  ]
}
```

---

### 1.4. `GET /api/hcsn/{tenantID}/purchased-apps`

Danh sách app đã mua theo từng OU/PBCM. Dùng trong **Bước 2 (After Convert — hcsn_getting_state)**, gọi lazy cho từng tenant đang convert (không gọi trước cho cả batch).

**Request:**
```
GET /api/hcsn/{tenantID}/purchased-apps
```

**Response:**
```json
{
  "TenantID": "guid",
  "PurchasedApps": [
    {
      "OrganizationUnitID": "guid-ou-1",
      "OrganizationUnitName": "UBND xã",
      "BudgetCode": "BC_ROOT_01234",
      "Apps":
      [
        {
           "AppCode": "MMO",
           "HadStandaloneData": false
        },
        {
           "AppCode": "Salagov",
           "HadStandaloneData": false
        }
      ],
      "Users": [
        {
           "UserID": "guid-1",
           "FullName": "Nguyen Van A",
           "IsFirstAdmin": true // có phải là QTHT đầu tiên không
        },
        {
           "UserID": "guid-2",
           "FullName": "Nguyen Van B",
        }
      ] 
    },
    {
      "OrganizationUnitID": "guid-ou-2",
      "OrganizationUnitName": "Phòng kinh tế",
      "BudgetCode": "BC_ROOT_01235",
      "Apps":
      [
        {
           "AppCode": "QLCB",
           "HadStandaloneData": false
        },
        {
           "AppCode": "QLTSv2",
           "HadStandaloneData": false
        }
      ],
      "Users": [
        {
           "UserID": "guid-1",
           "FullName": "Nguyen Van A",
           "IsFirstAdmin": true // có phải là QTHT đầu tiên không
        },
        {
           "UserID": "guid-2",
           "FullName": "Nguyen Van B",
        }
      ] 
    },
  ]
}
```

---

## Phần 2 — API Core MPL trigger ra Sub-App (Output)

### 2.1. Trigger App chuyên môn (App CM)

**Request:**
```
POST {app.ConvertURL}?Rollback=false
```
```json
{
  "TenantID": "guid",
  "BudgetCode": "...",
  "ExtraData": {}
}
```

IGOV có SubTenant (PBCM con dạng OU):
```json
{
  "TenantID": "guid",
  "BudgetCode": "...",
  "SubTenants": [
    { "SubTenantID": "guid-ou-con", "BudgetCode": "...", "ExtraData": {} }
  ],
  "ExtraData": {}
}
```

**Response (sync, < 60s):** `200 OK` kèm kết quả trực tiếp.

**Response (async, quá 60s) — App tự gọi callback:**
```json
POST {core.CallbackURL}/api/hcsn/convert-callback
{
  "TenantID": "guid",
  "AppCode": "QLCB",
  "Success": true,
  "ConvertTime": "2026-07-16T10:00:00Z",
  "Error": null
}
```

**Rollback:** body giống rollback = false
```json
POST {app.ConvertURL}?Rollback=true
{
  "TenantID": "guid",
  "BudgetCode": "..."
  ...
}
```

---

### 2.2. Trigger VPS (dùng AMISFW — Process, TMS...)

**Request:**
```json
POST {vps.ConvertURL}?Rollback=false
{
  "TenantID": "guid",
  "Delta": {
    "Tenant": { "old-guid-1": "new-guid-1" },
    "User": { "old-guid-3": "new-guid-3", "old-guid-4": "new-guid-4" },
    "OrganizationUnit": { "old-guid-5": "old-guid-5" }
  }
}
```
- VPS (Process, TMS) có DB riêng, tách biệt Platform Core, nhưng data tham chiếu TenantID/UserID/OrganizationUnitID... của Core. ID đổi (Pool mode) thì VPS cần biết ID nào → ID nào để replace trong data của chính nó
- `Delta`: key là `EntityType` (`Tenant` | `User` | `OrganizationUnit` | `JobPosition` | `JobTitle`), value là dict `OldID → NewID` — **mỗi entity chứa nhiều cặp**, không giới hạn 1; chỉ liệt kê entity **có thay đổi**
- `NewID == OldID` nếu không conflict (Silo mode luôn rơi vào trường hợp này — giữ nguyên GUID)

**Response (async 202):**
```
202 Accepted
{ "jobID": "job-xxx" }
```

**Poll trạng thái:**
```
GET {vps.ConvertURL}/api/convert/status/{jobID}
→ { Status, Progress, Logs }
```

**Callback khi xong:**
```json
POST {core.CallbackURL}/api/hcsn/convert-callback
{
  "TenantID": "guid",
  "AppCode": "VPSProcess",
  "Success": true,
  "ConvertTime": "2026-07-16T10:05:00Z",
  "ShardConnection": "Server=silo-vps-xxx;Database=amis_vps_silo_xxx;",
  "Error": null
}
```

**Rollback:**
```json
POST {vps.ConvertURL}?Rollback=true
{
  "TenantID": "guid",
  "Delta": {
    "Tenant": { "old-guid-1": "new-guid-1" },
    "User": { "old-guid-3": "new-guid-3", "old-guid-4": "new-guid-4" },
    "OrganizationUnit": { "old-guid-5": "old-guid-5" }
  }
}
```

---

### 2.3. Trigger VPS (KHÔNG dùng AMISFW)

**Request:**
```json
POST {vps.ConvertURL}?Rollback=false
{
  "TenantID": "guid",
  "Delta": {
    "Tenant": { "old-guid-1": "new-guid-1" },
    "User": { "old-guid-3": "new-guid-3", "old-guid-4": "new-guid-4" },
    "OrganizationUnit": { "old-guid-5": "old-guid-5" }
  }
}
```

**Response:** `200 OK { "Success": true }` — không cần backup/restore/callback ShardConnection.

---

## Bảng tổng hợp

| API | Chiều | Bước dùng | Trạng thái |
|---|---|---|---|
| `tenant-convert` | HCSN → Tool | Bước 0 (input UI) | [BLOCK] quá hạn |
| `find?Type=...` | HCSN → Tool | Bước 1 (transform RAM) | [BLOCK] quá hạn |
| `jobposition/jobtitle` | HCSN → Tool | Bước 1 (transform RAM) | [BLOCK] deadline 13/07 |
| `purchased-apps` | HCSN → Tool | Bước 2 (hcsn_getting_state) | [BLOCK] chưa cung cấp |
| Trigger App CM | Tool → App CM | Bước 3 | [BLOCK] chờ App CM |
| Trigger VPS (AMISFW) | Tool → VPS | Bước 3 | [BLOCK] chờ VPS |
| Trigger VPS (non-AMISFW) | Tool → VPS | Bước 3 | [BLOCK] chờ VPS |
| Callback convert-callback | App/VPS → Core | Bước 3 (async) | [BLOCK] chờ tdlam |




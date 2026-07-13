# API Contracts — Chi tiết Request/Response

> File này gom toàn bộ contract API (body, param, response mẫu) liên quan pipeline convert.

---

## Phần 1 — API Core HCSN cung cấp cho Tool Migrate (Input)

### 1.1. `GET convert-candidates`

Danh sách tenant HCSN đủ điều kiện convert.

**Request:**
```
GET /api/hcsn/convert-candidates?skip=0&take=50
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
      "AppCode": "MMO;Salagov;Bumas",
      "HadStandaloneData": false,
    },
    {
      "OrganizationUnitID": "guid-ou-2",
      "OrganizationUnitName": "Phòng kinh tế",
      "BudgetCode": "BC_PBCM_001",
      "AppCode": "MMO;Salagov;Bumas;QLCB",
      "HadStandaloneData": false,
    }
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
    { "SubTenantID": "guid-ou-con", "BudgetCode": "..." }
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

**Rollback:**
```json
POST {app.ConvertURL}?Rollback=true
{
  "TenantID": "guid",
  "BudgetCode": "..."
}
```

---

### 2.2. Trigger VPS (dùng AMISFW — Process, TMS...)

**Request:**
```json
POST {vps.ConvertURL}?Rollback=false
{
  "TenantID": "guid",
  "ConnectionSource": "ioffice-conn-string",
  "ConnectionTarget": "silo-vps-conn-string",
  "Delta": { }
}
```

**Cấu trúc `Delta`:**
```json
"Delta": {
  "IdMap": {
    "old-guid-1": "new-guid-1",
    "old-guid-2": "old-guid-2"
  },
  "Entities": [
    {
      "EntityType": "Tenant",
      "Records": [
        { "OldID": "old-guid-1", "NewID": "new-guid-1" }
      ]
    },
    {
      "EntityType": "User",
      "Records": [
        { "OldID": "old-guid-3", "NewID": "old-guid-3" }
      ]
    }
  ]
}
```
- VPS (Process, TMS) có DB riêng của nó, tách biệt với Platform Core. Khi Core MPL clone xong 1 tenant sang silo, VPS cũng phải tự backup/restore DB của nó cho tenant đó — nhưng data trong DB của VPS lại tham
  chiếu tới TenantID, UserID, OrganizationUnitID... của Core. Nếu các ID này đổi (Pool mode) thì VPS phải biết ID nào đổi thành ID nào để replace lại trong data của chính nó.
- `IdMap`: map phẳng OldID→NewID toàn bộ — VPS dùng để Replace nhanh
- `Entities[].EntityType`: `Tenant` | `User` | `OrganizationUnit` | `JobPosition` | `JobTitle` — chỉ liệt kê entity **có thay đổi**
- `NewID == OldID` nếu không conflict (giữ nguyên GUID)

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
{ "TenantID": "guid" }
```

---

### 2.3. Trigger VPS (KHÔNG dùng AMISFW)

**Request:**
```json
POST {vps.ConvertURL}?Rollback=false
{ "TenantID": "guid" }
```

**Response:** `200 OK { "Success": true }` — không cần backup/restore/callback ShardConnection.

---

## Bảng tổng hợp

| API | Chiều | Bước dùng | Trạng thái |
|---|---|---|---|
| `convert-candidates` | HCSN → Tool | Bước 0 (input UI) | [BLOCK] quá hạn |
| `find?Type=...` | HCSN → Tool | Bước 1 (transform RAM) | [BLOCK] quá hạn |
| `jobposition/jobtitle` | HCSN → Tool | Bước 1 (transform RAM) | [BLOCK] deadline 13/07 |
| `purchased-apps` | HCSN → Tool | Bước 2 (hcsn_getting_state) | [BLOCK] chưa cung cấp |
| Trigger App CM | Tool → App CM | Bước 3 | [BLOCK] chờ App CM |
| Trigger VPS (AMISFW) | Tool → VPS | Bước 3 | [BLOCK] chờ VPS |
| Trigger VPS (non-AMISFW) | Tool → VPS | Bước 3 | [BLOCK] chờ VPS |
| Callback convert-callback | App/VPS → Core | Bước 3 (async) | [BLOCK] chờ tdlam |




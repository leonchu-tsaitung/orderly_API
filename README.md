# Orderly API - 餐廳供應商訂單系統

> **Minimal Version 1.0.0** - 簡化版 API 規範，專注於核心訂單流程

## 🎯 系統範疇

### 核心功能
- **供應商功能**: 報價單管理 (CRUD + listing)、訂單處理 (detail + listing + modify + confirm)
- **餐廳功能**: 報價單查詢 (listing + detail)、訂單管理 (create + modify + detail + listing + cancel)

### 支援角色
- **供應商 (Suppliers)**: 提供商品價格表，處理餐廳訂單
- **餐廳 (Restaurants)**: 查詢價格，建立和管理訂單

---

## 🔑 關鍵業務邏輯

### 1. 價格表驗證機制
```
嚴格驗證流程：
✓ 價格表必須在有效期內 (valid_end_date >= 當前日期)
✓ 訂單商品必須存在於價格表中
✓ 下訂單時的價格必須與價格表完全一致
✗ 價格不符將導致訂單建立失敗 (HTTP 422)
```

### 2. 訂單狀態流轉
```
created → confirmed → completed
   ↓
cancelled (僅限 created 狀態)
```

### 3. 客戶群組機制
- 基於餐廳認證資訊自動識別客戶群組
- 不同客戶群組適用不同的價格表
- 系統自動匹配對應的有效價格表

---

## 📋 完整商業流程案例

### Case 1: 標準訂單流程 - 成功案例

#### 背景
**供應商**: 新鮮蔬果有限公司  
**餐廳**: 美味小館 (VIP客戶群組)  
**商品**: 有機番茄、新鮮高麗菜

#### 流程步驟

**Step 1: 供應商建立價格表**
```http
POST /suppliers/price-lists
```
```json
{
  "customer_group_code": "VIP",
  "valid_end_date": "2024-12-31",
  "price_items": [
    {
      "item_code": "TOMATO_001",
      "item_name": "有機番茄",
      "price": 120.00,
      "unit": "公斤"
    },
    {
      "item_code": "CABBAGE_002", 
      "item_name": "新鮮高麗菜",
      "price": 45.00,
      "unit": "顆"
    }
  ]
}
```

**Step 2: 餐廳查詢可用價格表**
```http
GET /restaurants/price-lists?supplier_id=SUPPLIER_001
```
```json
[
  {
    "price_list_id": "PL_20241201_001",
    "supplier_id": "SUPPLIER_001", 
    "customer_group_code": "VIP",
    "valid_end_date": "2024-12-31",
    "item_count": 2,
    "created_at": "2024-12-01T09:00:00Z"
  }
]
```

**Step 3: 餐廳查看價格表詳情**
```http
GET /restaurants/price-lists/PL_20241201_001
```

**Step 4: 餐廳建立訂單**
```http
POST /restaurants/orders
```
```json
{
  "supplier_id": "SUPPLIER_001",
  "delivery_date": "2024-12-05", 
  "order_items": [
    {
      "item_code": "TOMATO_001",
      "quantity": 10.0,
      "expected_price": 120.00
    },
    {
      "item_code": "CABBAGE_002",
      "quantity": 20.0, 
      "expected_price": 45.00
    }
  ]
}
```

**Step 5: 供應商確認訂單**
```http
POST /suppliers/orders/ORDER_001/confirm
```
```json
{
  "confirmed_items": [
    {
      "item_code": "TOMATO_001",
      "confirmed_price": 120.00,
      "confirmed_quantity": 10.0
    },
    {
      "item_code": "CABBAGE_002", 
      "confirmed_price": 45.00,
      "confirmed_quantity": 20.0
    }
  ]
}
```

**✅ 結果**: 訂單成功建立並確認，總金額 $2,100

---

### Case 2: 價格驗證失敗案例

#### 背景
餐廳使用過時的價格資訊下訂單

#### 問題情境
```http
POST /restaurants/orders
```
```json
{
  "supplier_id": "SUPPLIER_001",
  "delivery_date": "2024-12-05",
  "order_items": [
    {
      "item_code": "TOMATO_001", 
      "quantity": 10.0,
      "expected_price": 100.00  // 舊價格，實際為 120.00
    }
  ]
}
```

#### 系統回應
```http
HTTP 422 Unprocessable Entity
```
```json
{
  "error": {
    "code": "PRICE_MISMATCH",
    "message": "商品價格不符", 
    "details": "商品 TOMATO_001 的預期價格 $100.00 與價格表中的價格 $120.00 不符"
  }
}
```

**❌ 結果**: 訂單建立失敗，餐廳需要使用正確價格重新下訂

---

### Case 3: 訂單取消案例

#### 背景
餐廳在供應商確認前取消訂單

#### 取消流程
```http
DELETE /restaurants/orders/ORDER_002
```
```json
{
  "reason_code": "customer_change_mind",
  "reason_text": "菜單臨時調整，不需要這批蔬菜"
}
```

#### 系統回應
```json
{
  "message": "訂單已成功取消",
  "cancelled_at": "2024-12-02T14:30:00Z",
  "order_id": "ORDER_002", 
  "previous_status": "created"
}
```

**✅ 結果**: 訂單成功取消

---

## 🚀 快速開始

### 1. 認證
所有 API 請求需要在 Header 中包含 Bearer Token:
```http
Authorization: Bearer YOUR_JWT_TOKEN
```

### 2. 基礎 URL
```
生產環境: https://api.orderly.com/v1
測試環境: https://staging-api.orderly.com/v1
```

### 3. 內容格式
```http
Content-Type: application/json
Accept: application/json
```

---

## 📚 API 文檔

完整的 API 規範請參考：
- [OpenAPI 3.1.0 規範](./docs/api-minimal.yaml)
- 支援 Swagger UI 預覽

---

## ⚠️ 重要注意事項

### 價格一致性
- 下訂單時必須提供 `expected_price`
- 價格必須與當前有效價格表完全一致
- 價格不符會導致 422 錯誤

### 訂單狀態限制
- 只有 `created` 狀態的訂單可以取消
- 訂單一旦 `confirmed` 就無法取消
- 取消操作不可逆

### 價格表有效期
- 系統會自動驗證價格表有效期
- 過期價格表無法用於下訂單
- 建議供應商及時更新價格表

---

## 🔧 技術規格

- **API 版本**: OpenAPI 3.1.0
- **認證方式**: JWT Bearer Token
- **回應格式**: JSON
- **錯誤處理**: 標準 HTTP 狀態碼 + 結構化錯誤訊息
- **無分頁機制**: 所有清單端點直接回傳陣列

---

## 📞 支援與回饋

如有技術問題或建議，請聯繫：
- **技術支援**: support@orderly-api.com
- **API 文檔**: 基於 Issue #30 需求設計的精簡版本
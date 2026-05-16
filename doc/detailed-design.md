# 購物網站詳細設計文件 (Detailed Design Document)

## 1. 系統架構設計 (Architecture Design)

為確保「功能獨立性以便單元測試」，本專案前後端架構均採用模組化與分層設計。

### 1.1 後端架構 (Node.js + Express)
採用嚴格的 **三層式架構 (3-Tier Architecture)**，確保業務邏輯與資料存取解耦，高度支援單元測試 (Unit Testing)：
*   **Controller 層 (控制器)**: 負責接收 HTTP Request、驗證傳入參數、呼叫 Service 層，並回傳 HTTP Response。不包含任何業務邏輯。
*   **Service 層 (服務)**: 系統的核心業務邏輯。負責處理資料運算、商業規則驗證，完全獨立於 HTTP 請求上下文。測試時可輕鬆 Mock Repository 層。
*   **Repository / DAO 層 (資料存取)**: 負責直接與 MongoDB (Mongoose) 互動，執行 CRUD 操作。

### 1.2 前端架構 (React)
*   **元件設計 (Component Design)**: 採用 Atomic Design 概念，區分 Presentational Components (純 UI 展示，容易進行 Snapshot Test) 與 Container Components (負責與全域狀態或 API 互動)。
*   **狀態管理 (State Management)**: 
    *   **伺服器狀態**: 使用 `React Query` 進行 API 資料快取、重試與同步。
    *   **客戶端狀態**: 使用 `Zustand` 處理 UI 狀態與購物車狀態。
*   **路由設計 (Routing)**: 使用 `React Router` 配合 Lazy Loading 提升效能。

---

## 2. 資料庫綱要設計 (Database Schema)

### 2.1 User (使用者)
*   `_id`: ObjectId
*   `email`: String (Unique, Indexed)
*   `password`: String (Bcrypt Hashed)
*   `name`: String
*   `role`: Enum ['Customer', 'Admin', 'SuperAdmin'] (權限區分)
*   `createdAt` / `updatedAt`: Timestamp

### 2.2 Product (商品)
*   `_id`: ObjectId
*   `name`: String
*   `description`: String
*   `price`: Number
*   `stock`: Number (庫存量)
*   `category`: String (可建立 Index 以加速查詢)
*   `images`: Array of Strings (URL)
*   `isActive`: Boolean (上/下架狀態)

### 2.3 Order (訂單)
*   `_id`: ObjectId
*   `user`: ObjectId (Ref: User)
*   `items`: Array of Objects
    *   `product`: ObjectId (Ref: Product)
    *   `quantity`: Number
    *   `priceAtPurchase`: Number (鎖定購買時價格)
*   `totalAmount`: Number (原始總金額)
*   `discountApplied`: ObjectId (Ref: Discount, Optional)
*   `finalAmount`: Number (折扣後最終金額)
*   `status`: Enum ['Pending', 'Paid', 'Shipped', 'Completed', 'Cancelled']
*   `paymentMethod`: Enum ['CreditCard', 'ConvenienceStore', 'Crypto']
*   `shippingDetails`: Object (收件資訊)

### 2.4 Cart (購物車)
*(註：未登入者使用 LocalStorage，登入後同步至資料庫以跨裝置保留)*
*   `_id`: ObjectId
*   `user`: ObjectId (Ref: User, Unique)
*   `items`: Array of Objects
    *   `product`: ObjectId (Ref: Product)
    *   `quantity`: Number

### 2.5 Discount (折扣碼)
*   `_id`: ObjectId
*   `code`: String (Unique, e.g., 'SUMMER20')
*   `type`: Enum ['Percentage', 'Fixed']
*   `value`: Number (e.g., 20% or $100)
*   `minPurchaseAmount`: Number (最低消費門檻)
*   `expiryDate`: Date
*   `isActive`: Boolean

---

## 3. 核心功能詳細設計與 API 規劃

### 3.1 認證與授權模組 (Auth & Authorization)
*   **設計策略**: 使用 JWT (JSON Web Token)，基於安全性考量，登入成功後將 Token 設定於 `HttpOnly Cookie` 中，防止 XSS 攻擊讀取 Token。
*   **API 規劃**:
    *   `POST /api/auth/register`: 註冊新使用者 (密碼需 hash)。
    *   `POST /api/auth/login`: 驗證帳密，發行 HttpOnly Cookie。
    *   `POST /api/auth/logout`: 清除 Cookie。
    *   `GET /api/auth/me`: 取得當前登入者資訊 (依賴 Cookie 驗證)。
*   **測試重點**: 確保 JWT 簽發邏輯正確；驗證 Middleware 在沒有 Token 或權限不足時能正確阻擋請求。

### 3.2 商品與購物車模組 (Product & Cart)
*   **設計策略**: 
    *   商品列表需支援分頁 (Pagination) 與分類過濾。
    *   購物車加入商品時需檢查 `Product.stock` 是否充足。
*   **API 規劃 (商品)**:
    *   `GET /api/products`: 獲取商品列表 (支援 `?page=1&limit=10&category=xxx`)。
    *   `GET /api/products/:id`: 獲取單一商品詳情。
*   **API 規劃 (購物車)**:
    *   `GET /api/cart`: 獲取當前使用者購物車。
    *   `POST /api/cart/items`: 新增/更新商品至購物車。
    *   `DELETE /api/cart/items/:productId`: 移除購物車商品。
*   **測試重點**: `CartService` 必須獨立測試加入購物車時的庫存邊界值計算，以及商品累加邏輯。

### 3.3 結帳與金流模組 (Checkout & Payment)
*   **設計策略**: 根據提案書需求，實作多種金流。以「建立訂單」為核心，再導向各金流服務。
*   **API 規劃**:
    *   `POST /api/orders/checkout`: 提交購物車內容與收件資訊，建立 `Pending` 狀態訂單。
    *   `POST /api/payments/stripe`: 處理信用卡金流 (回傳 Client Secret 給前端)。
    *   `POST /api/payments/crypto`: 驗證前端傳入的 ERC20 Transaction Hash (可串接 Ethers.js 驗證 Sepolia 測試網交易)。
    *   `POST /api/payments/store-mock`: 處理超商取貨付款的 Mock 流程。
*   **測試重點**: 結帳 `CheckoutService` 必須能在計算總價時，正確扣除折扣碼 (`DiscountService`) 的金額，並鎖定商品當下價格 `priceAtPurchase`。

### 3.4 後台管理系統 (Admin Dashboard)
*   **設計策略**: 前端實作專屬的管理員 Layout。後端實作 RBAC Middleware，驗證 `req.user.role`。
*   **API 規劃**:
    *   `POST /api/admin/products`: (Admin Only) 上架新商品。
    *   `PUT /api/admin/products/:id`: (Admin Only) 編輯商品資訊/價格/庫存。
    *   `GET /api/admin/orders`: (Admin Only) 檢視所有訂單。
    *   `PUT /api/admin/orders/:id/status`: (Admin Only) 更新訂單出貨狀態。
    *   `POST /api/admin/users`: (SuperAdmin Only) 新增後台員工帳號。
*   **測試重點**: RBAC Middleware 的單元測試，確保不同 Role 嘗試存取特定 Endpoint 時的正確拒絕/放行行為。

---

## 4. 單元測試策略 (Testing Strategy)

為落實「功能獨立性」，專案應包含以下測試規劃：

1.  **後端 Service 單元測試 (Unit Tests)**:
    *   使用 `Jest` 作為測試框架。
    *   針對 `OrderService`, `CartService`, `DiscountService` 等核心商業邏輯撰寫測試。
    *   **隔離資料庫**: 完全利用 Jest 提供的 Mock 功能，Mock `Repository` 層的資料庫操作，專注測試邏輯（如計算折扣是否正確、庫存不足是否拋出 Error）。
2.  **後端 API 整合測試 (Integration Tests)**:
    *   使用 `Supertest` 搭配記憶體資料庫 (如 `mongodb-memory-server`)。
    *   測試完整的 Request -> Controller -> Service -> DB -> Response 流程。
3.  **前端 Hook / 邏輯測試**:
    *   針對複雜的自定義 Hook (如 `useCart`, `useDiscount`) 使用 `React Testing Library` 或 `@testing-library/react-hooks` 進行單元測試。

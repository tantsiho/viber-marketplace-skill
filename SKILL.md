---
name: ecommerce-marketplace
description: Blueprint for building multi-vendor marketplace with Next.js App Router, Prisma, Auth.js. Covers roles (BUYER/SELLER/ADMIN), RBAC, ProductStatus, folder structure, and pain-point solutions. Use when building or extending 多賣家電商、marketplace、蝦皮風格、seller dashboard、商品審核、RBAC、or buyer/seller/admin 後台.
---

# Next.js 多賣家電商平台開發藍圖

當使用者要打造或擴充「多賣家電商／Marketplace（蝦皮風格）」時，依本藍圖：同一套帳號、角色 BUYER/SELLER/ADMIN、路由群組 (buyer)/(seller)/(admin)、middleware + Server Action 雙重權限、商品狀態機與審核流程。實作時對照下列結構與程式片段，避免常見痛點（無草稿、越權、後台混亂）。

---

## 目標：避免的常見痛點

- 商品頁沒有草稿／下架
- 後台混亂、權限控制差
- 買家／賣家／管理員功能混在一起
- 賣家看到別人資料（越權）

---

## 1. 平台類型與角色定義

- **類型**：多賣家（Multi-vendor Marketplace），買家前台、賣家獨立後台、平台 admin 管全部 + 審核
- **帳號**：同一套帳號，email 註冊／登入，依 role 跳不同 dashboard
- **角色**：**BUYER** / **SELLER** / **ADMIN**（可再加 **SELLER_STAFF**）

---

## 2. 推薦技術棧

| 層級 | 技術 |
|------|------|
| Framework | Next.js 15/16（App Router + React Server Components） |
| Styling | Tailwind CSS + shadcn/ui 或 Radix UI |
| Auth & RBAC | Auth.js v5 + 自寫 RBAC middleware（或 Permit.io / Clerk） |
| Database | Prisma + PostgreSQL（或 Supabase） |
| State | Zustand 或 React Context；或 Server Components 直接 fetch |
| Deployment | Vercel |

---

## 3. 資料模型（Prisma 核心）

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  password  String?
  role      Role     @default(BUYER)
  seller    Seller?  @relation(fields: [sellerId], references: [id])
  sellerId  String?  @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
enum Role { BUYER SELLER ADMIN }

model Seller {
  id          String    @id @default(uuid())
  user        User      @relation(fields: [userId], references: [id])
  userId      String    @unique
  storeName   String
  storeSlug   String    @unique
  description String?
  approved    Boolean   @default(false)
  products    Product[]
  orders      Order[]
}

model Product {
  id          String        @id @default(uuid())
  title       String
  slug        String        @unique
  description String?
  price       Float
  stock       Int           @default(0)
  status      ProductStatus @default(DRAFT)
  seller      Seller        @relation(fields: [sellerId], references: [id])
  sellerId    String
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt
}
enum ProductStatus { DRAFT PENDING PUBLISHED HIDDEN SOLD_OUT }

model Order {
  id        String      @id @default(uuid())
  buyerId   String
  sellerId  String
  total     Float
  status    OrderStatus  @default(PENDING_PAYMENT)
  items     OrderItem[]
  createdAt DateTime     @default(now())
}
model OrderItem {
  id        String   @id @default(uuid())
  orderId   String
  productId String
  quantity  Int
  price     Float
  order     Order    @relation(fields: [orderId], references: [id])
  product   Product  @relation(fields: [productId], references: [id])
}
enum OrderStatus { PENDING_PAYMENT PAID SHIPPED DELIVERED COMPLETED CANCELLED REFUNDED }
```

---

## 4. 專案資料夾結構（modular）

```
app/
├── (auth)/                    # 登入／註冊／忘記密碼
│   ├── login/page.tsx
│   └── register/page.tsx
├── (buyer)/                   # 前台買家（layout: navbar/cart）
│   ├── products/[slug]/page.tsx
│   ├── cart/page.tsx
│   └── account/page.tsx
├── (seller)/                  # 賣家後台（auth + seller role）
│   ├── dashboard/page.tsx
│   ├── products/page.tsx, [id]/edit/page.tsx
│   ├── orders/page.tsx
│   └── settings/page.tsx
├── (admin)/                   # 平台後台（admin role）
│   ├── dashboard/page.tsx
│   ├── users/page.tsx, sellers/page.tsx
│   ├── products/approve/page.tsx
│   └── reports/page.tsx
└── api/

lib/
├── auth/config.ts
├── rbac/index.ts
└── prisma/

components/
├── ui/
├── dashboard/
├── seller/, admin/
└── layout/SellerLayout.tsx, AdminLayout.tsx
```

---

## 5. RBAC：middleware + Server Action 雙重檢查

**middleware**：依 path 與 `token.role` 阻擋未授權進入 `/seller/*`、`/admin/*`。

```typescript
export default withAuth(
  function middleware(req) {
    const token = req.nextauth.token
    if (!token) return NextResponse.redirect(new URL("/login", req.url))
    const { pathname } = req.nextUrl
    if (pathname.startsWith("/seller") && token.role !== "SELLER")
      return NextResponse.redirect(new URL("/login", req.url))
    if (pathname.startsWith("/admin") && token.role !== "ADMIN")
      return NextResponse.redirect(new URL("/login", req.url))
  },
  { callbacks: { authorized: ({ token }) => !!token } }
)
export const config = { matcher: ["/seller/:path*", "/admin/:path*"] }
```

**Server Action**：賣家相關 query 一律 `where: { sellerId: session.user.sellerId! }`，並檢查 `session?.user.role === "SELLER"`，否則 throw Unauthorized。

**登入後跳轉**：`SELLER` → `/seller/dashboard`，`ADMIN` → `/admin/dashboard`，其餘 → `/`。

---

## 6. 開發優先順序

1. Auth.js + Prisma + User/Seller + role
2. middleware + (buyer)/(seller)/(admin) layout
3. 賣家商品管理（列表、編輯、草稿／上架／下架、篩選）
4. 商品狀態機 + 前台只顯示 PUBLISHED
5. 訂單（賣家只看自己、admin 看全部）
6. admin 商品審核
7. Dashboard 指標（銷售額、待處理訂單）
8. 進階：批量、匯入、財務、運費模板

---

## 7. 痛點對應

| 痛點 | 做法 |
|------|------|
| 無草稿／下架 | ProductStatus enum + 按鈕控制 |
| 賣家看到別人資料 | 所有 query `where sellerId = ...` |
| 後台亂 | 路由群組 + 專屬 layout |
| 權限亂 | middleware + server action 雙重檢查 |
| 一次寫不完 | 「儲存草稿」→ status = DRAFT |

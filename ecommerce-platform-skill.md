# Next.js 多賣家電商平台開發藍圖（架構棧整合版）

針對台灣／華語市場的多賣家電商平台（類似蝦皮風格）的完整開發藍圖與實作指南，適合從零自刻（hand-crafted）或小團隊開發。

**專案定位**：Next.js App Router + Prisma + PostgreSQL + Auth.js 的多賣家 Marketplace 基礎架構。

---

## 目標：解決常見自刻電商後台痛點

- 商品頁沒有草稿功能
- 建好的商品無法下架
- 後台亂七八糟、權限控制差
- 買家／賣家／管理員共用帳號卻功能混亂
- 資料越權（賣家看到別人商品）

---

## 1. 平台類型與角色定義

- **類型**：多賣家平台（Multi-vendor Marketplace，蝦皮風格）
  - 買家在前台逛買
  - 賣家有獨立後台管自己商品／訂單
  - 平台 admin 管全部 + 審核

- **帳號系統**：同一套帳號，用 email 註冊／登入，登入後依 role 跳不同 dashboard

- **主要角色**（Role enum）
  - **BUYER**：一般買家，前台購物 + 會員中心
  - **SELLER**：賣家，只能管理自己的店舖、商品、訂單
  - **ADMIN**：平台管理員，可管理所有賣家、審核商品、查看全域報表
  - （可再加 **SELLER_STAFF**：賣家子帳號）

---

## 2. 推薦技術棧（2026 主流）

| 層級 | 技術 |
|------|------|
| Framework | Next.js 15/16（App Router + React Server Components） |
| Styling | Tailwind CSS + shadcn/ui 或 Radix UI（admin 面板超適合） |
| Auth & RBAC | Auth.js v5（credential + OAuth）+ 自寫 RBAC middleware（或用 Permit.io / Clerk Organizations 快速版） |
| Database / ORM | Prisma + PostgreSQL（或 Supabase 若想 serverless） |
| State | Zustand 或 React Context（簡單 dashboard 夠用）；或 Server Components 直接 fetch |
| Deployment | Vercel（Next.js 原生最佳） |

---

## 3. 資料模型（Prisma schema 核心片段）

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  password  String?  // credential auth 時使用 (bcrypt hash)
  role      Role     @default(BUYER)
  seller    Seller?  @relation(fields: [sellerId], references: [id])
  sellerId  String?  @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum Role {
  BUYER
  SELLER
  ADMIN
}

model Seller {
  id          String    @id @default(uuid())
  user        User      @relation(fields: [userId], references: [id])
  userId      String    @unique
  storeName   String
  storeSlug   String    @unique
  description String?
  approved    Boolean   @default(false)  // 平台審核通過才可上架商品
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

enum ProductStatus {
  DRAFT
  PENDING     // 待平台審核（多賣家必備）
  PUBLISHED
  HIDDEN
  SOLD_OUT
}

model Order {
  id        String      @id @default(uuid())
  buyerId   String
  sellerId  String
  total     Float
  status    OrderStatus @default(PENDING_PAYMENT)
  items     OrderItem[]
  createdAt DateTime    @default(now())
}

model OrderItem {
  id        String   @id @default(uuid())
  orderId   String
  productId String
  quantity  Int
  price     Float    // 當時成交價
  order     Order    @relation(fields: [orderId], references: [id])
  product   Product  @relation(fields: [productId], references: [id])
}

enum OrderStatus {
  PENDING_PAYMENT
  PAID
  SHIPPED
  DELIVERED
  COMPLETED
  CANCELLED
  REFUNDED
}
```

---

## 4. 建議專案資料夾結構（modular 設計）

```
app/
├── (auth)/                    # 公開路由：登入／註冊／忘記密碼
│   ├── login/page.tsx
│   └── register/page.tsx
├── (buyer)/                   # 前台買家區（layout 包 navbar/cart 等）
│   ├── products/[slug]/page.tsx
│   ├── cart/page.tsx
│   └── account/page.tsx
├── (seller)/                  # 賣家後台（需 auth + seller role）
│   ├── dashboard/page.tsx
│   ├── products/
│   │   ├── page.tsx           # 商品列表
│   │   └── [id]/edit/page.tsx
│   ├── orders/page.tsx
│   └── settings/page.tsx
├── (admin)/                   # 平台後台（需 admin role）
│   ├── dashboard/page.tsx
│   ├── users/page.tsx         # 管理所有用戶
│   ├── sellers/page.tsx       # 賣家審核／列表
│   ├── products/approve/page.tsx  # 商品審核
│   └── reports/page.tsx
└── api/                       # App Router API routes（若不用 server actions）

lib/
├── auth/
│   └── config.ts              # Auth.js 設定 + role helpers
├── rbac/
│   └── index.ts               # 權限檢查 helper
└── prisma/                    # schema 等

components/
├── ui/                        # shadcn/ui 元件
├── dashboard/                 # 共用 dashboard 卡片、表格
├── seller/                    # 賣家專用 component
├── admin/                     # admin 專用
└── layout/
    ├── SellerLayout.tsx
    └── AdminLayout.tsx

public/
prisma/
└── schema.prisma
```

---

## 5. 權限控制（RBAC）實作重點

用 **middleware** 保護路由，再在 **Server Action** 做資料隔離（雙重檢查，防越權）。

### 5.1 middleware.ts（保護路由）

```typescript
import { withAuth } from "next-auth/middleware"
import { NextResponse } from "next/server"

export default withAuth(
  function middleware(req) {
    const token = req.nextauth.token
    if (!token) return NextResponse.redirect(new URL("/login", req.url))

    const { pathname } = req.nextUrl

    if (pathname.startsWith("/seller") && token.role !== "SELLER") {
      return NextResponse.redirect(new URL("/login", req.url))
    }
    if (pathname.startsWith("/admin") && token.role !== "ADMIN") {
      return NextResponse.redirect(new URL("/login", req.url))
    }
    // seller 只能看自己資料 → 在 server action 再細查
  },
  { callbacks: { authorized: ({ token }) => !!token } }
)

export const config = { matcher: ["/seller/:path*", "/admin/:path*"] }
```

### 5.2 Server Action 裡再細緻檢查（防越權）

```typescript
"use server"
import { getServerSession } from "next-auth"
import { prisma } from "@/lib/prisma"

export async function getMyProducts() {
  const session = await getServerSession()
  if (!session || session.user.role !== "SELLER") throw new Error("Unauthorized")

  return prisma.product.findMany({
    where: { sellerId: session.user.sellerId! },
  })
}
```

### 5.3 登入後依角色跳轉（login page 或 callback）

```typescript
if (user.role === "SELLER") {
  redirect("/seller/dashboard")
} else if (user.role === "ADMIN") {
  redirect("/admin/dashboard")
} else {
  redirect("/")
}
```

---

## 6. 開發優先順序建議

1. 設定 **Auth.js + Prisma + User/Seller 模型 + role 欄位**（先 credential 或 Google OAuth）
2. 實作 **middleware + 三種 layout**：`(buyer)` / `(seller)` / `(admin)`
3. 完成 **賣家商品管理**（列表、編輯、草稿／上架／下架、篩選）← 最常卡關處
4. **商品狀態機** + 前台只顯示 `PUBLISHED`
5. **訂單管理**（賣家只看自己的、admin 看全部）
6. **平台 admin 商品審核流程**
7. **Dashboard 關鍵指標**（銷售額、待處理訂單等）
8. 進階：**批量、匯入、財務對帳、運費模板**

---

## 7. 常見痛點對應解決方案

| 痛點 | 解決方案 |
|------|----------|
| 商品沒有草稿／下架功能 | `ProductStatus` enum + 按鈕控制狀態 |
| 賣家看到別人資料 | 所有 query 強制 `where sellerId = ...` |
| 後台亂七八糟 | 分離路由群組 + 專屬 layout |
| 權限混亂 | middleware + server action 雙重檢查 |
| 一次寫不完 | 「儲存草稿」按鈕 → `status = DRAFT` |

---

## License

MIT License。歡迎 fork、PR、star。若有幫助，請在 repo 裡 mention 出處，謝謝。

**作者**：美術數膠｜Fine Art Digital Plastic (@psrmidi)

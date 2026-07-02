# 价格模型与后台 API 设计

> 版本：v1.1
> 日期：2026-07-02
> 状态：已实现（catalog-service + catalog-backoffice-web + catalog-web 接入）
> v1.1 变更：商品/分类中英双语字段；客户访问码登录；公开取价 API；catalog-web 接真实接口

---

## 1. 价格模型（已定稿）

采用「价格级别 + 客户价格组」模式：

```
PriceLevel      价格级别（默认 4 个，可增删）
ProductPrice    商品 × 级别 → 具体价格（每商品最多 N 条）
Product         商品，自带 list_price（默认价/零售价）
PriceGroup      价格组（如：批发A、VIP），可设默认级别
PriceGroupRule  价格组 × 分类 → 价格级别（分类可为一级或二级）
Customer        客户 → 关联一个价格组（可为空）
```

### 取价规则（fallback 链）

对某客户展示某商品价格时，按以下顺序取第一个命中的：

1. 客户价格组中**该商品二级分类**的规则 → 该级别的商品价；
2. 客户价格组中**该商品一级分类**的规则 → 该级别的商品价（二级规则优先于一级）；
3. 价格组的**默认级别** → 该级别的商品价；
4. 以上任一步命中级别但商品未录该级别价 → 继续向下；
5. **商品默认价 list_price**（兜底，永远存在）。

匿名客户 / 未关联价格组的客户 / 停用客户 → 直接 list_price。

> 安全要求：取价必须在 catalog-service 服务端完成，接口只返回「当前客户适用的那一个价格」，绝不能把全部级别价发给前端。

### 订单快照

下单时把成交单价写入订单明细（快照），后续调价、调组不影响历史订单。（订单模块待实现）

---

## 2. 数据库

- PostgreSQL，库名 `catalog`（开发环境 localhost:5432，postgres/123456）。
- 表由 JPA `ddl-auto: update` 自动建：`categories`、`products`、`product_prices`、`price_levels`、`price_groups`、`price_group_rules`、`customers`、`admin_users`。
- 分类为两级：一级 `parent_id IS NULL` 且**不带图片**；二级带 `image_url`。层级限制在 API 层强制。
- **中英双语**：分类 `name_zh`/`name_en`，商品 `name_zh`/`name_en`、`description_zh`/`description_en`。中文必填、英文可空，前端展示时互为回落；搜索同时匹配中英文名。
- **客户访问码**：`customers.access_code`（唯一，8 位、去除易混淆字符），后台生成/重置，客户用它在前台登录。

## 3. catalog-service 后台 API（第一版）

Base URL: `http://localhost:8081`（8080 被本机其他服务占用）

### 认证（session cookie）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/login` | `{username, password}`；错误 → 401 `{message}` |
| GET | `/api/auth/me` | 当前登录人 `{username, displayName}` |
| POST | `/api/auth/logout` | 退出 |

默认账号：`admin / admin123`（首次启动自动创建，**上线前必须改**）。
除 `/api/auth/**`、`/api/public/**`、`/uploads/**` 外全部需要登录（401）。

### 管理接口（均需登录）

| 资源 | 路径 | 说明 |
|---|---|---|
| 分类 | `GET/POST /api/admin/categories`，`PUT/DELETE /api/admin/categories/{id}` | GET 返回两级树；一级分类 imageUrl 强制为 null；三级 → 400；删除有子分类/商品 → 400 |
| 商品 | `GET/POST /api/admin/products`，`GET/PUT/DELETE /api/admin/products/{id}` | 分页 `?page&size&keyword&categoryId`（传一级分类 id 时自动包含其子分类）；请求体含 `prices:[{priceLevelId,price}]`，保存时整体重建；商品必须挂二级分类 |
| 价格级别 | `GET/POST /api/admin/price-levels`，`PUT/DELETE .../{id}` | 被引用时删除 → 409 |
| 价格组 | `GET/POST /api/admin/price-groups`，`PUT/DELETE .../{id}` | 含 `defaultPriceLevelId` 与 `rules:[{categoryId,priceLevelId}]`（保存整体重建）；有客户关联时删除 → 400 |
| 客户 | `GET/POST /api/admin/customers`，`PUT/DELETE .../{id}` | 分页 `?page&size&keyword`（搜姓名/电话）；`priceGroupId` 可空 |
| 上传 | `POST /api/admin/uploads`（multipart `file`） | 仅图片（jpg/png/gif/webp，≤10MB），返回 `{url:"/uploads/xxx.png"}`；文件存 `catalog-service/uploads/`（已 gitignore） |

另有 `POST /api/admin/customers/{id}/access-code`：生成/重置客户访问码（旧码立即失效）。

分页响应统一为 `{content, total, page, size}`；错误统一为 `{message}`（400/401/404/409）。

### 公开接口（前台用，无需管理登录）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/public/auth/login` | `{accessCode}` → 建立客户会话（session 属性，与管理登录互不影响）；无效/停用 → 401 |
| GET | `/api/public/auth/me` | `{name, priceGroupName}` |
| POST | `/api/public/auth/logout` | 仅清除客户会话 |
| GET | `/api/public/categories` | 两级分类树（双语 + 二级图片） |
| GET | `/api/public/products` | 仅上架商品，分页 `?categoryId&keyword&page&size`；每个商品返回 `price`（服务端按当前客户价格组解析的适用价）与 `listPrice` |
| GET | `/api/public/products/{id}` | 商品详情（同上取价）；下架 → 404 |

> 取价只在服务端做，响应只含"当前身份适用的一个价格"，不泄漏其他级别价。fallback 链见第 1 节，已实现于 `service/PriceResolver.java`，并有 E2E 验证（二级规则命中 / 一级规则命中但缺级别价回落默认级别 / 匿名 listPrice）。

## 4. catalog-backoffice-web（第一版）

- Vite 8 + React 19 + antd 6 + react-router 7；`pnpm dev` 启动（5173）。
- Vite 代理：`/api`、`/uploads` → `http://localhost:8081`，同源无 CORS。
- 已实现页面：登录、商品管理（分页/搜索/分类筛选/级别价编辑/图片上传）、分类管理（两级树、二级图片）、价格组（规则编辑器）、价格级别、客户管理；订单页为占位。
- 路由守卫：未登录访问任何页面 → 重定向 `/login`；API 401 → 自动跳登录页。

## 5. catalog-web（已接入真实 API）

- Next.js rewrites 把 `/api`、`/uploads` 代理到 catalog-service（`CATALOG_API_URL` 环境变量可覆盖，默认 localhost:8081），同源无 CORS。
- 页面：产品（一级分类侧栏 + 二级分类图片卡）、分类商品列表、搜索、购物车（localStorage，价格实时按登录身份取）、我的（访问码登录/退出 + 中英文切换）。
- 双语：语言存 localStorage（`lib/i18n.tsx`），产品/分类名按语言显示并互为回落。
- 划线价：`price < listPrice` 时显示划线原价。
- 注意：本机 3000 端口被其他进程占用时 next dev 会自动用 3001。

## 6. 待办 / 下一步

- [ ] 订单模块（下单、订单列表、状态流转、价格快照）
- [ ] 修改 admin 密码功能、后台账号管理
- [ ] 生产环境配置（数据库密码外置、HTTPS、上传目录持久化）
- [ ] catalog-web 商品详情页（目前只有列表）

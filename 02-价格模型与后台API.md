# 价格模型与后台 API 设计

> 版本：v1.0
> 日期：2026-07-02
> 状态：已实现（catalog-service + catalog-backoffice-web 第一版）

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

分页响应统一为 `{content, total, page, size}`；错误统一为 `{message}`（400/401/404/409）。

## 4. catalog-backoffice-web（第一版）

- Vite 8 + React 19 + antd 6 + react-router 7；`pnpm dev` 启动（5173）。
- Vite 代理：`/api`、`/uploads` → `http://localhost:8081`，同源无 CORS。
- 已实现页面：登录、商品管理（分页/搜索/分类筛选/级别价编辑/图片上传）、分类管理（两级树、二级图片）、价格组（规则编辑器）、价格级别、客户管理；订单页为占位。
- 路由守卫：未登录访问任何页面 → 重定向 `/login`；API 401 → 自动跳登录页。

## 5. 待办 / 下一步

- [ ] 订单模块（下单、订单列表、状态流转、价格快照）
- [ ] catalog-web 客户端接入真实 API（替换 mock-data），实现客户登录 + 服务端取价接口 `/api/public/...`
- [ ] 客户登录方式（手机号+验证码 / 账号密码 / 访问码，待定）
- [ ] 修改 admin 密码功能、后台账号管理
- [ ] 生产环境配置（数据库密码外置、HTTPS、上传目录持久化）

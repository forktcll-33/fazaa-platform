# 🗄️ ERD — Fazaa Platform (PostgreSQL)

> الهدف: بنية بيانات نظيفة، قابلة للتوسع، وتخدم MVP (عميل/فني/طلبات/مدفوعات/تقييمات/نزاعات).

---

## 🧩 نظرة عامة على الجداول

- **auth & users**
  - `users` — حسابات جميع المستخدمين (عميل/فني/إدارة)
  - `user_profiles` — تفاصيل إضافية (اسم، صورة، لغة…)
  - `device_tokens` — لتراسل الإشعارات (FCM/APNS)
- **providers (الفنيون)**
  - `providers` — ملف الفني (تخصص، نطاق العمل، الحالة)
  - `provider_documents` — وثائق الهوية/الرخص
  - `provider_services` — الخدمات التي يقدمها الفني وأسعارها الإرشادية
  - `service_categories` — شجرة الفئات (سباكة/كهرباء/مكيفات/…)
  - `service_items` — عناصر/باقات جاهزة (تنظيف غرفة، كشف تسريب…)
  - `provider_service_areas` — نطاقات العمل (مدن/إحداثيات اختيارية)
- **orders (الطلبات)**
  - `orders` — الطلب الرئيسي
  - `order_items` — تفاصيل العناصر/الخدمات داخل الطلب
  - `order_events` — تتبع حالة الطلب (timeline: created/accepted/arrived/started/completed/cancelled)
  - `order_media` — صور/فيديو قبل/بعد
  - `order_chat_messages` — دردشة داخل الطلب
- **payments & wallet**
  - `payments` — عمليات الدفع (عربون/كامل)
  - `wallets` — محفظة كل فني
  - `wallet_transactions` — قيود مالية (عمولة/تحويل/سحب)
  - `payout_requests` — طلبات سحب أرباح الفني
- **quality**
  - `ratings` — تقييمات العملاء للفنيين
  - `disputes` — نزاعات الطلبات
- **ops**
  - `admin_users` — حسابات الإدارة
  - `audit_logs` — سجل تدقيق للعمليات الإدارية

---

## 🧱 Enums (Postgres)

- `user_role`: `customer`, `provider`, `admin`
- `provider_status`: `pending_review`, `approved`, `rejected`, `suspended`
- `document_type`: `national_id`, `residency_id`, `work_permit`, `other`
- `order_status`: `created`, `awaiting_payment`, `queued`, `assigned`, `accepted`, `enroute`, `arrived`, `in_progress`, `completed`, `cancelled`
- `payment_status`: `initiated`, `authorized`, `captured`, `failed`, `refunded`, `released`
- `payment_method`: `mada`, `apple_pay`, `stc_pay`, `visa`, `mastercard`
- `dispute_status`: `open`, `under_review`, `resolved_refund`, `resolved_reject`, `dismissed`
- `media_type`: `image`, `video`
- `chat_sender_type`: `customer`, `provider`, `system`

> ملاحظة: يمكن حفظ الـ enums كـ `CHECK CONSTRAINT` إن رغبت، لكن enums الأصلية في PG مناسبة هنا.

---

## 👥 users & auth

### `users`
- `id` (PK, uuid, default gen_random_uuid())
- `phone_e164` (unique)
- `email` (nullable, unique)
- `role` (user_role, not null, default `customer`)
- `password_hash` (nullable — نعتمد OTP في البداية)
- `is_active` (bool, default true)
- `created_at`, `updated_at`

**Indexes:**  
- `idx_users_phone` (unique)  
- `idx_users_role`

### `user_profiles`
- `user_id` (PK/FK → users.id, on delete cascade)
- `full_name`
- `avatar_url`
- `preferred_lang` (e.g., `ar`, `en`)
- `default_city`
- `rating_avg` (numeric(3,2), cached)
- `rating_count` (int)
- `created_at`, `updated_at`

### `device_tokens`
- `id` (PK)
- `user_id` (FK)
- `token` (unique)
- `platform` (`ios`/`android`)
- `created_at`, `updated_at`

---

## 👷 providers (الفنيون)

### `providers`
- `user_id` (PK/FK → users.id)
- `status` (provider_status, default `pending_review`)
- `headline` (نبذة قصيرة)
- `bio` (وصف)
- `base_city`
- `lat`, `lng` (nullable — آخر تموضع معروف)
- `avg_eta_minutes` (int, nullable)
- `avg_response_sec` (int, nullable)
- `rating_avg` (numeric(3,2)), `rating_count` (int)
- `created_at`, `updated_at`

### `provider_documents`
- `id` (PK)
- `provider_id` (FK → providers.user_id)
- `type` (document_type)
- `file_url`
- `verified` (bool, default false)
- `expires_at` (nullable)
- `created_at`, `updated_at`

### `service_categories`
- `id` (PK)
- `parent_id` (nullable FK self)
- `slug` (unique) — مثال: plumbing, hvac
- `name_ar`, `name_en`
- `sort_order` (int)
- `is_active` (bool, default true)

### `service_items`
- `id` (PK)
- `category_id` (FK → service_categories.id)
- `slug` (unique) — مثال: split-ac-clean, leak-detection
- `name_ar`, `name_en`
- `description_ar`, `description_en`
- `unit` (مثال: "زيارة", "جهاز", "متر")
- `is_active` (bool, default true)

### `provider_services`
- `id` (PK)
- `provider_id` (FK → providers.user_id)
- `service_item_id` (FK → service_items.id)
- `base_price` (numeric(10,2))
- `min_price` (numeric(10,2), nullable)
- `max_price` (numeric(10,2), nullable)
- `is_active` (bool, default true)

**Unique:** (`provider_id`, `service_item_id`)

### `provider_service_areas`
- `id` (PK)
- `provider_id` (FK → providers.user_id)
- `city`
- `polygon_geojson` (nullable) — لنطاقات دقيقة لاحقًا

---

## 🧾 orders

### `orders`
- `id` (PK, uuid)
- `customer_id` (FK → users.id)
- `provider_id` (FK → users.id, nullable حتى الإسناد)
- `status` (order_status, default `created`)
- `category_id` (FK → service_categories.id)
- `scheduled_at` (nullable) — الآن أو لاحقًا
- `address_text`
- `lat`, `lng`
- `issue_description` (text)
- `estimated_price` (numeric(10,2), nullable)
- `final_price` (numeric(10,2), nullable)
- `payment_id` (FK → payments.id, nullable)
- `cancel_reason` (nullable)
- `created_at`, `updated_at`

**Indexes:**  
- `idx_orders_customer`  
- `idx_orders_provider`  
- `idx_orders_status`  
- `idx_orders_geo` (btree on city or gist on point if needed)

### `order_items`
- `id` (PK)
- `order_id` (FK → orders.id, on delete cascade)
- `service_item_id` (FK)
- `qty` (numeric(10,2) or int)
- `unit_price` (numeric(10,2))
- `line_total` (generated: qty * unit_price)

### `order_events`
- `id` (PK)
- `order_id` (FK)
- `status_from`, `status_to` (order_status)
- `note` (nullable)
- `created_at` (timestamp)

### `order_media`
- `id` (PK)
- `order_id` (FK)
- `uploader_id` (FK → users.id)
- `type` (media_type)
- `file_url`
- `created_at`

### `order_chat_messages`
- `id` (PK)
- `order_id` (FK)
- `sender_type` (chat_sender_type)
- `sender_id` (FK → users.id)
- `message` (text, nullable إذا كان فقط مرفق)
- `attachment_url` (nullable)
- `created_at`

---

## 💳 payments & wallet

### `payments`
- `id` (PK)
- `order_id` (FK → orders.id, unique)
- `amount` (numeric(10,2))
- `currency` (char(3), default `SAR`)
- `status` (payment_status)
- `method` (payment_method, nullable حتى الاختيار)
- `provider_reference` (gateway charge id)
- `captured_at` (nullable)
- `released_at` (nullable) — إفراج للـ wallet
- `created_at`, `updated_at`

### `wallets`
- `id` (PK)
- `owner_id` (FK → users.id) — لكل فني محفظة
- `balance` (numeric(12,2), default 0)
- `currency` (char(3), default `SAR`)
- `created_at`, `updated_at`

**Unique:** (`owner_id`)

### `wallet_transactions`
- `id` (PK)
- `wallet_id` (FK → wallets.id)
- `type` (`credit`/`debit`)
- `amount` (numeric(12,2))
- `ref_order_id` (nullable)
- `ref_payment_id` (nullable)
- `description`
- `created_at`

### `payout_requests`
- `id` (PK)
- `wallet_id` (FK)
- `amount` (numeric(12,2))
- `iban` (string) — أو حساب بنكي
- `status` (`requested`, `processing`, `paid`, `rejected`)
- `created_at`, `updated_at`

---

## ⭐ الجودة: التقييمات والنزاعات

### `ratings`
- `id` (PK)
- `order_id` (FK unique) — تقييم لكل طلب
- `rater_id` (FK → users.id)
- `ratee_id` (FK → users.id) — غالبًا الفني
- `score` (int 1..5)
- `comment` (nullable)
- `created_at`

**Aggregates:** تحديث `rating_avg`/`rating_count` في `providers`/`user_profiles` عبر trigger.

### `disputes`
- `id` (PK)
- `order_id` (FK)
- `opened_by` (FK → users.id)
- `status` (dispute_status, default `open`)
- `reason` (text)
- `resolution_note` (nullable)
- `created_at`, `updated_at`

---

## 🛡️ الإدارة والتدقيق

### `admin_users`
- `id` (PK)
- `user_id` (FK → users.id) — أو حساب مستقل
- `roles` (jsonb) — صلاحيات دقيقة مستقبلًا
- `created_at`

### `audit_logs`
- `id` (PK)
- `actor_user_id` (FK → users.id or admin_users)
- `action` (string)
- `target_table` (string)
- `target_id` (uuid/int)
- `metadata` (jsonb)
- `created_at`

---

## 🔎 فهارس (Indexes) مهمة للأداء

- `orders(status, created_at desc)`
- جغرافي: `orders` على `POINT(lat, lng)` باستخدام `postgis` مستقبلًا
- `provider_services(provider_id, service_item_id)`
- `order_chat_messages(order_id, created_at)`
- `ratings(ratee_id, created_at)`

---

## 🧪 ملاحظات تنفيذ سريعة

- استخدم `uuid-ossp` أو `pgcrypto` لتوليد UUIDs.
- خزّن المبالغ دائمًا كنumeric وثبّت `currency`.
- الصور على S3 + حفظ روابطها فقط.
- عمليات الدفع عبر Webhooks → تحدّث `payments` ثم `wallets`.
- Trigger لتحديث متوسط التقييم بعد كل إدخال في `ratings`.

---

## 🧰 مثال DDL مبسط لعدة جداول (مرجعي)

```sql
create type user_role as enum ('customer','provider','admin');
create type order_status as enum ('created','awaiting_payment','queued','assigned','accepted','enroute','arrived','in_progress','completed','cancelled');

create table users (
  id uuid primary key default gen_random_uuid(),
  phone_e164 text unique not null,
  email text unique,
  role user_role not null default 'customer',
  password_hash text,
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table providers (
  user_id uuid primary key references users(id) on delete cascade,
  status text not null default 'pending_review',
  headline text,
  bio text,
  base_city text,
  lat double precision,
  lng double precision,
  rating_avg numeric(3,2),
  rating_count int default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table orders (
  id uuid primary key default gen_random_uuid(),
  customer_id uuid not null references users(id),
  provider_id uuid references users(id),
  status order_status not null default 'created',
  category_id bigint,
  scheduled_at timestamptz,
  address_text text,
  lat double precision,
  lng double precision,
  issue_description text,
  estimated_price numeric(10,2),
  final_price numeric(10,2),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

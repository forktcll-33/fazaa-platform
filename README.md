# Fazaa (فزعة) — Marketplace للمهنيين والعمالة المستقلة

## الرؤية
تطبيق سعودي يربط العملاء بالمهنيين (سباكة، كهرباء، مكيفات، تنظيف، نقل... إلخ) مع حجز/دفع/تتبع/تقييم، مع تركيز على السرعة والمصداقية.

## نطاق نسخة الإطلاق (MVP)
✔ تطبيق عميل (iOS + Android)
✔ تطبيق فني لاستقبال الطلبات والقبول والإنهاء
✔ API لإدارة المستخدمين والطلبات والمدفوعات

## التقنية (Stack)
- Mobile: Flutter (Dart)
- Backend API: NestJS (TypeScript) + PostgreSQL + Redis
- خرائط: Google Maps SDK
- دفع إلكتروني: HyperPay / Tap / Moyasar
- OTP: Unifonic / Twilio

## هيكلة المشروع (Monorepo)
apps/mobile        # تطبيق Flutter
services/api        # Backend (NestJS)
packages/common     # نماذج مشتركة
infra/              # Docker/CI/CD
docs/               # توثيق المتطلبات والواجهات والنموذج البياني

# إعداد الحساب السحابي والمزامنة — دفتر تداول

## ما أُضيف

- عميل Supabase للـ Expo مع PKCE وحفظ الجلسة في `expo-secure-store`.
- زر تسجيل Google وزر مزامنة لا يعملان إلا بطلب المستخدم من صفحة الإعدادات. لا توجد مهمة مزامنة خلفية.
- ترحيل PostgreSQL ينشئ `profiles`, `trading_accounts`, `trades`, `capital_events`, `sync_metadata` ويفعّل RLS وسياسات ملكية.
- نموذج Worker Cloudflare لحماية مسارات R2 برمز Supabase والتحقق من المستخدم من Auth API.
- إعداد EAS لتجميع APK داخلي.

## إعداد Supabase وGoogle OAuth

1. أنشئ مشروعاً في Supabase وسجّل عنوان المشروع وPublishable/anon key من **Project Settings → API**. لا تضع Service Role key في التطبيق.
2. افتح SQL Editor وشغّل `supabase/migrations/0001_cloud_sync.sql`.
3. من **Authentication → Providers → Google** فعّل Google وأدخل OAuth Client ID وClient Secret من Google Cloud Console. أضف redirect URI الذي يعرضه Supabase إلى قائمة Authorized redirect URIs لدى Google.
4. ضمن **Authentication → URL Configuration → Redirect URLs** أضف مخطط التطبيق متبوعاً بـ `://oauth/callback`؛ المخطط الفعلي هو قيمة `scheme` الناتجة من `app.config.ts` (افحص `npx expo config --json`).
5. انسخ `.env.example` إلى `.env` واملأ `EXPO_PUBLIC_SUPABASE_URL` و`EXPO_PUBLIC_SUPABASE_ANON_KEY`. تأكد أن المخطط المسجل لدى Supabase يطابق إعداد Expo. يعرض التطبيق رسالة إذا لم تُضبط المتغيرات.

## إعداد Cloudflare R2 والـ Worker

1. أنشئ Bucket خاصاً في Cloudflare R2. لا تفعّل الوصول العام.
2. أضف `cloudflare/worker/src/index.ts` إلى مشروع Worker بـ Wrangler، واربط binding اسمه `R2` بالـ bucket.
3. اضبط `SUPABASE_URL` في إعدادات Worker، وأضف Supabase publishable/anon key كـ Worker secret باسم `SUPABASE_ANON_KEY` (الكود الحالي يمرر المفتاح إلى Auth API؛ يلزم ربطه في Worker وإضافته إلى ترويسة `apikey`). لا تضف مفاتيح R2 للتطبيق.
4. انشر Worker واضبط `EXPO_PUBLIC_R2_WORKER_URL` للتطبيق.
5. يجب أن تبقى الصور خاصة، وأن تُرسل طلباتها بخانة `Authorization: Bearer <Supabase access token>`.

## بناء APK

1. ثبّت EAS CLI: `npm install -g eas-cli` ثم نفّذ `eas login`.
2. جهّز المتغيرات العامة في بيئة EAS أو محلياً (`.env` لا ترفعه إلى Git).
3. من مجلد المشروع: `eas build --platform android --profile preview`.
4. نزّل APK من رابط نتيجة EAS وثبته على جهاز Android للاختبار. للبناء المحلي يلزم إعداد Android SDK؛ EAS السحابي هو المسار الافتراضي هنا.

## تحذيرات مهمة قبل بيانات حقيقية

هذه نسخة تكامل أولية وليست جاهزة بعد للاستخدام الإنتاجي أو نقل بيانات مالية فعلية. لا تدّعِ نجاح الاختبارات التي لم تُنفذ. مخزن التطبيق المحلي الحالي عبارة عن جداول SQLite عامة لكل التطبيق ولا يفصل البيانات حسب UID؛ لا تسجل مستخدماً آخر على الجهاز قبل استكمال عزل/نقل قاعدة البيانات. المزامنة الحالية تنقل صفوف الحساب/الصفقات/حركات رأس المال على أساس `data` JSONB، لكنها لا تطبق بعد دفتر تغييرات محلياً أو tombstones للحذف، ولا تحل التعارض بطريقة تفاعلية (غياب معالج التعارض يرجّح النسخة الأحدث/السحابية)، ولا تنقل بايتات الصور إلى R2. لا توجد حتى الآن واجهة حذف حساب سحابية آمنة أو endpoint إداري لحذف Auth؛ لا تستخدمها لحذف بيانات إنتاجية.

كما ينبغي استكمال اختبار Worker عملياً، وتقييد CORS على النطاقات الضرورية، وإضافة حدود حجم موثوقة للرفع، وإنشاء روتين حذف الصور والحساب، وإتمام تطبيق المصادقة عبر callback بعد التحقق من Deep Link على Android حقيقي. سياسات RLS تقلل الوصول العرضي، لكنها لا تعوض اختبارات أمنية شاملة.

## التحقق

شغّل `pnpm check`, `pnpm test` و`pnpm lint`. لم تُجرَ اختبارات Google OAuth أو Supabase/R2 الحية من دون إعدادات مشروعك ومفاتيحك. لا تضع أي سر في Git أو APK.

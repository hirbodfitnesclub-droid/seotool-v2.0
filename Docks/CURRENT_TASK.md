# CURRENT_TASK.md — تمرکز فعلی سیستم

## ۱. درخت تمرکز (Focus Tree)
```
src/
├── services/quota/                          [پوشهٔ جدید — کانون فیچر F3]
│   ├── quotaService.ts                       [Pure: types + impressionWeight + applyQuotaLens + sort — تسک F3.1] ◀ شروع از اینجا
│   ├── quotaAllocationService.ts             [پیش‌محاسبهٔ سراسری + کش + signature — تسک F3.2]
│   └── quotaCsvService.ts                    [تمپلیت + پارس Zod — تسک F3.3]
├── services/pipeline/
│   └── lensPipeline.ts                       [ترکیب تک‌مرجع لایو+سهمیه — تسک F3.5 ◀ قلب فیچر]
├── contexts/
│   └── QuotaContext.tsx                      [state + persistence localStorage — تسک F3.4]
├── components/
│   └── QuotaBadge.tsx                        [نشانگر سهمیه/ایمپرشن — تسک F3.6]
├── pages/
│   ├── Config.tsx                            [سکشن تنظیمات سهمیه — تسک F3.6]
│   └── PageDetail.tsx                        [toggle + صدا زدن lensPipeline — تسک F3.5/F3.6]
├── services/analysis/
│   └── analysisService.ts                    [هم‌ترازی مسیر AI با نمایش — تسک F3.5]
└── core/queue/
    └── TaskExecutor.ts                       [اعمال لنز در مسیر صف — تسک F3.7 تکمیلی]
```

## ۲. جزئیات تکمیلی تسک فعلی (تسک F3.1 — هستهٔ سرویس Pure عینک سهمیه)

### عنوان و هدف
**تسک F3.1 — هستهٔ سرویس Pure عینک سهمیه (`quotaService.ts`)**
تعریف تایپ‌ها و منطق محاسباتی Pure لایهٔ سوم («عینک سهمیه سراسری»): محاسبهٔ ضریب کران‌دار `impressionWeight` (log-normalized)، تابع نرمال‌سازی عنوان برای تطبیق `H1 ↔ title`، فیلتر/وزن‌دهی per-source (`applyQuotaLens`)، و سورت اکید قطعی. این فایل **هیچ** وابستگی به Dexie/React/localStorage ندارد.

> شرح کامل: «شش قانون قطعی عینک سهمیه سراسری» در `PROJECT.md` و معماری کامل لایه در `ARCHITECTURE.md §۱۱`؛ نقشهٔ تسک‌های F3.1 تا F3.7، ترتیب اجرا و معیار پذیرش در `tasks.md → فیچر F3`.

### قیود بحرانی این فیچر (مرور سریع)
- **لایه سوم composable:** اول الگوریتم داخلی، بعد لایو، بعد سهمیه. لایه‌ها روی هم چیده می‌شوند، نه merge. `scorer.ts`/`idfCalculator.ts`/`db.ts`/اسکیمای Dexie دست‌نخورده.
- **کلید اتصال = عنوان نرمال‌شده** (`H1 ↔ page.title`)، چون اسکیمای `Page` فیلد `URL` ندارد و منجمد است؛ URL فقط متادیتای نمایشی.
- **سقف سراسری:** هر مقصد فقط در «بهترین» صفحاتِ مبدأ (بر اساس امتیاز خام یال) و حداکثر به‌اندازهٔ `quota = round(percentage/100 × totalInternalLinks)` ظاهر می‌شود؛ پس از آن حذف.
- **اولویت ایمپرشن کران‌دار:** ضریب حداکثر ~۲× تا ربط معنایی حاکم بماند.
- **تک‌مرجع بودن (درس F2.7):** نمایش و AI هر دو همان پایپ‌لاین را صدا می‌زنند؛ صفر منطق لنز در UI.

## ۳. وضعیت تسک‌ها

### فیچر F2 (Temporal/Live) — تکمیل‌شده
- **تسک F2.5 — UI تنظیمات Live در Config.tsx** [کامل ✅]
- **تسک F2.6 — Quick Toggle + اعمال Boost در PageDetail.tsx** [کامل ✅]
- **تسک F2.7 — دیباگ و یکپارچه‌سازی عینک لایو (پایپ‌لاین تک‌مرجع)** [کامل ✅]

### فیچر F3 (Global Quota & Impression Lens) — جدید
- **تسک F3.1 — هستهٔ سرویس Pure (`quotaService.ts`)** [در حال شروع 🚧 ◀ تسک فعال]
- **تسک F3.2 — تخصیص سراسری + کش (`quotaAllocationService.ts`)** [در صف ⏳]
- **تسک F3.3 — سرویس CSV سهمیه (`quotaCsvService.ts`)** [در صف ⏳]
- **تسک F3.4 — QuotaContext + persistence** [در صف ⏳]
- **تسک F3.5 — پایپ‌لاین ترکیبی تک‌مرجع + هم‌ترازی UI/AI** [در صف ⏳ — قلب فیچر]
- **تسک F3.6 — UI: Config + Quick Toggle + QuotaBadge** [در صف ⏳]
- **تسک F3.7 — اعمال لنز در مسیر صف دسته‌ای (TaskExecutor)** [تکمیلی ⏳]

---

## ۴. رله کانتکست (Context Relay)

### وضعیت پروژه و کارهای قبلی:
- فیچر F2 (عینک لایو/Temporal) کامل شد: کل منطق به پایپ‌لاین Pure تک‌مرجع `buildLiveOrderedList` بازگردانده شد و هر دو مسیر نمایش/AI همان را صدا می‌زنند. الگوریتم پایه و Dexie دست‌نخورده ماندند.

### شروع فیچر F3 (عینک سهمیه سراسری):
- **درخواست کاربر:** کنترل توزیع لینک‌های داخلی در سطح کل سایت. کاربر یک CSV با ستون‌های `URL، H1، Impressions، Percentage` و یک عدد دستی «کل لینک‌های داخلی سایت» (مثلاً ۳۶۸۷) وارد می‌کند. هر مقصد سهمیه‌ای دارد (`درصد × کل`) و باید فقط در بهترین صفحاتِ مبدأ تا سقف سهمیه ظاهر شود؛ پس از پر شدن، در بقیه نادیده گرفته شود. مقاصد پرایمپرشن باید کاندیدای قوی‌تری باشند.
- **تصمیمات معمار (منجمد):** (۱) لایهٔ سوم composable روی خروجی لایو/خام. (۲) چون اسکیما فیلد URL ندارد و منجمد است، کلید اتصال CSV به گراف = **عنوان نرمال‌شده**؛ URL فقط نمایشی. (۳) برای سقف سراسری نیاز به **پیش‌محاسبهٔ Reverse Index سراسری** با کش per-project است (آینهٔ `inlinkGraphService` با signature/chunking/invalidation). (۴) انتخاب مبدأها بر اساس **امتیاز خام یال** است؛ ایمپرشن نقش **بین‌مقصدی** دارد و از طریق `impressionWeight` کران‌دار اعمال می‌شود. (۵) تضمین فقط «سقف بالا» است (کم‌تخصیص مجاز، بیش‌تخصیص ممنوع)؛ بازتوزیع دقیق به فاز بعد موکول شد.
- **در تسک فعال (F3.1):** ساخت پوشهٔ `src/services/quota/` و فایل Pure `quotaService.ts` شامل تایپ‌ها (`QuotaRow`, `QuotaTargetInfo`, `QuotaAllocation`, `FinalCandidate`)، ثابت `IMPRESSION_BOOST_STRENGTH`، `computeImpressionWeights`، `normalizeTitle` (هم‌رفتار با `normalizeString` لایو)، `applyQuotaLens`، و `sortByFinalScore`. این تسک فقط منطق Pure است؛ ساخت Allocation و کش در تسک F3.2 می‌آید.

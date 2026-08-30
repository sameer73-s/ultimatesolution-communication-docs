# ADR-006 — ترحيل مزود قاعدة البيانات إلى PostgreSQL

| الحقل | القيمة |
|---|---|
| الحالة | **مقترح** في WBS 3.7؛ يصبح **مقبولًا** بعد اعتماد ودمج طلب السحب |
| التاريخ | 2026-08-30 |
| القرار | استبدال SQL Server بـ PostgreSQL عبر `Npgsql.EntityFrameworkCore.PostgreSQL` لتبسيط الاستضافة على Render |

## السياق

اعتمد الدليل المعماري في الإصدار 1.0 SQL Server مع Entity Framework Core Code-First. الخطوة WBS 3.7 تُحضّر استضافة الـAPI على Render. Render يوفّر PostgreSQL مُدارًا كخدمة أصلية، بينما استضافة SQL Server تتطلب مسارًا منفصلًا أكثر تعقيدًا. لا توجد بيانات إنتاجية يجب الحفاظ عليها في سلسلة هجرات SQL Server.

## القرار

- مزود البيانات المعتمد هو **PostgreSQL** + **EF Core Code-First**.
- معرفة المزود محصورة في `UltimateSolution.Infrastructure` عبر `UseNpgsql` وحزمة `Npgsql.EntityFrameworkCore.PostgreSQL`.
- تُحذف هجرات SQL Server (`nvarchar` و`uniqueidentifier` و`SqlServer:Identity`) ويُعاد توليد هجرة أولية واحدة متوافقة مع PostgreSQL.
- يُضاف `render.yaml` يصف خدمة Web وقاعدة PostgreSQL دون نشر فعلي في هذه الخطوة.

لا يتغير اتجاه الاعتماد ولا عقود Application. تبقى الاختبارات على قاعدة InMemory في بيئة `Testing`.

## العلاقة بالقرارات السابقة

| ADR | العلاقة |
|---|---|
| ADR-001 | لا تغيير. Jitsi يبقى خلف `IMeetingMediaService` داخل Infrastructure. |
| ADR-002 | لا تغيير. سياسة اعتماد الملخص تبقى في Application. |
| ADR-003 | لا تغيير. يبقى الوسيط `Mediator` بترخيص MIT؛ يُحظر MediatR/LuckyPenny. |
| ADR-004 | لا تغيير. حدود تكامل الذكاء الاصطناعي كما هي. |
| ADR-005 | لا تغيير. توثيق JWT في OpenAPI كما هو. |

## غير مشمول

النشر الفعلي على Render، واختيار الخطة التجارية والمنطقة، وإدارة أسرار الإنتاج: كلها مؤجلة لخطوة لاحقة بعد اعتماد هذا القرار.

التنفيذ التفصيلي ومراجعة الهجرات موجودان في مستودع Backend: `docs/adr/ADR-006-postgresql-migration.md` و`docs/database-provider-migration.md`.

# دليل معماري (Architecture Guideline)
## مشروع منصة التواصل المؤسسي – Ultimate Solution
**الإصدار:** 1.1
**الغرض:** هذا المستند هو المرجع الثابت الذي يُعاد استخدامه في كل مهمة تُرسل إلى Manus. أي كود أو تصميم يُنتَج في هذا المشروع **يجب** أن يلتزم بما هو موثّق هنا، ولا يجوز الانحراف عنه دون موافقة صريحة.

---

## 1. نظرة عامة على المشروع

منصة تواصل مؤسسي داخلية (بديل لـ Microsoft Teams) تخدم التواصل بين الموظفين والمدراء داخل شركة Ultimate Solution، مع ميزة تنافسية أساسية: **الذكاء الاصطناعي لتلخيص وإدارة الاجتماعات**، وتوسّع مستقبلي ليشمل البريد الإلكتروني.

### المكونات الرئيسية
1. مراسلة فورية (Chat) + حالة الحضور (Presence)
2. اجتماعات صوتية/مرئية (Meetings)
3. تفريغ صوتي وتلخيص ذكي للاجتماعات + استخراج المهام (AI Meeting Intelligence)
4. إدارة الاجتماعات (جدولة، أجندة، متابعة)
5. (مستقبلاً) بريد إلكتروني مدمج

---

## 2. التقنيات المعتمدة (Tech Stack) — ثابتة ولا تتغير

| الطبقة | التقنية | ملاحظات |
|---|---|---|
| Backend API | **ASP.NET Core Web API** (أحدث LTS) | Clean Architecture |
| قاعدة البيانات | **PostgreSQL** + **Entity Framework Core** (Code-First) عبر `Npgsql.EntityFrameworkCore.PostgreSQL` | **ADR-006:** استُبدل SQL Server لتبسيط الاستضافة على Render. معرفة المزود محصورة في Infrastructure. |
| الاتصال الفوري | **SignalR** | للشات، الحضور، وإشارات المكالمات (signaling) |
| المكالمات | **Jitsi** (قرار معتمد للمرحلة الحالية) خلف تجريد كامل `IMeetingMediaService` في طبقة Application | **إلزامي:** لا يوجد أي اعتماد مباشر على Jitsi خارج `Infrastructure`. الهدف: استبدال لاحق بمحرك WebRTC مخصص بالكامل (عندما يُطلب تحكم كامل في تجربة الوسائط) دون أي تعديل على Application أو API أو Flutter. أي كود يستدعي وظائف الاجتماع (بدء/إنهاء/انضمام/تسجيل) يمر حصراً عبر هذه الواجهة |
| المصادقة | **ASP.NET Identity + JWT** | صلاحيات (Roles): Admin / Manager / Employee |
| الذكاء الاصطناعي | خدمة منفصلة (Microservice/API خارجي) — Whisper أو Azure Speech للتفريغ، Claude API للتلخيص واستخراج المهام | لا تُدمج داخل الـ API الرئيسي، بل تُستدعى كخدمة مستقلة |
| Frontend | **Flutter** | Clean Architecture + Bloc + Feature-based |
| إدارة الحالة | **flutter_bloc** | حصراً — لا Provider ولا GetX |
| Dependency Injection (Flutter) | **get_it** (+ `injectable` إن لزم) | |
| HTTP Client (Flutter) | **Dio** | مع Interceptors للتوكن والأخطاء |
| البريد (مستقبلي) | **MailKit** / **Microsoft Graph API** | لا يُبنى الآن، فقط تُترك نقاط تمديد (extension points) |

---

## 3. معمارية الـ Backend (ASP.NET Clean Architecture)

### هيكل المجلدات (الحل/Solution)

```
UltimateSolution.Communication/
│
├── src/
│   ├── UltimateSolution.Domain/              # Entities, Enums, Domain Exceptions, Interfaces
│   │   ├── Entities/
│   │   ├── Enums/
│   │   └── Interfaces/
│   │
│   ├── UltimateSolution.Application/         # Use Cases, DTOs, Validators, CQRS (MediatR)
│   │   ├── Features/
│   │   │   ├── Auth/
│   │   │   ├── Chat/
│   │   │   ├── Meetings/
│   │   │   └── AiSummary/
│   │   ├── Common/ (Behaviors, Mappings - AutoMapper)
│   │   └── Interfaces/  (IApplicationDbContext, IAiService, ...)
│   │
│   ├── UltimateSolution.Infrastructure/      # EF Core, تنفيذ الـ Interfaces، خدمات خارجية
│   │   ├── Persistence/ (DbContext, Migrations, Repositories)
│   │   ├── Identity/
│   │   ├── SignalR/ (Hubs)
│   │   └── ExternalServices/ (AI, Storage, Email لاحقاً)
│   │
│   └── UltimateSolution.API/                 # Controllers, Middlewares, Program.cs
│       ├── Controllers/
│       ├── Hubs/  (أو تُستدعى من Infrastructure)
│       └── Middlewares/
│
└── tests/
    ├── UltimateSolution.Application.Tests/
    └── UltimateSolution.API.IntegrationTests/
```

### قواعد صارمة (لا استثناء):
- **اتجاه الاعتماد (Dependency Rule):** `API → Application → Domain` و`Infrastructure → Application/Domain`. الـ Domain لا يعتمد على أي طبقة أخرى إطلاقاً.
- استخدام نمط **CQRS مع MediatR** لكل Use Case (Command/Query منفصلين)
- **Repository Pattern + Unit of Work** بين Application و Infrastructure
- كل Feature (Chat, Meetings, AiSummary...) له مجلد مستقل داخل `Application/Features`
- التحقق من المدخلات عبر **FluentValidation**
- الأخطاء تُعاد كـ `Result<T>` أو عبر Exception Middleware موحّد — لا يُرمى Exception خام للـ Controller
- كل Endpoint موثّق عبر **Swagger/OpenAPI**
- الاتصال بقاعدة البيانات حصراً عبر `IApplicationDbContext` (لا DbContext مباشر في Application)
- **قرار معماري ثابت — اعتماد ملخص AI (ADR-002):** الإصدار الأول يخوّل المنظم أو `Manager` فقط باعتماد `MeetingSummary` عبر `ApproveMeetingSummaryCommand`. **يُبنى هذا التفويض كسياسة قابلة للتوسعة (Authorization Policy) في Application، لا كفحص دور ثابت داخل الـ Handler**، لضمان إمكانية إضافة دور مراجعة إضافي مستقبلاً (مثل Reviewer من الموارد البشرية) بإضافة قاعدة إلى السياسة فقط، دون تعديل الـ Command أو الـ Entity أو منطق الأعمال.
- **قرار معماري ثابت — طبقة وسائط الاجتماعات:** يُعرَّف `IMeetingMediaService` في `Application/Interfaces` بعمليات مجردة بالكامل (بدء اجتماع، إنهاء اجتماع، انضمام/مغادرة مشارك، بدء/إيقاف تسجيل). التنفيذ الحالي `JitsiMeetingMediaService` يقع حصراً في `Infrastructure/ExternalServices`. **ممنوع** أي استدعاء مباشر لـ Jitsi (SDK أو API) من أي طبقة غير Infrastructure. الغرض: استبدال لاحق بمحرك WebRTC مخصص بالكامل (لتحكم كامل في تجربة الوسائط) عبر إضافة `WebRtcMeetingMediaService` جديد ينفّذ نفس الواجهة، دون أي تعديل على Application أو API أو Flutter.

---

## 4. معمارية الـ Frontend (Flutter Clean Architecture + Bloc + Feature-based)

```
lib/
├── core/
│   ├── constants/
│   ├── errors/            # Failures, Exceptions
│   ├── network/           # Dio client, Interceptors, NetworkInfo
│   ├── theme/             # هوية Ultimate Solution (راجع القسم 5)
│   ├── utils/
│   ├── routing/           # go_router أو auto_route
│   └── widgets/           # Shared/reusable widgets
│
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/    # remote_datasource.dart
│   │   │   ├── models/         # DTOs (توسّع من entities)
│   │   │   └── repositories/   # implementation
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   ├── repositories/   # abstract contracts
│   │   │   └── usecases/
│   │   └── presentation/
│   │       ├── bloc/           # bloc, event, state (كل ملف منفصل)
│   │       ├── pages/
│   │       └── widgets/
│   │
│   ├── chat/            # نفس البنية بالضبط
│   ├── meetings/        # نفس البنية بالضبط
│   ├── ai_summary/      # نفس البنية بالضبط
│   └── ...
│
└── injection_container.dart   # get_it setup
```

### قواعد صارمة (لا استثناء):
- الطبقات الثلاث (data / domain / presentation) **معزولة تماماً** — الـ Presentation لا يستدعي Data مباشرة، فقط عبر UseCases من Domain
- **Domain لا يعتمد على Flutter SDK** (طبقة Dart خالصة قابلة لإعادة الاستخدام واختبارها بمعزل)
- كل Feature تحتوي Bloc خاص بها، بأسماء صريحة: `AuthBloc`, `AuthEvent`, `AuthState` (استخدام `freezed` أو `equatable` مقبول)
- كل استدعاء API يمر عبر `RemoteDataSource` ثم `Repository Implementation` ثم `UseCase`
- معالجة الأخطاء عبر `Either<Failure, T>` (dartz) أو نمط مشابه موحّد في كل الـ Features
- لا `setState` لإدارة حالة الأعمال (Business Logic) — فقط لتفاصيل UI بحتة إن لزم
- كل Feature **مستقلة قابلة للحذف** دون كسر بقية المشروع

---

## 5. الهوية البصرية (Ultimate Solution Branding)

> **الحالة:** قيد الاستخراج آلياً بواسطة Manus من الموقعين الرسميين (راجع المرحلة 5.1 في "البرومبت الرئيسي"):
> - https://ultimatesolutionsportal.com/ (شعار WordPress — ملاحظة: يوجد ملف شعار بصيغة PNG على الموقع)
> - https://www.ultimate-erp.com/ (شعار Wix — نسخة شعار مختلفة قليلاً، يجب مقارنتها بالأولى)
>
> حتى اعتماد "Brand Extraction Report"، يُستخدم نظام ألوان مؤقت محايد (placeholder) قابل للاستبدال المركزي عبر `core/theme/`. **لا يُعتمد أي لون/شعار نهائياً إلا بعد مراجعتنا للتقرير المستخرج.**

العناصر المطلوب حسمها (تُملأ تلقائياً بعد التقرير):
- [ ] الشعار النهائي المعتمد (وحسم أي تعارض بين نسخة الموقعين)
- [ ] اللون الأساسي (Primary) — Hex
- [ ] اللون الثانوي/المميز (Accent) — Hex
- [ ] ألوان النصوص/الخلفيات القياسية — Hex
- [ ] الخط الرسمي (Font Family) وحالة توفره كخط مفتوح
- [ ] أي دليل هوية بصرية رسمي سابق إن وُجد لدى الشركة (يُطلب من الإدارة مباشرة إن لم يكن مستخرَجاً من الموقعين بشكل كافٍ)

عند التوفر، تُطبَّق حصراً عبر:
- Backend: قوالب البريد/العروض التي يولّدها النظام لاحقاً
- Frontend: `core/theme/app_colors.dart`, `app_text_styles.dart`, `app_theme.dart` — **مصدر واحد للحقيقة (Single Source of Truth)**، ممنوع hardcoding الألوان داخل الـ Widgets

---

## 6. عقد التكامل بين Backend و Frontend (API Contract)

- كل Response من الـ API يتبع شكل موحّد:
```json
{
  "success": true,
  "data": { },
  "message": "string",
  "errors": []
}
```
- الأخطاء تتبع رموز HTTP قياسية (400, 401, 403, 404, 500) مع رسالة واضحة بالحقل `errors`
- التوثيق الحي عبر Swagger يُستخدم كمرجع وحيد لتوليد الـ Models في Flutter (تجنّب أي افتراض غير موثّق)

---

## 7. معايير الجودة العامة (تنطبق على Backend و Frontend معاً)

1. تسمية واضحة ومتسقة بالإنجليزية لكل الكود (لا عربي في الكود، فقط في التعليقات إن لزم)
2. لا كود مكرر (DRY) — أي منطق مشترك يُرفع لطبقة `core`/`Common`
3. كل Feature جديدة تُبنى بنفس القالب الهيكلي تماماً دون ابتكار بنية مختلفة
4. اختبارات وحدة (Unit Tests) للـ Use Cases والـ Bloc كحد أدنى
5. Git commits واضحة ومقسّمة منطقياً (لا commit ضخم واحد لكل مرحلة)
6. لا اعتماديات (packages/NuGet) جديدة دون تبرير واضح في ملاحظات التسليم

---

## 8. سياسة الالتزام بالمراحل

هذا المشروع يُنفَّذ **حصراً على مراحل متسلسلة** (موثّقة في مستند "البرومبت الرئيسي"). **يُمنع** البدء بمرحلة تالية قبل اعتماد المرحلة الحالية من قبل فريق المراجعة (Claude + المطور المسؤول).

## 9. إدارة المصدر (Git / GitHub)

| العنصر | القرار |
|---|---|
| بنية المستودعات | **مستودعان منفصلان:** `ultimatesolution-communication-backend` (حل ASP.NET) و`ultimatesolution-communication-mobile` (مشروع Flutter)، بالإضافة إلى `ultimatesolution-communication-docs` كمرجع حي للدليل المعماري وADRs والبرومبت الرئيسي ومخرجات التصميم |
| الفرع الرئيسي | `main` محمي، بلا Push مباشر |
| استراتيجية الفروع | فرع مستقل لكل خطوة فرعية من WBS، بنفس الترقيم: `feature/<WBS-ID>-<اسم-مختصر>` (مثال: `feature/3.1-backend-foundation`) |
| بوابة الاعتماد | كل خطوة فرعية تُسلَّم عبر **Pull Request**، وليس كملف مباشر. قالب PR يتضمن رقم WBS والمخرج المتوقع ومعيار القبول |
| الدمج في `main` | **حصري بعد اعتماد صريح** من فريق المراجعة — الدمج نفسه يمثل بوابة الانتقال للخطوة التالية، بنفس منطق "معتمد، انتقل للخطوة التالية" |
| Manus لا يدمج بنفسه | يفتح PR وينتظر الدمج من الطرف المعتمِد فقط |

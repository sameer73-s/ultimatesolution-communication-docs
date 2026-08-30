# مخرجات المرحلة 1 — التصميم الهيكلي

**المشروع:** منصة التواصل المؤسسي — Ultimate Solution  
**الحالة:** مكتمل للمراجعة والاعتماد؛ لا يتضمن تنفيذًا برمجيًا.  
**تاريخ التحديث:** 27 أغسطس 2026

## ما تم تثبيته

تلتزم البنية المقترحة بـ ASP.NET Core Web API وPostgreSQL وEntity Framework Core وSignalR وASP.NET Identity + JWT في الخلفية، وبـ Flutter و`flutter_bloc` و`Dio` و`get_it` في العميل. تتبع الخلفية Clean Architecture وCQRS/Mediator وRepository + Unit of Work، مع وضع كل Feature في نطاقه المنفصل. يوثق ADR-006 اعتماد PostgreSQL بدل SQL Server لتبسيط الاستضافة على Render.

تم اعتماد **Jitsi** لطبقة وسائط الاجتماعات في المرحلة الحالية، مع قيد معماري ملزم: يُعرّف `IMeetingMediaService` داخل Application، وتبقى كل معرفة مباشرة بـ Jitsi وتنفيذه داخل Infrastructure فقط. وقد صُمم النموذج وعقود API ومخططات التسلسل باستخدام أسماء عامة عن المزوّد، بحيث يمكن تسجيل `WebRtcMeetingMediaService` مستقبلاً من دون تغيير Application أو API أو Flutter.

> **حد المرحلة:** هذه المخرجات تصميم قابل للمراجعة. لا تبدأ المرحلة 2 أو أي إنشاء للمشروع أو كود Backend/Flutter قبل اعتماد هذه المرحلة صراحةً.

## ملفات التسليم

| الملف | المحتوى | الغرض من المراجعة |
|---|---|---|
| [`diagrams/erd.mmd`](diagrams/erd.mmd) | مصدر Mermaid لمخطط الكيانات والعلاقات | مراجعة الجداول والعلاقات والمفاتيح ونقاط الامتداد. |
| [`diagrams/erd.png`](diagrams/erd.png) | نسخة مرسومة من ERD | قراءة سريعة للعلاقات في جلسة المراجعة. |
| [`diagrams/architecture.mmd`](diagrams/architecture.mmd) | مصدر Mermaid للمعمارية الطبقية | التحقق من اتجاه الاعتماد وعزل Jitsi. |
| [`diagrams/architecture.png`](diagrams/architecture.png) | نسخة مرسومة من المعمارية | مراجعة حدود الطبقات والتكاملات الخارجية. |
| [`diagrams/sequence_meeting_lifecycle.mmd`](diagrams/sequence_meeting_lifecycle.mmd) | تسلسل دورة الاجتماع | مراجعة البدء والانضمام والتسجيل والإنهاء. |
| [`diagrams/sequence_meeting_lifecycle.png`](diagrams/sequence_meeting_lifecycle.png) | نسخة مرسومة من التسلسل | تدقيق التدفق وقيد عدم الاعتماد على Jitsi خارج Infrastructure. |
| [`diagrams/sequence_ai_meeting_intelligence.mmd`](diagrams/sequence_ai_meeting_intelligence.mmd) | تسلسل معالجة التسجيل والذكاء الاصطناعي | مراجعة الطابور والتفريغ والملخص والمهام والإشعارات. |
| [`diagrams/sequence_ai_meeting_intelligence.png`](diagrams/sequence_ai_meeting_intelligence.png) | نسخة مرسومة من تسلسل الذكاء | تدقيق نقطة مراجعة المخرجات قبل إنشاء المهام. |
| [`data_model_design.md`](data_model_design.md) | قاموس النموذج وقواعده | مراجعة القيود والفهارس ونقاط تمديد البريد والـAI. |
| [`architecture_decisions.md`](architecture_decisions.md) | ADR-001 وADR-002 | اعتماد حد Jitsi وواجهة الوسائط وسياسة مراجعة مخرجات AI. |
| [`api/initial_api_contract.md`](api/initial_api_contract.md) | مسارات HTTP وSignalR والغلاف الموحد | مراجعة الأدوار، الأوامر/الاستعلامات، واستجابات API. |

## الملخص المعماري

| الطبقة | المسؤولية | قاعدة الاعتماد |
|---|---|---|
| `UltimateSolution.Domain` | الكيانات والـEnums والاستثناءات وقواعد المجال | لا تعتمد على أي طبقة أو إطار عمل أو مزوّد خارجي. |
| `UltimateSolution.Application` | Use Cases وCQRS/MediatR وDTOs وFluentValidation وواجهات التجريد | تعتمد على Domain فقط؛ تستخدم الواجهات لا التنفيذات. |
| `UltimateSolution.Infrastructure` | EF Core وIdentity وSignalR والـRepositories والـAdapters الخارجية | تنفذ واجهات Application وتتكامل مع PostgreSQL وJitsi وخدمات AI. |
| `UltimateSolution.API` | Controllers وMiddleware وSwagger/ OpenAPI واستضافة SignalR | يمرر الطلبات إلى Application؛ لا يحتوي منطق أعمال أو SDK خاصًا بالمزوّد. |
| Flutter | العرض وBloc واستخدام Use Cases للعميل وشبكة Dio | يستهلك REST وSignalR بعقود عامة، ولا يعرف Jitsi مباشرة. |

### قيد Jitsi الذي يجب التدقيق عليه

```text
Application/Interfaces/IMeetingMediaService
    ↑ implemented by
Infrastructure/ExternalServices/Meetings/JitsiMeetingMediaService
    ↓ only direct Jitsi SDK/API boundary
Jitsi Media Layer
```

لا يجوز أن تظهر أسماء Jitsi في Controllers أو Hubs أو Commands أو Queries أو DTOs أو كيانات Domain أو حزم Flutter أو أسماء إعدادات API. يكتفي `Program.cs` باستدعاء امتداد Infrastructure العام، بينما يبقى ربط التنفيذ الفعلي داخل مشروع Infrastructure.

## نقاط المراجعة المطلوبة

| القرار أو السؤال | الحالة في التصميم | المطلوب للاعتماد |
|---|---|---|
| طبقة الوسائط | **مُعتمد:** Jitsi خلف `IMeetingMediaService` | التحقق من التزام جميع الفرق بحد العزل الوارد في ADR-001. |
| دورة اعتماد ملخص AI | **مُصمم:** Draft ثم مراجعة ثم اعتماد ثم مهام مؤكدة | اعتماد `IMeetingSummaryApprovalPolicy` كسياسة Application قابلة للتوسعة؛ القاعدة الحالية تشمل المنظم أو `Manager` من دون فحص دور ثابت داخل الـHandler. |
| احتفاظ التسجيلات والنصوص | مفتوح عمدًا | اعتماد مدة الاحتفاظ، التصنيف، والتشفير قبل المهاجرات. |
| التخزين والطابور | مفتوح عمدًا | اختيار Object Storage وBackground Job Queue في بداية Backend Infrastructure. |
| مولد Swagger | محدد كمصدر تعاقد حي | اعتماد أن Flutter يولد نماذجه من OpenAPI عند التنفيذ لا من افتراضات يدوية. |
| البريد الإلكتروني | نقطة تمديد فقط | عدم إدراج MailKit أو Microsoft Graph في نطاق المرحلة الحالية. |

## شروط الانتقال إلى المرحلة 2

تنتقل المرحلة 2 إلى إعداد ملف إدارة المشروع وWBS والخطة الزمنية والمخاطر بعد اعتماد هذه المخرجات. ويُعدّ القبول كاملاً عندما يقر فريق المراجعة بصحة ERD، وحدود الطبقات، وتسلسل دورة الاجتماع والذكاء الاصطناعي، وعقود API، وقيد Jitsi المتكرر في ADR والمخطط والتسلسل.

# سجل القرارات المعمارية — المرحلة 1

## ADR-001: Jitsi خلف تجريد Application كامل

| الحقل | القرار |
|---|---|
| الحالة | **معتمد** من فريق المشروع للمرحلة الحالية. |
| السياق | تحتاج المنصة إلى اجتماعات صوتية/مرئية وتسجيل، مع إبقاء خيار الانتقال لاحقًا إلى محرك WebRTC مخصص دون إعادة بناء طبقات الأعمال أو العميل. |
| القرار | تعتمد المرحلة الحالية Jitsi **حصريًا** من خلال `JitsiMeetingMediaService` داخل `UltimateSolution.Infrastructure/ExternalServices/Meetings`. تُعرّف واجهة `IMeetingMediaService` في `UltimateSolution.Application/Interfaces`. |
| النتيجة | تتعامل Use Cases وControllers وHubs وFlutter مع عقود عامة لا تعرف اسم المزوّد. يمكن مستقبلاً إضافة `WebRtcMeetingMediaService` يطبق الواجهة ذاتها. |

> **قاعدة إلزامية:** يمنع وجود أي اعتماد مباشر على Jitsi، بما في ذلك SDK أو API أو أنواع DTO أو أسماء إعدادات أو أسماء حقول مزوّد-محددة، في Domain أو Application أو API أو Flutter. الاستثناء الوحيد هو Infrastructure، وبالتحديد Adapter الخاص بالوسائط وإعداداته الداخلية.

## عقد `IMeetingMediaService`

يوضع العقد في `UltimateSolution.Application/Interfaces/IMeetingMediaService.cs`. الأسماء أدناه جزء من التصميم لا تنفيذ جاهز؛ تستخدم أسماء إنجليزية متسقة وتعيد أنواعًا عامة لا تكشف Jitsi أو WebRTC.

```csharp
public interface IMeetingMediaService
{
    Task<Result<MeetingMediaSession>> StartMeetingAsync(
        StartMeetingMediaRequest request,
        CancellationToken cancellationToken);

    Task<Result> EndMeetingAsync(
        EndMeetingMediaRequest request,
        CancellationToken cancellationToken);

    Task<Result<JoinMeetingResult>> JoinParticipantAsync(
        JoinMeetingParticipantRequest request,
        CancellationToken cancellationToken);

    Task<Result> LeaveParticipantAsync(
        LeaveMeetingParticipantRequest request,
        CancellationToken cancellationToken);

    Task<Result<RecordingResult>> StartRecordingAsync(
        StartRecordingRequest request,
        CancellationToken cancellationToken);

    Task<Result<RecordingResult>> StopRecordingAsync(
        StopRecordingRequest request,
        CancellationToken cancellationToken);
}
```

| العقد العام | محتوى مقترح | محظورات صريحة |
|---|---|---|
| `StartMeetingMediaRequest` | `MeetingId`, `OrganizerUserId`, `ScheduledStartUtc`, `ParticipantUserIds` | `JitsiRoomName`, `JitsiJwt`, `JitsiDomain` أو أي إعداد مزوّد. |
| `MeetingMediaSession` | `MediaSessionReference`, `ExpiresAtUtc`, `Status` | نوع مزوّد أو SDK أو رابط يتطلب معالجة عميل خاصة. |
| `JoinMeetingParticipantRequest` | `MeetingId`, `UserId`, `ParticipantRole` | Token أو Claim باسم مزوّد محدد. |
| `JoinMeetingResult` | `MediaSessionReference`, `MediaJoinUrl`, `ExpiresAtUtc` | اسم غرفة Jitsi أو نوع JWT أو أي خاصية تجعل Flutter متصلة بمزوّد بعينه. |
| `StartRecordingRequest` و`StopRecordingRequest` | `MeetingId`, `RequestedByUserId`, `MediaSessionReference` | مُعرّف تسجيل أو قناة باسم مزوّد محدد داخل Application. |
| `RecordingResult` | `MediaRecordingReference`, `Status`, `AvailableAtUtc` | اسم مزوّد أو صيغة مسار تخزين خاصة بمزوّد داخل العقد. |

## توزيع المسؤوليات

| الطبقة | ما يجوز لها معرفته | ما لا يجوز لها معرفته |
|---|---|---|
| Domain | حالة الاجتماع، المشاركون، سياسات صلاحية عامة، ومراجع وسائط معتمة | Jitsi، WebRTC، SDK، URL غرفة، Token، أو بروتوكول تسجيل. |
| Application | الواجهة العامة، الطلبات والنتائج العامة، Use Cases، التفويض، ونتائج الأعمال | أي نوع أو اسم إعداد أو Endpoint أو Package خاص بـ Jitsi. |
| Infrastructure | تنفيذ `JitsiMeetingMediaService`، تكوين Jitsi، ترجمة الطلبات/النتائج، وتخزين أسرار المزوّد | منطق قرار الأعمال أو صلاحية المستخدم النهائية. |
| API | Controllers وJWT وSwagger وتمرير Commands/Queries؛ واستدعاء `AddInfrastructure(configuration)` العام | استدعاء SDK/API لـ Jitsi أو إنشاء `JitsiMeetingMediaService` بالاسم. |
| Flutter | `mediaJoinUrl` و`mediaSessionReference` العامة، وعرض حالات الاجتماع | SDK أو API أو اسم أو إعداد خاص بـ Jitsi. |

## التسجيل عبر Dependency Injection

يوضع ربط التنفيذ في نقطة إنشاء Infrastructure، مثل امتداد `AddInfrastructure`. يستدعي API هذا الامتداد العام عند بناء المضيف، ولا يسجل أو يستورد نوع `JitsiMeetingMediaService` بالاسم. بهذه الطريقة يبقى المرجع الخاص بالمزوّد محصورًا في مشروع Infrastructure حتى في تكوين DI.

```text
UltimateSolution.API
    -> services.AddInfrastructure(configuration)
        -> Infrastructure registers IMeetingMediaService
            -> JitsiMeetingMediaService
                -> Jitsi SDK / API
```

## قواعد فحص قبل بدء التنفيذ

| الفحص | معيار القبول |
|---|---|
| مراجع المشاريع | لا تشير مشاريع Domain أو Application أو API أو Flutter إلى أي Package أو Namespace خاص بـ Jitsi. |
| أسماء العقود | لا تحتوي DTOs وCommands وResponses وكيانات المجال على بادئة أو لاحقة Jitsi. |
| حدود DI | لا يظهر `JitsiMeetingMediaService` في `Program.cs` أو Controllers؛ يظهر التسجيل داخل Infrastructure فقط. |
| تدفق الوسائط | تستدعي Handlers `IMeetingMediaService` فقط، وتتحقق من `Result<T>` قبل تحديث حالة `Meeting`. |
| قابلية الاستبدال | يمكن تسجيل `WebRtcMeetingMediaService` لاحقًا في Infrastructure بنفس واجهة Application من دون تعديل على Application أو API أو Flutter. |

## ADR-002: مخرجات الذكاء الاصطناعي قابلة للمراجعة وسياسة اعتماد قابلة للتوسعة

تُحفظ نتائج التلخيص واستخراج المهام أولًا في `MeetingSummary` بحالة `Draft`. لا ينفذ `ApproveMeetingSummaryCommand` فحصًا ثابتًا من قبيل `Organizer` أو `Manager` داخل الـHandler. بدلاً من ذلك، تستدعي طبقة Application سياسة تفويض مجردة وقابلة للتوسعة، مثل `IMeetingSummaryApprovalPolicy`، للتحقق مما إذا كان المستخدم الحالي يملك صلاحية اعتماد الملخص في سياق الاجتماع المحدد.

```csharp
public interface IMeetingSummaryApprovalPolicy
{
    Task<Result> AuthorizeAsync(
        MeetingSummaryApprovalAuthorizationRequest request,
        CancellationToken cancellationToken);
}
```

| المكوّن | المسؤولية |
|---|---|
| `ApproveMeetingSummaryCommandHandler` | يحمّل الاجتماع والملخص، ويطلب قرار التفويض من السياسة، ثم يحوّل الملخص إلى `Approved` وينشئ المهام المؤكدة فقط عند نجاح السياسة. |
| `IMeetingSummaryApprovalPolicy` في Application | عقد التفويض العام الذي لا يحوي فحص Role ثابتًا ولا يعتمد على Controller أو SDK أو مزوّد خارجي. |
| تنفيذ السياسة في Infrastructure | يقيّم القاعدة الحالية: منظم الاجتماع أو صاحب دور `Manager`. يمكن لاحقًا إضافة مراجع موارد بشرية أو قاعدة تفويض قسمية بتعديل التنفيذ أو قواعد السياسة فقط. |
| API | يمرر هوية المستخدم الحالية وسياق الطلب إلى Command؛ لا يقرر في Controller من يحق له الاعتماد. |

> **قاعدة إلزامية:** إضافة دور مراجعة مستقبلي، مثل `Reviewer` في الموارد البشرية، يجب أن تتطلب إضافة قاعدة إلى سياسة التفويض أو تنفيذها فقط. يمنع تعديل `ApproveMeetingSummaryCommand` أو `MeetingSummary` أو منطق الأعمال لاستخدام فحص دور ثابت جديد.

يقلل القرار خطر تحويل مخرجات احتمالية إلى التزامات تشغيلية غير مراجعة، ويحافظ على قابلية تمديد حوكمة الاعتماد دون إعادة تصميم المنطق الأساسي.

## ADR-006: ترحيل مزود قاعدة البيانات إلى PostgreSQL

يُستبدل SQL Server بـ PostgreSQL عبر `Npgsql.EntityFrameworkCore.PostgreSQL` لتبسيط الاستضافة على Render. معرفة المزود محصورة في Infrastructure، وتُعاد توليد هجرة أولية واحدة بعد حذف أنواع SQL Server من السلسلة السابقة. التفاصيل الكاملة في [ADR-006-postgresql-migration.md](ADR-006-postgresql-migration.md). لا يشمل القرار النشر الفعلي على Render.

## قرارات ما زالت مفتوحة قبل التنفيذ

لا تحسم المرحلة 1 العناصر التالية. ينبغي اعتمادها قبل الإنتاج ضمن إعدادات Infrastructure أو الخدمات المتخصصة، ولا ينبغي افتراضها ضمن Domain أو Application أو Flutter.

| القرار أو البند المفتوح | الحالة المطلوبة قبل الإنتاج |
|---|---|
| طابور الخلفية، مزوّد Object Storage، سياسة الاحتفاظ بالتسجيلات والنصوص، وآلية إعادة المحاولة | اعتماد التصميم التشغيلي وإعدادهما في Infrastructure أو الخدمة المختصة. |
| **استمرارية جلسات وتسجيلات الوسائط بعد إعادة تشغيل التطبيق** | **مطلوب قبل الإنتاج:** لا يجوز أن يعتمد `JitsiMeetingMediaService` على `_sessions` و`_recordings` داخل الذاكرة فقط. يجب إعادة بناء `RoomName` حتميًا من `MeetingId` و`MediaSessionReference` المحفوظين، وجعل حالة التسجيل قابلة للاستعلام والتحكم من مزوّد وسائط موزّع أو مخزن حالة دائم. يجب التحقق من الحالة الفعلية قبل أي انتقال دائم لحالة الاجتماع أو التسجيل. |

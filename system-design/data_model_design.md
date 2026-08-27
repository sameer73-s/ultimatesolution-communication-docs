# تصميم نموذج البيانات — المرحلة 1

## مبدأ التقسيم

يقسم النموذج إلى حدود وظيفية واضحة: **Identity** للمستخدمين والأدوار، و**Chat** للقنوات والرسائل، و**Meetings** للجدولة والحضور والتسجيل، و**AI Meeting Intelligence** لوظائف التفريغ والملخص والمهام، و**Notifications/Audit** للأثر التشغيلي والحوكمة. يعتمد كل معرفٍ من معرفات المجال على `Guid`، باستثناء معرف مستخدم ASP.NET Identity الذي يبقى `string` لضمان التوافق مع هوية الإطار.

| المجال | الجداول المحورية | مسؤولية البيانات |
|---|---|---|
| Identity | `AppUser`, `UserProfile`, `Role`, `UserRole` | المصادقة، الأدوار، الملف التعريفي، وحالة المستخدم. |
| Chat | `ChatChannel`, `ChannelMembership`, `ChatMessage`, `MessageAttachment`, `MessageReadReceipt` | القنوات، العضوية، الرسائل، المرفقات، وآخر قراءة أو إيصال قراءة. |
| Meetings | `Meeting`, `MeetingParticipant`, `MeetingAgendaItem`, `MeetingRecording` | موعد الاجتماع وسياقه ومشاركوه وأجندته والتسجيلات المرتبطة به. |
| AI | `TranscriptionJob`, `TranscriptionSegment`, `MeetingSummary`, `ActionItem` | التتبع غير المتزامن للتفريغ، النص المقسم، ملخص الاجتماع، والمهام المستخرجة. |
| Cross-cutting | `Notification`, `AuditLog` | تنبيه المستلم وسجل التدقيق بمُعرّف ارتباط موحد. |

## قواعد البيانات والعلاقات

يمتلك `ChatChannel` أعضاءً عبر `ChannelMembership`، وهو نفسه **اختياريًا** سياق اجتماع من خلال `Meeting.ChannelId`. هذا يسمح باجتماع لقناة أو اجتماع مستقل دون فرض إنشاء قناة. يربط `ChatMessage.ReplyToMessageId` الرسائل المتفرعة داخل القناة نفسها، ويُفرض ذلك في منطق المجال والتحقق في طبقة Application لا بمجرد قيد مفتاح أجنبي.

يمر مسار نتيجة الاجتماع عبر علاقات صريحة: ينتج الاجتماع صفرًا أو أكثر من `MeetingRecording`، وينشئ كل تسجيل صفرًا أو أكثر من `TranscriptionJob` عند إعادة المحاولة، وينتج كل Job مقاطع نصية. يرتبط `MeetingSummary` باجتماع ومهمة تفريغ محددة، وينشئ الملخص صفرًا أو أكثر من `ActionItem`. لا تصبح المهام «مكتملة» تلقائيًا نتيجة إكمال الملخص؛ فحالتها دورة حياة مستقلة.

| القيد | المعالجة المقترحة |
|---|---|
| عضوية القناة | مفتاح مركب: `(ChannelId, UserId)`؛ لا يمكن تكرار العضو في القناة. |
| إيصال قراءة الرسالة | مفتاح مركب: `(MessageId, UserId)`؛ يحفظ أول وقت قراءة أو آخر وقت حسب سياسة المنتج المعتمدة. |
| حضور الاجتماع | مفتاح مركب: `(MeetingId, UserId)`؛ يجوز تحديث `JoinedAtUtc` و`LeftAtUtc` على السجل نفسه في الإصدار الأول. |
| زمن الاجتماع | يتحقق Use Case من أن `ScheduledEndUtc > ScheduledStartUtc`، وأن وقت البدء/الانتهاء الواقعي متسق مع الحالة. |
| حالة التسجيل والتفريغ | تستخدم Enums محفوظة نصيًا بصورة صريحة لسهولة الفحص والتدقيق؛ تمنع Application الانتقالات غير الصحيحة. |
| التسلسل الزمني للنص | فهرس فريد على `(TranscriptionJobId, SequenceNumber)`؛ تسترجع المقاطع بهذا الترتيب. |
| المهام | فهرس على `(AssigneeUserId, Status, DueAtUtc)` لدعم قائمة العمل والإشعارات. |

## قاعدة حاسمة لطبقة الوسائط

يحمل كيان `Meeting` حقلًا عامًا باسم `MediaSessionReference`، ويحمل `MeetingRecording` حقلًا عامًا باسم `MediaRecordingReference`. لا يحتوي النموذج على حقول من قبيل `JitsiRoomName` أو `JitsiToken` أو اسم مزوّد وسائط ثابت، لأن ذلك يكسر قابلية الاستبدال. هذه المراجع هي قيم معتمة `Opaque References` لا يفسرها إلا تنفيذ `IMeetingMediaService` داخل Infrastructure.

> لا يعرف Domain أو Application أو API أو Flutter أي SDK أو API خاص بـ Jitsi. تتعامل Use Cases مع عقود عامة، مثل `MeetingMediaSession` و`JoinMeetingResult` و`MeetingRecordingResult`، ولا تتعامل مع رابط غرفة أو Token بشكل مزوّد-محدد.

## نقاط التمديد

يظل البريد الإلكتروني خارج نطاق البناء الحالي. يوفر `Notification` حقول `Type`, `SourceType`, و`SourceId` لتوسيع وسائل الإيصال مستقبلًا دون تغيير نموذج الاجتماع أو المحادثة. وعند بدء مرحلة البريد، ينفذ Adapter مستقل في Infrastructure واجهة Application عامة للإرسال أو الإشعار، مثل `IOutboundNotificationService`؛ ولا ينبغي إنشاء كيان بريد أو اتصال Microsoft Graph في هذه المرحلة.

تسجل وظائف AI قيمة `ProviderKey` و`ProviderJobReference` في `TranscriptionJob` و`MeetingSummary` لأغراض التتبع التشغيلي فقط. يبقى اختيار Whisper أو Azure Speech أو Claude خلف واجهات Application منفصلة (`ITranscriptionService` و`IAiSummaryService`)، فلا يتسرب اسم المزوّد إلى Controllers أو واجهة Flutter.

## افتراضات مرحلة التصميم

يُعامل التسجيل والصوت والنصوص على أنها بيانات حساسة. لذلك يجب اعتماد سياسة الاحتفاظ والتصنيف والتشفير على مستوى البيئة قبل تنفيذ المهاجرات والإطلاق. هذه الوثيقة لا تفترض مدة احتفاظ أو موقع استضافة أو نوع ملف تسجيل؛ فتلك قرارات تشغيلية تحتاج اعتمادًا صريحًا قبل التنفيذ.

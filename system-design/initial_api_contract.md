# عقد API الأولي — المرحلة 1

**قاعدة النسخ:** جميع المسارات تبدأ بـ `/v1`، وتبقى أسماء الحقول والكود بالإنجليزية. لا تُولّد Models في Flutter من هذه الوثيقة يدويًا؛ يصبح Swagger/OpenAPI الناتج عن التنفيذ المرجع الحي والوحيد في المرحلة 4.

## الغلاف الموحد للاستجابة

```json
{
  "success": true,
  "data": {},
  "message": "Request completed.",
  "errors": []
}
```

عند فشل التحقق أو منطق المجال، تبقى قيمة `data` فارغة أو `null`، ويحتوي `errors` على مصفوفة عناصر ذات بنية متسقة: `field`, `code`, و`message`. يعيد Exception Middleware هذا الغلاف في الأخطاء المتوقعة؛ ولا يرمى استثناء خام من Controller.

| حالة HTTP | الاستعمال |
|---|---|
| `200 OK` | قراءة أو أمر تم بنجاح. |
| `201 Created` | إنشاء مورد جديد، مع ترويسة `Location` عند الملاءمة. |
| `202 Accepted` | طلب معالجة غير متزامنة، مثل بدء التفريغ أو إنشاء الملخص. |
| `400 Bad Request` | فشل FluentValidation أو انتقال حالة غير صحيح. |
| `401 Unauthorized` | توكن غائب أو غير صالح أو منتهي. |
| `403 Forbidden` | المستخدم موثق لكنه لا يملك الدور أو العضوية أو الصلاحية المطلوبة. |
| `404 Not Found` | المورد غير موجود أو غير متاح للمستخدم. |
| `409 Conflict` | تعارض تكرار أو عملية غير متوافقة مع حالة المورد. |
| `500 Internal Server Error` | خطأ غير معالج عبر Middleware مع `correlationId` في السجل. |

## المصادقة والهوية

| الطريقة والمسار | الدور الأدنى | Command/Query المقترح | الغرض |
|---|---|---|---|
| `POST /v1/auth/register` | `Admin` | `RegisterUserCommand` | إنشاء موظف داخلي ودور أولي. لا يفتح التسجيل الذاتي في الإصدار الأول. |
| `POST /v1/auth/login` | مجهول | `LoginCommand` | استبدال بيانات الدخول بـ Access Token وRefresh Token حسب سياسة الهوية المعتمدة. |
| `POST /v1/auth/refresh` | مجهول مع Refresh Token صالح | `RefreshTokenCommand` | تجديد جلسة الوصول. |
| `POST /v1/auth/logout` | مستخدم موثق | `LogoutCommand` | إبطال Refresh Token أو الجلسة حسب سياسة الهوية. |
| `GET /v1/users/me` | مستخدم موثق | `GetCurrentUserQuery` | ملف المستخدم والدور والصلاحيات الفعالة. |
| `GET /v1/users` | `Admin` أو `Manager` حسب السياسة | `GetUsersQuery` | قائمة موظفين قابلة للترشيح بحسب القسم والحالة. |
| `GET /v1/users/{userId}` | مستخدم موثق | `GetUserByIdQuery` | ملف مستخدم متاح ضمن نطاق الصلاحية. |
| `PATCH /v1/users/{userId}/profile` | المالك أو `Admin` | `UpdateUserProfileCommand` | تحديث الاسم الظاهر والوظيفة والقسم والصورة. |

## القنوات والمحادثات

| الطريقة والمسار | الدور الأدنى | Command/Query المقترح | الغرض |
|---|---|---|---|
| `GET /v1/channels` | مستخدم موثق | `GetChannelsQuery` | قنوات المستخدم مع مؤشر آخر قراءة وحالة الأرشفة. |
| `POST /v1/channels` | مستخدم موثق | `CreateChannelCommand` | إنشاء قناة `Direct` أو `Group` أو `Channel`. |
| `GET /v1/channels/{channelId}` | عضو في القناة | `GetChannelByIdQuery` | تفاصيل القناة وعضويتها. |
| `PATCH /v1/channels/{channelId}` | مالك القناة أو `Admin` | `UpdateChannelCommand` | تعديل الاسم أو الحالة ضمن قواعد نوع القناة. |
| `POST /v1/channels/{channelId}/members` | مالك القناة أو `Admin` | `AddChannelMemberCommand` | إضافة عضو. |
| `DELETE /v1/channels/{channelId}/members/{userId}` | مالك القناة أو `Admin` | `RemoveChannelMemberCommand` | إزالة عضو مع حماية مالك القناة الأخير. |
| `GET /v1/channels/{channelId}/messages` | عضو في القناة | `GetMessagesQuery` | رسائل صفحة محددة بمؤشر زمني أو Cursor. |
| `POST /v1/channels/{channelId}/messages` | عضو في القناة | `SendMessageCommand` | إنشاء رسالة دائمة؛ لا يعتمد إرسال الرسالة على Hub فقط. |
| `PATCH /v1/messages/{messageId}` | مرسل الرسالة | `EditMessageCommand` | تحرير جسم الرسالة ضمن مهلة وقاعدة تدقيق لاحقة. |
| `DELETE /v1/messages/{messageId}` | مرسل الرسالة أو `Admin` | `DeleteMessageCommand` | حذف منطقي مع حفظ أثر تدقيق. |
| `POST /v1/messages/{messageId}/read` | عضو في القناة | `MarkMessageReadCommand` | تحديث إيصال القراءة أو آخر قراءة. |
| `POST /v1/messages/{messageId}/attachments` | مرسل الرسالة | `AttachMessageFileCommand` | تسجيل مرفق بعد تنفيذ تدفق رفع آمن في Infrastructure. |

## الاجتماعات وطبقة الوسائط المحايدة

تستخدم جميع مسارات الاجتماع عقودًا محايدة عن المزوّد. لا يظهر `Jitsi` في المسار أو اسم الطلب أو الاستجابة أو DTO. قد تحتوي استجابة الانضمام على `mediaJoinUrl` أو `mediaSessionReference` باعتبارهما قيمًا عامة معتمة؛ تستهلك Flutter هذه القيم بواسطة تجربة وسائط عامة قابلة للاستبدال، لا بواسطة SDK خاص بـ Jitsi.

| الطريقة والمسار | الدور الأدنى | Command/Query المقترح | الغرض |
|---|---|---|---|
| `GET /v1/meetings` | مستخدم موثق | `GetMeetingsQuery` | جدول المستخدم مع مرشحات الفترة والحالة والقناة. |
| `POST /v1/meetings` | مستخدم موثق | `ScheduleMeetingCommand` | جدولة اجتماع ومشاركين وأجندة وسياق قناة اختياري. |
| `GET /v1/meetings/{meetingId}` | مشارك أو مخول | `GetMeetingByIdQuery` | تفاصيل الاجتماع والحضور والأجندة وحالة التسجيل. |
| `PATCH /v1/meetings/{meetingId}` | المنظم أو `Manager` | `UpdateMeetingCommand` | تعديل موعد أو أجندة قبل بدء الاجتماع وفق السياسة. |
| `POST /v1/meetings/{meetingId}/participants` | المنظم أو `Manager` | `InviteMeetingParticipantCommand` | دعوة مشارك أو تحديث دوره. |
| `DELETE /v1/meetings/{meetingId}/participants/{userId}` | المنظم أو `Manager` | `RemoveMeetingParticipantCommand` | إزالة دعوة قبل الاجتماع. |
| `POST /v1/meetings/{meetingId}/start` | المنظم أو مفوض | `StartMeetingCommand` | بدء اجتماع عبر `IMeetingMediaService.StartMeetingAsync`. |
| `POST /v1/meetings/{meetingId}/end` | المنظم أو مفوض | `EndMeetingCommand` | إنهاء الاجتماع عبر `IMeetingMediaService.EndMeetingAsync`. |
| `POST /v1/meetings/{meetingId}/join` | مشارك مخول | `JoinMeetingCommand` | إصدار عقد انضمام عام عبر `IMeetingMediaService.JoinParticipantAsync`. |
| `POST /v1/meetings/{meetingId}/leave` | مشارك حالي | `LeaveMeetingCommand` | تسجيل مغادرة المشارك واستدعاء `LeaveParticipantAsync` إن تطلب مزوّد الوسائط ذلك. |
| `POST /v1/meetings/{meetingId}/recording/start` | المنظم أو مفوض | `StartMeetingRecordingCommand` | بدء التسجيل عبر `IMeetingMediaService.StartRecordingAsync`. |
| `POST /v1/meetings/{meetingId}/recording/stop` | المنظم أو مفوض | `StopMeetingRecordingCommand` | إيقاف التسجيل وإنشاء مورد `MeetingRecording` مع حالة انتظار. |
| `GET /v1/meetings/{meetingId}/recordings` | مشارك مخول | `GetMeetingRecordingsQuery` | التسجيلات وحالتها وروابط الوصول الموقعة عند توفرها. |

### مثال استجابة انضمام محايدة عن المزوّد

```json
{
  "success": true,
  "data": {
    "meetingId": "a8ea5d3b-bafd-4fcf-89af-cc812e238d85",
    "mediaSessionReference": "opaque-session-reference",
    "mediaJoinUrl": "https://media-host.example/join/opaque-token",
    "expiresAtUtc": "2026-08-27T12:30:00Z"
  },
  "message": "Meeting join session created.",
  "errors": []
}
```

## ذكاء الاجتماع والمهام

| الطريقة والمسار | الدور الأدنى | Command/Query المقترح | الغرض |
|---|---|---|---|
| `GET /v1/meetings/{meetingId}/transcription` | مشارك مخول | `GetMeetingTranscriptionQuery` | حالة آخر Job ومقاطع النص عند النجاح. |
| `POST /v1/recordings/{recordingId}/transcription` | منظم أو `Manager` | `RequestTranscriptionCommand` | وضع طلب تفريغ غير متزامن في الطابور؛ يعيد `202`. |
| `GET /v1/meetings/{meetingId}/summary` | مشارك مخول | `GetMeetingSummaryQuery` | الملخص والقرارات والمهام المقترحة مع حالة الاعتماد. |
| `POST /v1/meetings/{meetingId}/summary/generate` | منظم أو `Manager` | `GenerateMeetingSummaryCommand` | طلب تلخيص واستخراج مهام من تفريغ مكتمل؛ يعيد `202`. |
| `POST /v1/meetings/{meetingId}/summary/approve` | وفق `IMeetingSummaryApprovalPolicy` | `ApproveMeetingSummaryCommand` | اعتماد الملخص وتحويل المهام المؤكدة إلى `ActionItem`؛ تقيم السياسة القاعدة الحالية للمنظم أو `Manager` وتبقى قابلة لضم أدوار مراجعة إضافية. |
| `GET /v1/action-items` | مستخدم موثق | `GetActionItemsQuery` | مهام المستخدم بحسب الحالة والموعد والاجتماع. |
| `PATCH /v1/action-items/{actionItemId}` | المسؤول أو `Manager` | `UpdateActionItemCommand` | تحديث العنوان أو الموعد أو الحالة ضمن قواعد المجال. |

## الإشعارات وSignalR

تعالج HTTP الحالة الدائمة، فيما يعالج SignalR النقل الفوري والحضور والكتابة والإشارات. يستدعي أي Hub في النهاية Use Cases أو ناشر أحداث Application؛ لا يجوز أن ينشئ Hub رسالة أو اجتماعًا أو مهمة بتجاوز CQRS أو قواعد التحقق.

| القناة | استدعاءات العميل المسموح بها | أحداث الخادم الأساسية |
|---|---|---|
| `/hubs/chat` | `SubscribeChannel(channelId)`, `UnsubscribeChannel(channelId)`, `StartTyping(channelId)`, `StopTyping(channelId)` | `messageCreated`, `messageUpdated`, `messageDeleted`, `typingChanged`, `presenceChanged`, `messageRead` |
| `/hubs/meetings` | `SubscribeMeeting(meetingId)`, `UnsubscribeMeeting(meetingId)` | `meetingStarted`, `meetingEnded`, `recordingProcessing`, `summaryReadyForReview`, `transcriptionFailed` |
| `/hubs/notifications` | `SubscribeUserNotifications()` | `notificationCreated`, `actionItemsCreated`, `notificationRead` |

| الطريقة والمسار | الدور الأدنى | Command/Query المقترح | الغرض |
|---|---|---|---|
| `GET /v1/notifications` | مستخدم موثق | `GetNotificationsQuery` | قائمة الإشعارات المؤرشفة مع حالة القراءة. |
| `POST /v1/notifications/{notificationId}/read` | مستلم الإشعار | `MarkNotificationReadCommand` | تحديث حالة القراءة. |

## حدود متعمدة لهذه المرحلة

لا تحتوي هذه العقود على Endpoints للبريد أو Microsoft Graph أو MailKit؛ فتلك نقاط تمديد مستقبلية فقط. ولا تلتزم هذه الوثيقة الآن بنوع مزوّد التخزين أو نوع طابور الخلفية أو صيغة رابط وسائط بعينها؛ تثبت هذه التفاصيل في إعداد Infrastructure عند بدء التنفيذ وبعد اعتماد متطلبات الأمن والتشغيل.

# نقل إدارة المشروع إلى Gemini

**المشروع:** منصة التواصل المؤسسي — Ultimate Solution  
**تاريخ اللقطة:** 2026-09-28  
**مصدر اللقطة:** الكود على `origin/main`، وملفات الفروع المفتوحة، وGitHub (`gh pr list` / `gh pr view` / `gh api` / `gh run view`).  
**ما لم يُمس:** لا تعديل على مستودعي Backend أو Mobile.

أي بند لم يُثبت من الكود أو من GitHub في هذه الجلسة مُعلَّم بـ **غير متحقق**.

## 1. ملخص المشروع والمعمارية

منصة تواصل داخلية لشركة Ultimate Solution: دردشة وحضور، اجتماعات صوتية/مرئية، وتلخيص اجتماعات بالذكاء الاصطناعي مع استخراج مهام. البريد نقطة تمديد لاحقة.

ثلاثة مستودعات GitHub تحت الحساب `sameer73-s`:

| المستودع | الدور | `origin/main` عند اللقطة |
|---|---|---|
| [ultimatesolution-communication-docs](https://github.com/sameer73-s/ultimatesolution-communication-docs) | الدليل المعماري، ADRs، تصميم المرحلة 1 | `800347d` |
| [ultimatesolution-communication-backend](https://github.com/sameer73-s/ultimatesolution-communication-backend) | ASP.NET Core Clean Architecture | `8f00c6b` |
| [ultimatesolution-communication-mobile](https://github.com/sameer73-s/ultimatesolution-communication-mobile) | Flutter | `20df319` |

مجلد المتطلبات `diagrams/` (بجانب المستودعات، وفيه `00_README.md` حتى `06_competitor_reference.md` ومخططان Mermaid) **ليس مستودع git**. قُرئ من القرص المحلي فقط.

### الشكل المعماري المتحقق في Backend على `main`

اتجاه الاعتماد متحقق: `API` يرجع إلى Application وInfrastructure، وApplication يرجع إلى Domain فقط (`UltimateSolution.Application.csproj`)، وDomain بلا حزم (`UltimateSolution.Domain.csproj`).

| الطبقة | ما هو موجود على `main` |
|---|---|
| Domain | كيانات الدردشة (`ChatChannel`, `ChatMessage`, `ChannelMember`, `MessageReadState`)، الاجتماعات والتسجيل، إشعارات، ذكاء الاجتماعات، أدوار `Admin` / `Manager` / `Employee` في `SystemRoles`. أنواع القنوات على `main`: `Direct` و`Group` و`Channel` فقط. |
| Application | CQRS عبر `Mediator` (ليس MediatR): Features هي `Identity` و`Chat` و`Meetings` و`AiSummary` و`Notifications`. FluentValidation. `Result` / `IUnitOfWork` / مستودعات. `IMeetingMediaService` و`ITranscriptionService` و`ISummaryService` و`IMeetingSummaryApprovalPolicy`. |
| Infrastructure | EF Core + PostgreSQL (`UseNpgsql`)، InMemory للاختبار، Identity، SignalR (`ChatHub`, `NotificationsHub`)، `InMemoryPresenceTracker`، `JitsiMeetingMediaService`، محول HTTP عام لذكاء الاجتماعات ومحول اختباري. |
| API | Controllers: Auth, Profile, Management, Channels, Messages, Presence, Meetings, MeetingIntelligence, ActionItems (قائمة وتحديث مهام الملخص)، Notifications, Health. غلاف `ApiResponse`. |

العميل على `main` يضم `lib/core` (شبكة Dio، أخطاء، ثيم مؤقت، توجيه `go_router`، DI عبر `get_it`) و`lib/features/auth` بطبقات data / domain / presentation. ميزة الدردشة ليست على `main`؛ هي في PR مفتوح (القسم 4).

## 2. القرارات الثابتة ADR-001 إلى ADR-006

| ADR | الحالة في المصدر | القرار | أين النص |
|---|---|---|---|
| ADR-001 | **معتمد** في نص القرار | Jitsi حصراً خلف `IMeetingMediaService` في Application، والتنفيذ `JitsiMeetingMediaService` داخل Infrastructure فقط. لا اسم Jitsi في Domain أو Application أو API أو Flutter. | [adr/architecture_decisions.md](../../adr/architecture_decisions.md) |
| ADR-002 | مذكور كقرار ثابت في الدليل | ملخص الاجتماع يُحفظ `Draft`. اعتماد الملخص يمر عبر `IMeetingSummaryApprovalPolicy` لا بفحص دور ثابت داخل أمر الاعتماد. القاعدة الحالية: المنظم أو `Manager`. | نفس الملف، والقسم 3 من [Architecture_Guideline.md](../../reference/Architecture_Guideline.md) |
| ADR-003 | **مقبول** (2026-08-28) | استبدال MediatR بـ `Mediator.Abstractions` و`Mediator.SourceGenerator` الإصدار `3.0.2` (ترخيص MIT). حظر MediatR وLucky Penny. | [backend `docs/adr/ADR-003`](https://github.com/sameer73-s/ultimatesolution-communication-backend/blob/main/docs/adr/ADR-003-mediator-library.md) — **غير موجود** كملف في مستودع docs |
| ADR-004 | **مقبول** (2026-08-28) | نموذج ذكاء الاجتماعات والعقود العامة الآن. ربط Whisper وClaude والطابور والتخزين والاحتفاظ مؤجل صراحة. | [backend `docs/adr/ADR-004`](https://github.com/sameer73-s/ultimatesolution-communication-backend/blob/main/docs/adr/ADR-004-ai-meeting-intelligence-integration-boundary.md) — **غير موجود** في مستودع docs |
| ADR-005 | **مقبول ومنجز** (2026-08-28) | توثيق JWT Bearer في OpenAPI للعمليات غير المجهولة. أسرار JWT الإنتاجية تبقى بنداً مفتوحاً. | [backend `docs/adr/ADR-005`](https://github.com/sameer73-s/ultimatesolution-communication-backend/blob/main/docs/adr/ADR-005-openapi-jwt-security-documentation.md) — **غير موجود** في مستودع docs |
| ADR-006 | النص على `main` ما زال يقول **مقترح** حتى الدمج، بينما [PR #2](https://github.com/sameer73-s/ultimatesolution-communication-docs/pull/2) **دُمج** في 2026-08-30 | PostgreSQL عبر `Npgsql.EntityFrameworkCore.PostgreSQL` بدل SQL Server، لتهيئة استضافة Render. معرفة المزود في Infrastructure. النشر الفعلي خارج نطاق القرار. | [adr/ADR-006-postgresql-migration.md](../../adr/ADR-006-postgresql-migration.md) |

التحقق من الكود على backend `main` يطابق جوهر ADR-001 وADR-003 وADR-004 وADR-005 وADR-006: لا ظهور لاسم `Jitsi` خارج Infrastructure، الحزم هي Mediator `3.0.2` وNpgsql `10.0.3`، ولا توجد حزمة MediatR في ملفات المشاريع. أمر `ApproveMeetingSummaryCommandHandler` يستدعي `IMeetingSummaryApprovalPolicy` ثم يرمي `DomainForbiddenException` عند فشل السياسة. التنفيذ `OrganizerOrManagerMeetingSummaryApprovalPolicy` في Infrastructure.

## 3. جدول WBS

لا يوجد ملف Excel للمرحلة 2 (ورقة WBS / Timeline / المخاطر) داخل المستودعات الثلاثة ولا داخل مجلد `diagrams/`. الترقيم أدناه مأخوذ من [Manus_Master_Prompt.md](../../reference/Manus_Master_Prompt.md) ومن عناوين الـPRs. ملف الإصدار الوحيد المكتوب هو `docs/WBS-3.8.3.md` وهو **ليس على `main`**.

| المعرف | النطاق | الحالة | الدليل |
|---|---|---|---|
| 0 | بحث السوق وعرض الإدارة | لم يبدأ | لا ملف عرض في المستودعات |
| 1 | التصميم الهيكلي (ERD، معمارية، تسلسل، عقد API) | مدمج | `system-design/` على docs `main` |
| 2 | Excel إدارة المشروع | لم يبدأ | لا `.xlsx` في المستودعات ولا في `diagrams/` |
| 3.1 | أساس Backend | مدمج | backend [PR #1](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/1) |
| 3.2 | Identity + JWT، ثم استبدال MediatR | مدمج | backend [PR #2](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/2) |
| 3.3 | دردشة وحضور SignalR | مدمج | backend [PR #3](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/3) |
| 3.4 | اجتماعات + محول Jitsi | مدمج | backend [PR #4](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/4) |
| 3.5 | ذكاء الاجتماعات (عقود ومحولات، بلا مزود حقيقي) | مدمج | backend [PR #5](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/5) |
| 3.6 | إشعارات داخل التطبيق | مدمج | backend [PR #6](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/6) |
| 3.6 | توثيق JWT في OpenAPI | مدمج | backend [PR #7](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/7) |
| 3.7 | ترحيل PostgreSQL + `render.yaml` بلا نشر ضمن الخطوة | مدمج | backend [PR #8](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/8) وdocs [PR #2](https://github.com/sameer73-s/ultimatesolution-communication-docs/pull/2) |
| 3.8.2 | كيانات أدوار/مشاريع/ESS ضمن فرع التزامن | مفتوح داخل PR واحد، بلا دمج مستقل | جزء من backend [PR #11](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/11). فرع محلي `feature/phase-3.8.2-roles-migrations` بلا رأس على origin |
| 3.8.3 | فهرس فريد لمصدر رسالة ActionItem وتحويل رسالة إلى مهمة | **مفتوح** | backend [PR #11](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/11). [PR #9](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/9) و[PR #10](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/10) مغلقة ولم تُدمج |
| 3.9 | GitHub Actions: build وtest | مدمج | backend [PR #13](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/13). [PR #12](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/12) مغلق |
| 4.1 | Flutter core | مدمج | mobile [PR #1](https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/1) |
| 4.2 | Flutter Auth | مدمج | mobile [PR #2](https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/2)، ثم إصلاحات [PR #3](https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/3) و[PR #4](https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/4) |
| 4.3 | Flutter Chat + SignalR | **مفتوح** | mobile [PR #5](https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/5) |
| 4.4 | Flutter Meetings | لم يبدأ | لا `lib/features/meetings` على `main` ولا في فرق PR #5 |
| 4.5 | Flutter AI Summary | لم يبدأ | لا `lib/features/ai_summary` |
| 4.6 | Flutter Notifications | لم يبدأ | لا `lib/features/notifications` |
| 5 | استخراج الهوية البصرية وتطبيقها | لم يبدأ | الثيم في `lib/core/theme/app_theme.dart` ما زال placeholder حسب الدليل |
| 6 | البريد | لم يبدأ | لا MailKit ولا Microsoft Graph في مشاريع Backend |

حزمة `diagrams/` أوسع من الشات المدمج: ملفات، ردود، منشن، تفاعلات، تثبيت، حذف بموافقة الطرفين، ESS، ومهام مشاريع. على `main` القنوات بلا نوع `Project` وبلا هذه المسارات. كيانات `Project` و`EssAccessRequest` و`MessageDeletionRequest` ونوع `Project = 4` تظهر في فرق PR #11 فقط، بلا Controllers باسم ESS أو حذف الرسائل في قائمة ملفات ذلك الـPR.

### آخر إصدار موثق

| الموقع | ماذا يقول | حكم اللقطة |
|---|---|---|
| docs `main` (`800347d`) | لا `docs/status-log.md` ولا `docs/WBS-3.8.3.md`. آخر دمج هو ADR-006 / WBS 3.7 | آخر قرار **مدمج** في مستودع docs |
| [docs PR #3](https://github.com/sameer73-s/ultimatesolution-communication-docs/pull/3) فرع `docs/wbs-3.8.3-release-notes` عند `a8d5900` | سجل الحالة يسمّي آخر إصدار **WBS-3.8.3** والخطوة التالية `feature/4.3-chat-feature` في mobile | **مفتوح**، `MERGEABLE`، `CLEAN`، بلا `reviewDecision`. ليس على `main` |
| backend `main` | هجرة واحدة: `20260830151350_InitialPostgreSql`. لا هجرة `20260901202720_Phase383ActionItemUniqueSourceMessage` | تنفيذ 3.8.3 غير مدمج |

إذن: الإصدار الذي تسمّيه وثيقة PR #3 «آخر إصدار» هو WBS-3.8.3، وهذا التوثيق والهجرة المرتبطة به غير مدمجين. اسم الخطوة التالية في ذلك السجل (`feature/4.3-chat-feature`) لا يطابق الفرع الفعلي `feature/4.3-flutter-chat`.

### backend PR #11

| الحقل | القيمة عند 2026-09-28 |
|---|---|
| الرابط | https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/11 |
| العنوان | WBS 3.8.3 - Action Items concurrency validation and migration rollback fix |
| الحالة | **OPEN**، ليس draft، `mergedAt` فارغ |
| الرأس | `feature/3.8.3-action-items-concurrency` عند `6701e7e` |
| القاعدة | `main` |
| GitHub | `mergeable=MERGEABLE`، `mergeStateStatus=CLEAN`، `reviewDecision` فارغ |
| الحجم | 39 ملفاً، +4326 / −25 (حسب `gh pr view`) |
| الفحوصات | `gh pr checks 11`: لا فحوصات على الفرع. **غير متحقق:** هل تنجح اختبارات هذا الفرع |
| المحتوى المتحقق بقراءة الفرق | هجرتان غير موجودتين على `main`: `20260901141520_Phase3_8_RolesAndEntities` و`20260901202720_Phase383ActionItemUniqueSourceMessage` (فهرس فريد على `SourceMessageId`، و`Down()` يعيد المفتاح بـ `ReferentialAction.Restrict`). أمر `ConvertMessageToActionItem` ومسار `POST /api/v1/messages/{messageId}/action-items`. أدوار جديدة `ProjectManager` و`HR`. نوع قناة `Project` |

اختبار التعارض في `tests/UltimateSolution.Application.Tests/Features/ActionItems/ConvertMessageToActionItemHandlerTests.cs` على هذا الفرع معلّق في الكود بأنه Unit Test يحاكي `DbUpdateException` وليس PostgreSQL Integration Test. `FakeFailingUnitOfWork` يرمي `DbUpdateException` برسالة داخلية اسم الفهرس، بلا قاعدة PostgreSQL. المعالج في Application يلتقط الاستثناء بمقارنة `GetType().Name == "DbUpdateException"` مع نص `23505` أو اسم الفهرس.

### mobile PR #5

| الحقل | القيمة عند 2026-09-28 |
|---|---|
| الرابط | https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/5 |
| العنوان | feat(chat): WBS 4.3 Flutter Chat feature |
| الحالة | **OPEN**، ليس draft، `mergedAt` فارغ |
| الرأس | `feature/4.3-flutter-chat` عند `e3a400d` (2026-09-23) |
| القاعدة | `main` |
| GitHub | `mergeable=MERGEABLE`، `mergeStateStatus=CLEAN`، `reviewDecision` فارغ |
| الحجم | 65 ملفاً، +8005 / −3 |
| الفحوصات | لا workflow في المستودع (`git ls-tree origin/main -- .github` يظهر قالب PR فقط). `gh pr checks 5`: لا فحوصات. `gh run list` فارغ |
| المحتوى | طبقات chat كاملة، `signalr_netcore` ^1.4.4، مسارات `/channels` و`/channels/:channelId`، ومجلد `windows/` أُضيف في `e3a400d` |

`docs/review/test-results-summary.md` على الفرع مؤرخ `2026-08-31T12:48:38Z`: 27 اختباراً، ناجح 26، متخطى 1 (`live_hub_verify_test` إلا إذا فُعّل `LIVE_HUB`)، فاشل 0، و`flutter analyze` بلا مشاكل. هذا الملف **لم يُحدَّث** في `e3a400d`. **غير متحقق:** نتيجة `flutter test` بعد إضافة Windows وتعديل صفحتي الدخول.

ملاحظة SignalR على الفرع تقول إن اتصالاً حياً بـ `https://ultimatesolution-communication-backend.onrender.com/hubs/chat` نجح في 2026-08-31. **غير متحقق:** هل الخدمة ما زالت تعمل الآن. ADR-006 يقول إن النشر ليس جزءاً من قرار الترحيل، و`render.yaml` على `main` يصرّح أنه تهيئة لخطوة استضافة لاحقة (`autoDeployTrigger: "off"`).

## 4. حالة كل مستودع

### docs

| البند | الحالة |
|---|---|
| فروع origin عند اللقطة | `main`، `feature/3.4-media-continuity-decision`، `feature/3.7-postgresql-migration`، `docs/wbs-3.8.3-release-notes` |
| PRs | #1 مدمج، #2 مدمج، #3 مفتوح |
| اختبارات | لا مشروع اختبار في المستودع |
| حماية `main` | `GET .../branches/main/protection` يعيد 404 «Branch not protected». `rulesets` يعيد `[]` |
| اعتماديات | لا حزم تطبيق. المستودع Markdown |

### backend

| البند | الحالة على `origin/main` `8f00c6b` |
|---|---|
| فروع origin التي ما زالت موجودة | `main`، `feature/3.1-backend-foundation` حتى `feature/3.7-postgresql-migration`، `feature/3.3-chat-presence`، `feature/3.4-meetings`، `feature/3.5-ai-meeting-intelligence`، `feature/3.6-notifications`، `feature/3.6-openapi-security`، `feature/3.8.3-action-items-concurrency`، `chore/3.9-ci-dotnet-build-test` |
| فروع محلية بلا رأس origin | `feature/phase-3.8.2-roles-migrations` (`4561e0d`)، `feature/phase-3.8.3-message-to-task` (`2976327`). `chore/3.9-ci-dotnet-build-test-v2` تتبعه على origin صار `gone` بعد الدمج |
| اختبارات `main` | GitHub Actions [run 35895589369](https://github.com/sameer73-s/ultimatesolution-communication-backend/actions/runs/35895589369) في 2026-09-23: نجاح. `UltimateSolution.Application.Tests` 11 ناجحاً، `UltimateSolution.API.IntegrationTests` 12 ناجحاً، 0 فاشل. بيئة `Testing` وقاعدة InMemory كما في `docs/database-provider-migration.md` |
| حماية `main` | 404 و`rulesets: []`، مثل docs |
| SDK | `global.json`: الإصدار `10.0.111`، `rollForward: latestFeature`. المشاريع `net10.0` |

اعتماديات backend على `main` (من ملفات `csproj`، لا من تخمين):

| الحزمة | الإصدار | المشروع |
|---|---|---|
| `Mediator.Abstractions` | 3.0.2 | Application وAPI |
| `Mediator.SourceGenerator` | 3.0.2 | Application |
| `FluentValidation.DependencyInjectionExtensions` | 12.1.1 | Application |
| `Microsoft.Extensions.DependencyInjection.Abstractions` | 10.0.11 | Application |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | 10.0.3 | Infrastructure |
| `Microsoft.EntityFrameworkCore.InMemory` | 10.0.11 | Infrastructure |
| `Microsoft.AspNetCore.Identity.EntityFrameworkCore` | 10.0.11 | Infrastructure |
| `Microsoft.EntityFrameworkCore.Design` | 10.0.11 | Infrastructure وAPI |
| `System.IdentityModel.Tokens.Jwt` | 8.19.2 | Infrastructure |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | 10.0.11 | API |
| `Microsoft.AspNetCore.OpenApi` | 10.0.11 | API |

لا `MediatR` ولا `AutoMapper` ولا `MailKit` في ملفات المشاريع على `main`.

### mobile

| البند | الحالة |
|---|---|
| `main` `20df319` | core + auth. صفحات `login_page.dart` و`register_page.dart` موجودة. لا `signalr_netcore` في `pubspec.yaml` على `main` |
| PR #5 | يضيف chat والحزمة `signalr_netcore: ^1.4.4` ومجلد `windows/` |
| فروع origin | `main`، `feature/4.1-flutter-core-setup`، `feature/4.2-flutter-auth`، `feature/4.3-flutter-chat`، `fix/auth-failure-mapping`، `fix/flutter-auth-routes-registration` |
| اختبارات على `main` | ملفات `test/core_setup_test.dart` و`app_router_test.dart` و`auth_*`. **غير متحقق:** لم يُشغَّل `flutter test` على `main` في هذه الجلسة، ولا CI على GitHub |
| حماية `main` | 404 و`rulesets: []`. هذا هو بند «حماية main غير مفعلة في mobile»، وهو متحقق أيضاً على backend وdocs |

اعتماديات mobile على `main` من `pubspec.yaml`:

| الحزمة | القيد |
|---|---|
| Dart SDK | `^3.13.2` |
| `dio` | `^5.11.0` |
| `get_it` | `^9.2.1` |
| `go_router` | `^18.0.0` |
| `flutter_bloc` | `^9.1.1` |
| `flutter_secure_storage` | `^11.0.0` |
| `flutter_lints` (dev) | `^6.0.0` |

لا `dartz` ولا `injectable` ولا `equatable` في هذا الملف. DI على فرع الدردشة يسجّل `SecureAccessTokenStore` لا `InMemoryAccessTokenStore`. الصنف الذاكري ما زال في `api_client.dart` وموسوماً في الكود بأنه للتطوير والاختبار.

## 5. ديون تقنية وبنود مفتوحة قبل الإنتاج

| البند | الدليل | الحالة |
|---|---|---|
| جلسات وتسجيلات Jitsi في الذاكرة | `JitsiMeetingMediaService` يحتفظ بـ `_sessions` و`_recordings` كـ `ConcurrentDictionary`. مرجع الجلسة `media-session-{Guid}` جديد في كل بدء. `docs/meetings.md` وADR-001 في docs يطلبان قبل الإنتاج إعادة بناء الغرفة من حالة دائمة والتحقق من المزوّد قبل تغيير حالة الاجتماع | مفتوح على `main` |
| الحضور InMemory | `services.AddSingleton<IPresenceTracker, InMemoryPresenceTracker>()` و`docs/chat-presence.md`: ليس حضوراً موزعاً | مفتوح على `main` |
| اختبار تعارض ActionItems | الاختبار على فرع PR #11 وحدة محاكاة، و`docs/WBS-3.8.3.md` على docs PR #3 يسجّله بنداً مفتوحاً قبل الإنتاج. لا اختبار تكامل PostgreSQL في مشروع الاختبارات على `main` | مفتوح، والكود نفسه غير مدمج |
| Whisper وClaude غير مربوطين | `ExternalMeetingIntelligenceService` عميل HTTP إلى `MeetingIntelligence:ServiceUrl` ومسار `v1/transcriptions`. عند غياب العنوان يفشل برمز `meeting_intelligence_service_not_configured`. بيئة الاختبار تستخدم `TestMeetingIntelligenceService`. لا اسم Whisper أو Claude في `src`. ADR-004 يؤجل الربط | مفتوح على `main` |
| حماية `main` في mobile | GitHub 404 وrulesets فارغة | مفتوح. الدليل و`CONTRIBUTING.md` يقولان إن `main` محمي |
| طابور خلفية، Object Storage، احتفاظ التسجيلات، إعادة المحاولة | قسم «قرارات ما زالت مفتوحة» في `architecture_decisions.md` وADR-004 | مفتوح |
| أسرار JWT و`MeetingMedia` و`MeetingIntelligence` | ADR-005 يُبقي مفتاح الإنتاج وIssuer/Audience والتدوير مفتوحاً. `render.yaml` يضع `Jwt__Key` و`MeetingMedia__*` و`MeetingIntelligence__*` على `sync: false` | مفتوح |
| أدوار زائدة عن الدليل داخل PR #11 | `ProjectManager` و`HR` ليسا في القسم 2 من الدليل (الثلاثة: Admin / Manager / Employee) | على الفرع المفتوح فقط |
| توسيع نطاق PR #11 | يخلط كيانات 3.8.2 (مشاريع، ESS، طلب حذف رسالة) مع قيد 3.8.3 | مراجعة قبل أي دمج |
| نتيجة اختبارات PR #11 وPR #5 بعد آخر commit | لا فحوصات GitHub | **غير متحقق** |
| توفر Render الآن | وثيقة 2026-08-31 فقط | **غير متحقق** |

أوامر غير الاعتماد في `AiSummaryHandlers.cs` (`RequestTranscription` و`GenerateMeetingSummary`) تستدعي `AiSummaryAuthorization.EnsureOrganizerOrManager` داخل Application. هذا ليس مسار `ApproveMeetingSummaryCommand`، ومسار الاعتماد نفسه يستخدم السياسة. يُذكر حتى لا يُعامل كل فحص دور في الملف كخرق لـ ADR-002.

## 6. مخالفات أو تعارضات بين الكود والدليل

هذه تعارضات نصية أو بنيوية متحققة، لا قائمة أمنيات.

1. **MediatR ما زال في نصوص معتمدة بينما الكود على Mediator.** القسم 3 من [Architecture_Guideline.md](../../reference/Architecture_Guideline.md) (تعليق المجلد وسطر CQRS) و[Manus_Master_Prompt.md](../../reference/Manus_Master_Prompt.md) المرحلة 3 و[system-design/diagrams/architecture.mmd](../../system-design/diagrams/architecture.mmd) و[system-design/README.md](../../system-design/README.md) في جدول الطبقات و`diagrams/00_README.md` تقول MediatR. السطر 9 من `system-design/README.md` يقول Mediator. الكود وADR-003 على Mediator `3.0.2`. الدليل حُدّث إلى PostgreSQL في الإصدار 1.1 ولم يُحدَّث سطر MediatR. ADR-003 نفسه غير منسوخ إلى `adr/` في مستودع docs.

2. **ADR-006 على docs `main` ما زال «مقترحاً حتى الدمج»** بعد دمج PR #2 في 2026-08-30. الحالة في الملف لم تُقلب إلى مقبول.

3. **الدليل وCONTRIBUTING يقولان إن `main` محمي وبلا push مباشر.** GitHub لا يفعّل حماية الفرع ولا rulesets على المستودعات الثلاثة.

4. **AutoMapper مذكور في شجرة المجلدات بالقسم 3** («Mappings - AutoMapper») ولا توجد حزمة AutoMapper في أي `csproj` على `main`. التخطيط يدوي في الكود.

5. **«الاتصال بقاعدة البيانات حصراً عبر `IApplicationDbContext`».** الواجهة موجودة في Application، والمعالجات على `main` تستخدم المستودعات و`IUnitOfWork` ولا تحقن `IApplicationDbContext`. هذا يوافق Repository + Unit of Work المطلوب في الدليل نفسه، ويخالف القراءة الحرفية لكلمة «حصراً» إن قُصد أن كل استعلام يمر من تلك الواجهة.

6. **مسار DI في Flutter.** الدليل: `lib/injection_container.dart`. الكود: `lib/core/di/injection_container.dart`.

7. **ملفات Bloc.** الدليل يطلب ملفات منفصلة للـ bloc والـ event والـ state. `auth_bloc.dart` وملفات chat على PR #5 تجمعها في ملف واحد.

8. **نمط الخطأ في Flutter.** الدليل يذكر `Either<Failure, T>` (dartz) أو نمطاً مشابهاً. الكود يستخدم سجلاً `({T? value, Failure? failure})` و`Failure` مغلق في `lib/core/errors/failures.dart`، بلا `dartz`. الجملة «أو نمط مشابه» تجعل هذا انحرافاً عن المثال لا خرقاً إن قُبل النمط الموحد.

9. **روابط مكسورة في تصميم المرحلة 1.** `system-design/README.md` يشير إلى `architecture_decisions.md` و`api/initial_api_contract.md` بمسار نسبي داخل `system-design/`. الملفان ليسا هناك: القرارات في `adr/architecture_decisions.md` والعقد في `system-design/initial_api_contract.md`.

10. **سجل الحالة المطلوب كخطوة تالية يتعارض مع الواقع.** على فرع docs PR #3 الخطوة التالية `feature/4.3-chat-feature`، بينما PR #5 مفتوح أصلاً على `feature/4.3-flutter-chat`. توثيق WBS-3.8.3 يصف الهجرة والفهرس كأن الإصدار قائم، وهما غير مدمجين في backend `main`.

11. **معالجة التعارض ليست في طبقة الحفظ كما يقول نص WBS-3.8.3 على PR #3.** النص: المعالجة في طبقة الحفظ بالتقاط `DbUpdateException`. الكود على فرع PR #11 يلتقط الاستثناء داخل `ConvertMessageToActionItemHandler` في Application، وبمقارنة اسم النوع كنص.

12. **لافتة المرحلة 1 قديمة.** `system-design/README.md` يقول إن المخرجات تصميم ولا يبدأ الكود قبل اعتماد المرحلة. Backend حتى 3.7 و3.9 وMobile حتى 4.2 مدمجان بعد ذلك التاريخ (27 أغسطس 2026 في رأس الملف).

13. **البرومبت الرئيسي ما زال يذكر SQL Server** في مخطط المعمارية للمرحلة 1. الدليل 1.1 وADR-006 و`architecture.mmd` على PostgreSQL.

14. **حزمة الدردشة المحلية تقول «جاهزة للمراجعة قبل التنفيذ»** (`diagrams/00_README.md`) وتذكر MediatR. تنفيذ الشات في Backend مدمج، وعميل الشات في PR مفتوح، وكثير من متطلبات الحزمة لم يبدأ (القسم 3).

ما طابق الدليل على `main` ويُذكر حتى لا يُعاد فتحه بلا سبب: عزل Jitsi، سياسة اعتماد الملخص لأمر الاعتماد، PostgreSQL في Infrastructure، غلاف `ApiResponse`، SignalR في Infrastructure، أدوار الثلاثة، Flutter Bloc + Dio + get_it لميزة Auth، وعدم استيراد Flutter داخل `domain/`.

## 7. قواعد العمل

من [Architecture_Guideline.md](../../reference/Architecture_Guideline.md) القسم 9 و[CONTRIBUTING.md](../../CONTRIBUTING.md) وقالب [PULL_REQUEST_TEMPLATE.md](../../.github/PULL_REQUEST_TEMPLATE.md):

- فرع لكل خطوة: `feature/<WBS-ID>-<اسم-مختصر>`. مثال الدليل: `feature/3.1-backend-foundation`.
- كل خطوة تنتهي بـ Pull Request فيه رقم WBS والمخرج المتوقع ومعيار القبول.
- لا دمج في `main` إلا بعد اعتماد صريح من الجهة المخولة. فتح الـPR ليس اعتماداً. هذه الجلسة لا تدمج شيئاً.
- لا حزمة ولا قرار معماري خارج الدليل إلا بموافقة صريحة.
- الانحرافات التي حدثت فعلاً ولا تُؤخذ سابقة: `chore/3.9-*`، `fix/auth-failure-mapping`، `fix/flutter-auth-routes-registration`، `docs/wbs-3.8.3-release-notes`، والفرعان المحليان `feature/phase-3.8.*`.

### شكل برومبت Cursor

```text
المستودع: <backend | mobile | docs>
الفرع: feature/<WBS-ID>-<اسم-مختصر> من origin/main
المهمة: <جملة واحدة ونطاق WBS>
الملفات: <مسارات مسموح لمسها فقط>
الاختبارات: <الأمر ومعيار النجاح>
المطلوب: <سلوك يمكن مراجعته>
غير مسموح: تعديل مستودع آخر، دمج main، إضافة حزمة غير مذكورة هنا
```

## 8. الخطوة التالية المقترحة

لا تبدأ WBS 4.4 ولا أي ميزة جديدة.

الخطوة التالية هي مراجعة [backend PR #11](https://github.com/sameer73-s/ultimatesolution-communication-backend/pull/11) من غير دمج، ثم إبقاء [docs PR #3](https://github.com/sameer73-s/ultimatesolution-communication-docs/pull/3) معلقاً حتى يطابق النص ما يُعتمد، ثم مراجعة [mobile PR #5](https://github.com/sameer73-s/ultimatesolution-communication-mobile/pull/5) من غير دمج.

المبررات:

- بوابة المشروع هي الاعتماد ثم الدمج. ثلاثة PRs مفتوحة بلا `reviewDecision`.
- WBS-3.8.3 موثّق كآخر إصدار في docs PR #3 بينما الهجرة والاختبار ليسا على backend `main`. دمج التوثيق قبل حسم الكود يجعل المرجع الحي يصف سلوكاً غير مدمج.
- PR #11 أوسع من عنوانه: أدوار خارجة عن الدليل، كيانات ESS ومشاريع، واختبار تعارض محاكى سجّله التوثيق نفسه بنداً قبل الإنتاج. لا فحوصات CI على الفرع.
- سجل الحالة يسمي خطوة mobile لم تُفتح بهذا الاسم؛ PR #5 مفتوح بالفعل. إعادة تشغيل `flutter test` بعد `e3a400d` جزء من مراجعته، لأن ملخص 26 نجاحاً أقدم من commit ويندوز.
- بدء اجتماعات Flutter قبل حسم هذين الـPR يخالف قاعدة التسلسل في القسم 8 من الدليل.

## 9. ملفات تُرفع إلى Gemini في بداية كل جلسة

ارفع هذه المجموعة ثابتة، ثم أضف فرق الـPR المفتوح الذي ستراجعه في تلك الجلسة. لا تكتفِ بملف نقل قديم إذا تغيّر `main`.

من مستودع docs على `main`، وبعد دمج هذا الملف:

- `docs/handover/GEMINI_HANDOVER.md` (هذا الملف)
- `reference/Architecture_Guideline.md`
- `reference/Manus_Master_Prompt.md`
- `adr/architecture_decisions.md`
- `adr/ADR-006-postgresql-migration.md`
- `CONTRIBUTING.md`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `system-design/README.md`
- `system-design/data_model_design.md`
- `system-design/initial_api_contract.md`
- `system-design/diagrams/architecture.mmd`
- `system-design/diagrams/erd.mmd`
- `system-design/diagrams/sequence_meeting_lifecycle.mmd`
- `system-design/diagrams/sequence_ai_meeting_intelligence.mmd`

من الفرع المفتوح `docs/wbs-3.8.3-release-notes` طالما PR #3 لم يُدمج ولم يُغلق:

- `docs/status-log.md`
- `docs/WBS-3.8.3.md`

من backend `main`:

- `docs/adr/ADR-003-mediator-library.md`
- `docs/adr/ADR-004-ai-meeting-intelligence-integration-boundary.md`
- `docs/adr/ADR-005-openapi-jwt-security-documentation.md`
- `docs/meetings.md`
- `docs/chat-presence.md`
- `docs/ai-meeting-intelligence.md`
- `docs/database-provider-migration.md`
- `README.md`

من القرص المحلي (بلا git): مجلد `diagrams/` كاملاً عند مناقشة نطاق الدردشة الموسّع.

عند مراجعة PR بعينه: رابط الـPR و`git diff origin/main...<head>` لذلك الرأس، لا الاعتماد على وصف الـPR وحده.

## 10. برومبت نظام جاهز لـ Gemini

انسخ النص التالي كما هو إلى تعليمات النظام:

```text
أنت مدير المشروع والمعماري التقني لمنصة التواصل المؤسسي Ultimate Solution. تعمل على ثلاثة مستودعات: ultimatesolution-communication-docs وultimatesolution-communication-backend وultimatesolution-communication-mobile، وعلى مجلد diagrams المحلي غير المتتبع في git.

مرجعك الثابت هو Architecture Guideline وADRs من ADR-001 إلى ADR-006 وملف GEMINI_HANDOVER.md. ADR-003 وADR-004 وADR-005 تعيش في مستودع backend. الكود على origin/main يغلب نصاً قديماً في الدليل عندما يتعارض الاثنان في نقطة محسومة بقرار مقبول: الوسيط هو Mediator بترخيص MIT وليس MediatR، وقاعدة البيانات PostgreSQL وليس SQL Server، وJitsi يبقى خلف IMeetingMediaService داخل Infrastructure فقط.

لا تدمج أي Pull Request. لا تطلب push إلى main. كل خطوة جديدة فرع feature/<WBS-ID>-<اسم-مختصر> من origin/main ثم Pull Request. الدمج قرار بشري صريح بعد المراجعة. لا تضف حزمة أو دوراً أو مزوداً خارج الدليل إلا إذا طُلبت الموافقة صراحة.

قبل أن تصف حالة WBS، تحقق من GitHub: ما المدمج على main، وما المفتوح، وما الذي لم يبدأ. وثيقة WBS-3.8.3 وسجل الحالة ليسا على docs main عند تسليم هذا الملف؛ هما في docs PR #3. تنفيذ التزامن في backend هو PR #11 المفتوح. دردشة Flutter هي mobile PR #5 المفتوح. إذا تغيرت هذه الحقائق بعد 2026-09-28 فالحقيقة هي git وGitHub لا هذا الملخص.

ميّز ما تحقق بقراءتك مما لم يتحقق، واكتب «غير متحقق» على الثاني. لا تخترع SHAs ولا نتائج اختبارات ولا حالة حماية فروع.

شكل كل رد:
1) الحكم أو الخطوة المطلوبة في سطرين.
2) الحقائق المتحققة مع المستودع والفرع أو رقم PR.
3) البنود غير المتحققة.
4) المخاطر على القرارات المعمارية إن وُجدت.
5) إن كلّفت Cursor، استخدم هذا القالب فقط: المستودع، الفرع، المهمة، الملفات، الاختبارات، المطلوب، وغير المسموح.
لا تبدأ ميزة Flutter جديدة قبل حسم PR #11 وPR #5 وdocs PR #3، ما لم يقرر الإنسان خلاف ذلك كتابة.
```

## 11. بنود غير متحقق مجمّعة

- نجاح أو فشل اختبارات فرع backend PR #11 (لا فحوصات GitHub، ولم تُشغَّل محلياً هنا).
- نجاح `flutter test` على mobile `main`، وبعد commit `e3a400d` على PR #5.
- عمل خدمة Render الآن. المتوفر وثيقة فرع بتاريخ 2026-08-31.
- وجود ملف Excel أو عرض مرحلة 0 خارج هذه المجلدات الثلاثة و`diagrams/`.

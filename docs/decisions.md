# Design decisions

Six decisions from the project's decision log of more than 400 entries, each with its context, what was chosen, what it was chosen over, and what it costs.

## The database is the last guard against an invalid booking

A booking touches several rows at once: the appointment, the member's package, a credit ledger row, an event and an outbox row. Two staff members, or a member and a trainer, can act on the same calendar in the same second.

The booking rules run in TypeScript inside one server-managed transaction, and Postgres refuses the invalid states on its own. An exclusion constraint keeps a provider's confirmed appointments from overlapping, in any branch. A second one, on a shared table of staff time blocks, covers tours and appointments together. A check constraint keeps a package's used count between zero and its size, and a trigger refuses any booking outside the branch's effective opening hours, for every role.

The alternatives were application checks alone, which race, or putting the whole booking in Postgres functions, which is harder to unit test. The rules stay in a framework-free package with 305 unit tests, and the constraints are tested with real concurrent connections.

The cost: rules live in two languages, and lock order matters. Locks on availability weeks are always taken in ascending id order, because any other order can deadlock against a trainer editing their weekly template.

## One contract and generated clients

Three codebases call the API: the API's own tests, the Windows app and the Android app. A field renamed on one side and not the other is a bug that ships.

Request and response schemas are Zod objects in one shared package. The API validates requests with them, the API tests parse responses with them, and the typed client the Windows app uses is built on them. The same package produces an OpenAPI 3.1 document (99 operations, 150 schemas), and the Kotlin models for Android are generated from that document. Responses are open to new fields and new enum values, so an older client does not break when the server adds one.

The alternative was hand-written models per client, or React Native with shared TypeScript. The project tried an Expo app first and replaced it with native Kotlin.

The cost: the generators are code to maintain. The drift checks for the committed OpenAPI file, the Kotlin models and the generated strings run in a local Android gate, not in CI.

## Permissions are checked in the API and again in Postgres

Nine roles share one database: member, trainer, consultant, receptionist and five manager levels. A trainer must see only their own trainees' sessions; a consultant sees every provider's calendar to find a free trainer.

The API checks a permission map (38 grants) in middleware. It then opens the transaction as a restricted role and sets the account, session, acting role, organization and branch as transaction-local settings. Row-level security policies read those settings and call the same permission table, which also lives in the database. A user with several roles picks one acting role per request, and visibility follows that role alone, never a union. Branches are isolated; only area managers and above switch branch, one at a time.

The alternative was checking in the API only. With RLS underneath, a check missed in the API still returns only the rows the acting role may see.

The cost: a policy set on every table, one transaction per command, and a separate test suite for RLS. Foreign keys carry the organization id, so a row cannot point at another tenant's data.

## Background work goes through an outbox and an always-on worker

Bookings trigger notifications, weeks publish on a schedule, standing sessions are booked ahead, and reminders fire 24 and 2 hours before a session. The API runs on a serverless host with no long-running process.

Each write inserts an outbox row in the same transaction as the change. A separate worker process, run as a systemd service on an always-on Linux host, makes a pass every 15 seconds through nine steps. It claims rows with `FOR UPDATE SKIP LOCKED`, does the work and marks the row done in one transaction, and records a failure with a backoff in a second one.

The alternatives were sending from the request, which loses messages when the send fails after commit, or a managed queue, which adds a service for a single-branch pilot.

The cost: one more process to deploy and watch. While it is down, nothing is sent and new weeks do not publish. When it returns, overdue weeks publish on the first pass, and a reminder that would arrive after the session started is dropped.

## Built for Israel: Hebrew first and one time zone

The first users are gym members and staff in Israel, many of them not technical. Hebrew is the default language and reads right to left.

Every string lives in one catalog with Hebrew and English entries (853 keys each). A Hebrew entry can have masculine, feminine and neutral forms, and the text follows the form of address each person saved. Phone numbers, codes and emails are wrapped in Unicode isolates so they do not flip inside Hebrew text. The catalog is generated into Kotlin for Android. The Windows layout mirrors, with the navigation on the right.

Instants are stored as `timestamptz`. Weekly schedules, publication times and reminders are computed in local time with the Temporal API. Every branch row is pinned to `Asia/Jerusalem` by a check constraint, and DST cases are unit tested.

The alternative was general multi-zone support from the start. The cost: a second country means lifting the pin and reviewing every place that uses the pilot zone constant.

## The desktop app updates from a code-free public repository

Reception and managers use a Windows app. It has to update without an IT department, and the code repository is private.

Releases are built and uploaded by a script on the operator's PC. The script refuses a dirty tree, an unpushed commit or a version that is not newer. The installer, its block map and `latest.yml` go to a public GitHub repository that holds no code, and the installed app reads that feed with electron-updater. A failed check never blocks the app; an update nobody clicked installs at the next start. The API can demand a minimum version: an older app gets HTTP 426 and shows an update screen.

The alternatives were building releases in CI and using the private code repository as the update feed. The decision log rules out the second outright: the private repository is never the feed.

The cost: builds depend on one machine, and the staging installers are unsigned, so Windows shows a warning once per install. Signing is set up in the pipeline and switched off.

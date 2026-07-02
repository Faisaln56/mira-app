# Mira — Reality Audit

**Date:** 2026-07-02
**Scope:** GitHub repo `Faisaln56/mira-app`, Supabase project `personal assistant` (`lvhdacpubdibxahnwylj`), Vercel account.
**Question answered:** What actually exists and works, versus what is claimed, empty, or silently broken?

---

## TL;DR

Mira is **real and alive** as a WhatsApp message loop with working reminders and preference-aware replies. But roughly half the system it presents to the user is hollow: file ingestion doesn't exist, the memory system is almost never written to, "saved" confirmations go to the wrong store, the task list is a write-only junk drawer, and **none of the code that runs Mira is in version control** — this repo is completely empty. There is also one genuine security hole (an `anon`-callable function that can silently steal reminders).

---

## What is REAL ✅

| Component | Evidence |
|---|---|
| WhatsApp conversation loop | 1,308 messages since 2026-05-17, last activity 2026-06-29. Inbound dedup via `processed_messages` (220 rows), `conversation_state` kept current. |
| Reminder delivery | 10 sent, 1 failed after 5 retries. `claim_due_reminders()` uses `FOR UPDATE SKIP LOCKED` + status transitions — a properly engineered worker-queue pattern with real retry logic. |
| Style preferences | `user_preferences` has `address_form = "طال عمرك"` and `reply_verbosity = concise`, and outbound messages demonstrably honor both. |
| WHOOP integration | OAuth refresh token present and rotated as recently as 2026-06-25 — the token refresh loop runs. |
| Profile recall | "وش تعرف عني" returned an accurate profile (Jiwar, Mansard, …) from stored data. |
| Drive image index | 98 images indexed with real data. |

## What is NOT real ❌

### 1. This repository is empty — Mira has no source code anywhere in git
`mira-app` has zero commits, zero files, zero branches. The entire brain (webhook handler, tool router, reminder worker — presumably on Railway/Evolution API per conversation references) exists only in whatever hosted service runs it. **One misclick or platform incident and Mira is unrecoverable.** This is the single largest gap between the system's apparent solidity and its actual state.

### 2. WhatsApp file ingestion does not exist, but the UI pretends it's imminent
Every time a file arrives, Mira replies: *"حفظ ملفات واتساب عبر WAHA قيد التفعيل"* ("being activated"). In the last week alone a **VAT registration certificate PDF** (2026-06-28) and a **workout schedule** (2026-06-29) were sent and silently dropped. The `files` table has had **zero new rows since 2026-05-25**. Users reasonably assume a document handed to an assistant is kept; here it evaporates.

### 3. "Saved it" confirmations that don't save
On 2026-06-25 the user asked "وش احب اكل" → Mira: *no information*. Two minutes later the user said rice is his favorite → Mira: *"محفوظ عندي"* ("saved with me"). Reality: it was written as a **pending task** titled "ملاحظة: فيصل يفضل الرز…" — not into `memory_items` or `user_preferences`. Asking "وش احب اكل" again will return *no information* again. The confirmation is true in letter (a row exists) and false in effect (it will never be recalled).

### 4. The memory system is a shell
After 6 weeks and 1,308 messages: `memory_items` = **2 rows**, `memory_events` = **2 rows**, `tool_log` = **0 rows** (tool logging was designed, never wired). The schema describes a sophisticated memory architecture (confidence, provenance, supersedes chains); the data says it fires roughly once per three weeks.

### 5. The task list is write-only
24 of 25 tasks are `pending`; exactly one has ever been marked `done`. Contents include triplicate reminders ("قوم من الكرسي" ×3), duplicates ("مراجعة فاتورة ساكو" ×2, "إنشاء ملف مشروع بيت خالي" ×2), and misfiled non-tasks — a question ("تقدر تصممها على شكل صورة؟") and free-text notes stored as to-dos. On 2026-06-25 a philosophical question about Gödel's incompleteness theorems received the reply *"تم ربط المهمة بمجال النظام"* — the intent classifier filed a question as a task, twice.

### 6. Drive indexing is stale/empty
`drive_file_index`: **0 rows ever**. `drive_image_index`: last updated 2026-05-25 — no refresh in 5+ weeks. Any answer based on "your files in Drive" is served from a snapshot over a month old, or nothing.

---

## Security findings 🔐

1. **`claim_due_reminders()` is `SECURITY DEFINER` and executable by `anon`** via `/rest/v1/rpc/claim_due_reminders`. Anyone holding the project's anon key (public by design) can atomically claim all due reminders — **reading their contents and flipping them to `processing` so the real worker never sends them**. This is both a data-disclosure and a silent-denial-of-service hole. Fix: `REVOKE EXECUTE ON FUNCTION public.claim_due_reminders(integer) FROM anon, authenticated;` and let only the service-role worker call it. ([linter reference](https://supabase.com/docs/guides/database/database-linter?lint=0028_anon_security_definer_function_executable))
2. **All 15 tables: RLS enabled, zero policies.** Acceptable today (service-role-only backend = deny-all for anon), but it means no authorization model exists for any future client app — a mira-app frontend cannot read anything until policies are written. Flagged so it's a decision, not a surprise.

---

## Recommended order of work

1. **Revoke anon/authenticated EXECUTE on `claim_due_reminders`** — one statement, closes the only live vulnerability.
2. **Get Mira's code into this repo** — export the workflow/functions from wherever they run; the empty repo is the existential risk.
3. **Stop the false "saved" confirmations** — route note/preference utterances to `memory_items`/`user_preferences`, and only say "محفوظ" after the write to the store that recall actually reads.
4. **Ship or stop advertising WAHA file saving** — until it works, the reply should say files are *not* kept, so a VAT certificate isn't silently lost.
5. **Task hygiene** — dedupe, close stale items, and fix the intent classifier that files questions as tasks.
6. **Re-run or schedule Drive indexing** (file index has never run; image index is 5 weeks stale).

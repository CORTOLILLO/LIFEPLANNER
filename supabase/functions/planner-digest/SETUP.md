# Planner snapshot email: setup

Why: scheduled briefs run unattended, and the web fetch tool won't open the planner URL without a live approval. So the planner emails Taylor a snapshot twice a day, and the briefs read it through Gmail (rule 7 of claude/brief-rules.md).

Status: code written and tested against mock data Oct 13. **Not deployed yet**: Taylor does steps 1 to 5 below in the Resend and Supabase dashboards (Claude can't reach Supabase from its sandbox).

What it sends: from onboarding@resend.dev, subject "Planner snapshot AM YYYY-MM-DD" (6:30am ET) and "Planner snapshot PM YYYY-MM-DD" (7:15pm ET). Plain-text tables for yesterday through today+6 (done/total, every item, wins) plus the open list.

## Step 1. Resend (free email sender)
1. Sign up at resend.com **with taylor.n.cortolillo@gmail.com** (the free test sender can only mail the account's own address).
2. API Keys > Create API key (Sending access). Copy it (starts with `re_`). Don't paste it into chat.

## Step 2. Supabase secrets
Supabase dashboard > project tgwdkhjbhhhzevgmuiyi > Edge Functions > Secrets. Add:
- `RESEND_API_KEY` = the key from step 1
- `PLANNER_CODE` = `<access code, see claude/life-admin-setup-status.md in LIFE OPS>`
- `DIGEST_SECRET` = `<DIGEST_SECRET, see claude/planner-snapshot-setup.md in LIFE OPS>`

## Step 3. Deploy the function
Edge Functions > Deploy a new function > Via Editor. Name it `planner-digest`, replace the sample with the code at the bottom of this doc, deploy. Then open its settings and turn **Enforce JWT verification OFF** (the secret header protects it instead).

## Step 4. Schedule it
Database > Extensions: enable `pg_cron` and `pg_net`. Then SQL Editor, run:

```sql
select cron.schedule('planner-digest-am', '30 10,11 * * *', $$
  select net.http_post(
    url := 'https://tgwdkhjbhhhzevgmuiyi.supabase.co/functions/v1/planner-digest?slot=am',
    headers := '{"Content-Type":"application/json","x-digest-secret":"<DIGEST_SECRET, see claude/planner-snapshot-setup.md in LIFE OPS>"}'::jsonb,
    body := '{}'::jsonb);
$$);

select cron.schedule('planner-digest-pm', '15 23,0 * * *', $$
  select net.http_post(
    url := 'https://tgwdkhjbhhhzevgmuiyi.supabase.co/functions/v1/planner-digest?slot=pm',
    headers := '{"Content-Type":"application/json","x-digest-secret":"<DIGEST_SECRET, see claude/planner-snapshot-setup.md in LIFE OPS>"}'::jsonb,
    body := '{}'::jsonb);
$$);
```
Each job fires at both the daylight and standard time UTC offsets; the function only sends when it's the 6am (AM) or 7pm (PM) hour in Eastern, so it's one email each, all year.

## Step 5. Test it now
SQL Editor, run:
```sql
select net.http_post(
  url := 'https://tgwdkhjbhhhzevgmuiyi.supabase.co/functions/v1/planner-digest?slot=test',
  headers := '{"Content-Type":"application/json","x-digest-secret":"<DIGEST_SECRET, see claude/planner-snapshot-setup.md in LIFE OPS>"}'::jsonb,
  body := '{}'::jsonb);
```
A "Planner snapshot TEST" email should land within a minute (check Spam the first time and mark it Not spam). Then tell any LIFE OPS chat "snapshot test arrived" so it can check the email reads cleanly.

## Optional: keep it out of the inbox
Gmail filter: from `onboarding@resend.dev`, subject `Planner snapshot` > Skip the inbox, Mark as read, Never send to spam. The briefs search all mail, so archived is fine.

## To turn it off
SQL Editor: `select cron.unschedule('planner-digest-am'); select cron.unschedule('planner-digest-pm');`

## Code: supabase/functions/planner-digest/index.ts
```ts
// Supabase Edge Function: planner-digest
// Emails Taylor a plain-text snapshot of her planner so the scheduled briefs
// can read it through Gmail (unattended runs can't fetch the planner URL).
//
// Secrets (Edge Functions > Secrets):
//   RESEND_API_KEY  - from resend.com (account signed up with DIGEST_TO's address)
//   PLANNER_CODE    - the claude bridge access code
//   DIGEST_SECRET   - shared secret the cron job sends in x-digest-secret
//   DIGEST_TO       - optional, defaults to taylor.n.cortolillo@gmail.com
// Deploy with "Enforce JWT verification" OFF (the secret header protects it).
//
// Call: POST /functions/v1/planner-digest?slot=am|pm|test
//   am   sends only when it is 6am hour in Eastern (cron fires at both DST offsets)
//   pm   sends only when it is 7pm hour in Eastern
//   test sends right away

const TZ = "America/New_York";

type Row = Record<string, unknown>;

function etParts(d: Date) {
  const f = new Intl.DateTimeFormat("en-CA", {
    timeZone: TZ, year: "numeric", month: "2-digit", day: "2-digit",
    hour: "2-digit", minute: "2-digit", hourCycle: "h23", weekday: "short",
  });
  const p: Record<string, string> = {};
  for (const x of f.formatToParts(d)) p[x.type] = x.value;
  return p;
}

function addDays(ymd: string, n: number): string {
  const d = new Date(ymd + "T12:00:00Z");
  d.setUTCDate(d.getUTCDate() + n);
  return d.toISOString().slice(0, 10);
}

function weekday(ymd: string): string {
  return new Date(ymd + "T12:00:00Z").toLocaleDateString("en-US", { weekday: "short", timeZone: "UTC" });
}

function cell(v: unknown): string {
  if (v === undefined || v === null || v === "") return "-";
  if (typeof v === "object") return JSON.stringify(v);
  return String(v).replace(/\|/g, "/").replace(/\s+/g, " ");
}

function table(rows: Row[], cols: string[]): string {
  if (!rows.length) return "(none)";
  const head = cols.join(" | ");
  return [head, ...rows.map((r) => cols.map((c) => cell(r[c])).join(" | "))].join("\n");
}

async function bridge(base: string, query: string): Promise<Row> {
  const res = await fetch(`${base}?${query}`);
  const text = await res.text();
  if (!res.ok) throw new Error(`${query}: HTTP ${res.status} ${text.slice(0, 200)}`);
  try { return JSON.parse(text); } catch { throw new Error(`${query}: not JSON: ${text.slice(0, 200)}`); }
}

Deno.serve(async (req) => {
  const secret = Deno.env.get("DIGEST_SECRET");
  if (!secret || req.headers.get("x-digest-secret") !== secret) {
    return new Response("forbidden", { status: 403 });
  }

  const slot = new URL(req.url).searchParams.get("slot") ?? "test";
  const now = etParts(new Date());
  const hour = Number(now.hour);
  if (slot === "am" && hour !== 6) return new Response(`skip (ET hour ${hour})`);
  if (slot === "pm" && hour !== 19) return new Response(`skip (ET hour ${hour})`);

  const code = Deno.env.get("PLANNER_CODE");
  const key = Deno.env.get("RESEND_API_KEY");
  const to = Deno.env.get("DIGEST_TO") ?? "taylor.n.cortolillo@gmail.com";
  if (!code || !key) return new Response("missing PLANNER_CODE or RESEND_API_KEY", { status: 500 });

  const base = `${Deno.env.get("SUPABASE_URL")}/functions/v1/claude/${code}`;
  const today = `${now.year}-${now.month}-${now.day}`;
  const label = (n: number) => (n === -1 ? "yesterday" : n === 0 ? "today" : n === 1 ? "tomorrow" : `today+${n}`);

  const dayCols = ["task", "category", "routine", "kind", "checked", "carried_over", "start", "end", "priority"];
  const listCols = ["task", "category", "due", "added", "priority", "carries_forward"];
  const out: string[] = [];
  const errors: string[] = [];

  out.push(`PLANNER SNAPSHOT (${slot.toUpperCase()})`);
  out.push(`Taken ${now.weekday} ${today} ${now.hour}:${now.minute} Eastern. Read-only copy of the planner bridge (do=day and do=list).`);
  out.push("");

  for (let n = -1; n <= 6; n++) {
    const date = addDays(today, n);
    try {
      const d = await bridge(base, `do=day&date=${date}`);
      const items = (d.items as Row[]) ?? [];
      const wins = (d.wins as unknown[]) ?? [];
      out.push(`=== DAY ${date} ${weekday(date)} (${label(n)}) | done ${cell(d.done)} of ${cell(d.total)} ===`);
      out.push(table(items, dayCols));
      out.push(`wins: ${wins.length ? wins.map(cell).join("; ") : "(none)"}`);
      out.push("");
    } catch (e) {
      errors.push(String(e));
      out.push(`=== DAY ${date} (${label(n)}) | ERROR, could not read ===`, "");
    }
  }

  try {
    const l = await bridge(base, "do=list");
    const items = (l.open_items as Row[]) ?? [];
    out.push(`=== OPEN LIST (${items.length} items) ===`);
    out.push(table(items, listCols));
  } catch (e) {
    errors.push(String(e));
    out.push("=== OPEN LIST | ERROR, could not read ===");
  }
  out.push("", "=== END OF SNAPSHOT ===");
  if (errors.length) out.push("", "Errors:", ...errors);

  const text = out.join("\n");
  const html = `<pre style="font-family:monospace;font-size:12px;white-space:pre-wrap">${
    text.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;")
  }</pre>`;

  const send = await fetch("https://api.resend.com/emails", {
    method: "POST",
    headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      from: "LIFEPLANNER <onboarding@resend.dev>",
      to: [to],
      subject: `Planner snapshot ${slot.toUpperCase()} ${today}`,
      text,
      html,
    }),
  });
  const result = await send.text();
  return new Response(`${send.ok ? "sent" : "send failed"}: ${result}`, { status: send.ok ? 200 : 502 });
});
```

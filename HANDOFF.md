# Handoff: Monsey's Best / Old World Artisans

Start here in a new Claude chat. Say: "Read HANDOFF.md and continue."

## Setup
- The app code is in **monseysbest/monseys-best-app**. This repo (monseys-best) only holds this note and the
  `.claude/settings.json` allow rule, which lets Supabase SQL and migrations run without a prompt. Attach the app repo
  with add_repo (push access), clone it, then work on a `claude/...` branch.
- Supabase project `gvlbkakdjiywvcpgthcj`; Vercel team `monseysbest613`. Edge functions deploy on merge to main
  (a GitHub Action, `--no-verify-jwt`). The Next.js app deploys on Vercel.
- Every database change is logged in `monseys-best-app/docs/sql/2026-11-04-discounts-prepay.sql`. Add to it.

## How we work (the owner's standing rules)
- Fewer words. Short, plain answers, and cut-paste blocks when something is for another helper.
- Merge only after CI is green. An independent Claude review comes before anything goes live, so run a review agent
  on every PR.
- Customer texts and emails go live only with the owner's direct, confirmed OK (`settings.sms_live` is false; email
  is gated by `shopify_notify_to`). Don't ask about the live switch.
- Never commit customer personal data (phones, emails, names) or secrets. Stripe keys never go in app_secrets.
- Shabbos: fail closed. Database changes: additive. Don't merge `claude/project-thread-ezk9qc`.
- The owner prefers that we do things ourselves (use SQL rather than asking) and only ask him for a "go" when a tool
  truly needs it.

## The text bot (`supabase/functions/text-bot/index.ts`)
- It runs on Opus 5.5. Its behavior comes from the `RULES` prompt, the tools, and the reply filter (`replyProblem`).
- Owner mode: it answers only testers (`settings.text_bot_testers`: the owner's wife and a staff member) and the owner.
- Practice runs: POST `{kind:'simulate', scenarios:[{name, customer?, texts:[...]}]}` to text-bot with the
  `x-notify-secret` header. Nothing is sent or saved. To run one from SQL:
  `select net.http_post(url := replace((select value from public.app_secrets where key='notify_url'),'/notify','/text-bot'), headers := jsonb_build_object('Content-Type','application/json','x-notify-secret',(select value from public.app_secrets where key='notify_secret')), body := '<json>'::jsonb, timeout_milliseconds := 130000);`
  then read `net._http_response` by id about 2 minutes later. Limits: 12 scenarios per call, 15 texts each.
  Practice customers have no account, so balance questions can't be tested this way.
- The owner's direction (Nov 5): the bot is the customer's main help. It answers, asks follow-up questions, and offers
  a person instead of handing off. It hands off right away only if they ask for a person, are angry, or a tool says so.
  Complaints get an apology and a question first. Pranks get one sweet line (`hand_off.final_text`) and the bot goes
  quiet. No photos unless asked: `send_menu_picture` sends one numbered picture of every bread
  (`public/menu-collage.jpg`; keep its numbering in step with `MENU_PICTURE_ORDER`).
- Codes: `propose_order.code` is required (FAMILY90 was lost once when it was optional). Unknown codes get "I don't see
  that code".
- Email: an email given by text goes to `customers.unverified_email` and gets a "Confirm your email" link
  (`email_verify_requests`, account-link `?v=`). Only once confirmed does it join `customers.emails`, which is used
  for sign-in. In test mode no confirm link is sent.
- Order reminders: offered once (`offer_reminders`; the server appends the exact wording). Consent goes in
  `customers.order_reminders` (`merge_order_reminders`). STOP turns them off.
- Balance policy: $120 limit and 30 days by default; per customer in the chat's Customer panel. `propose_order` says
  exactly what to pay to unlock an order (`toUnlock`).

## What's live (Nov 5)
- Admin → Zelle: a searchable customer picker with a confirm step. Customer numbers are 4 digits (C1001…; the sequence
  stops at 9999 on purpose).
- Shared phone: the bot lists the accounts on that phone and asks which; "both are me" merges them (`merge_customers`).
- Blocked numbers: the Block button in the inbox (`blocked_phones`).
- Phone alerts open the installed app, not a Chrome tab, and come forward right away.
- "Everything Seeded" is the bread's name everywhere (products, Shopify title, bot, `bread_key`).

## Open
1. **Reminder sender:** nothing sends order reminders yet. Build a weekly job for `order_reminders.on = true` with
   carrier consent. Each text must include the brand name and "Reply STOP to unsubscribe", and match the campaign's
   registered samples (Twilio A2P: Low Volume Mixed, verified; ask the owner to read the samples from the console).
   Skip numbers that texted STOP. Keep it gated by `sms_live`.
2. **More bot practice:** keep throwing odd conversations at it (topic jumps, pranks, typos, weird questions) and fix
   whatever it fumbles.
3. **12 accounts with balances:** the owner is to check them; Shopify "zelle-…" tags hint at Zelles that were never
   recorded.
4. **Move ordering off Shopify:** sizes on; checkout opens only with the owner's OK right before; stop Shopify writes.
5. **Android driver-app signing key:** rotate it at the next rebuild (it's in git; drivers reinstall once).
6. Photos of halves and bilkelach (from the owner). RCS only if wanted. A per-phone reply cap for the bot.

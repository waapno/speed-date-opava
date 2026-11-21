# SpeedDating Ostrava — open source

Prestižní speed dating portál pro Ostravu a Moravskoslezský kraj.
Termíny, rezervace s kontrolou kapacit, platby (Stripe / PayPal),
potvrzovací e-maily a automatický přesun termínu při nenaplnění kapacity.

> **Open source & copyright**
> Tento projekt je open source pod licencí MIT s povinností uvedení zdroje
> (viz [`LICENSE`](./LICENSE)). Při jakémkoli použití, úpravě nebo nasazení
> kódu **musíte viditelně uvést odkaz na původní zdroj**:
> <https://github.com/speeddating-ostrava/speeddating-ostrava>
>
> © 2026 SpeedDating Ostrava. This project is open source (MIT + attribution).
> Any use or redistribution must include a visible link back to the source
> repository above.

## Hosting

Kód lze hostovat na GitHubu (GitHub Pages / GitHub Actions build) nebo na
vlastním serveru. Backendová část (webhooky, e-maily, správa termínů) běží
jako samostatný **Cloudflare Worker** ve složce [`workers/`](./workers) —
nezávisle na Lovable.

## Architektura

| Vrstva | Technologie |
| --- | --- |
| Frontend | TanStack Start + React + Tailwind |
| Termíny (obsah) | JSONBin (přes Cloudflare Worker) |
| Rezervace & kapacity | Postgres (atomická RPC funkce, ochrana proti přeplnění) |
| Platby | Stripe + PayPal webhooky ve Workeru |
| E-maily | Resend |
| Cron (rollover) | Cloudflare Worker `scheduled()` |

## Administrace termínů

Na `/admin` je rozhraní pro přidávání, úpravu a rušení termínů. Přihlašuje se
adresou Workeru + `ADMIN_TOKEN`; data se ukládají do JSONBin a Worker je
zrcadlí do databáze, aby zůstala zachovaná bezpečná kontrola kapacit.
Detailní nastavení viz [`workers/README.md`](./workers/README.md).

## Vývoj

```sh
git clone <this-repository-url>
cd <repository-name>
npm i
npm run dev
```

## Produkční nasazení (krok za krokem)

1. **Databáze** — vytvořte projekt Supabase a spusťte migrace ze `supabase/migrations/` (`supabase db push`).
2. **Worker (platby, e-maily, cron)** — `cd workers && wrangler deploy` a nastavte secrets
   (viz `workers/README.md`). Povinné pro ostrý provoz: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`,
   `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET`,
   `PAYPAL_WEBHOOK_ID`, `PAYPAL_ENV=live`, `RESEND_API_KEY`, `RESEND_FROM`, `ADMIN_TOKEN`,
   `ROLLOVER_TOKEN`, `PUBLIC_BASE_URL`, `WORKER_PUBLIC_URL`. **Nenastavujte `TEST_MODE`.**
3. **Web** — zkopírujte `.env.example` → `.env`, vyplňte, pak `npm i && npm run build`
   a nasaďte `dist/` (Cloudflare: `npx wrangler deploy` dle `dist/server/wrangler.json`).
4. **Webhooky** — Stripe: `https://<worker>/stripe` (`checkout.session.completed`);
   PayPal: `https://<worker>/paypal` (`PAYMENT.CAPTURE.COMPLETED`).
5. **Admin** — `https://<web>/admin`, přihlášení URL Workeru + `ADMIN_TOKEN`.

## Co je nového v této verzi

- **Admin dashboard**: Přehled (tržby, tržby 30 dní, vráceno, obsazenost, M/Ž),
  Rezervace (hledání, filtr dle stavu/termínu, export CSV, ruční potvrzení, zrušení,
  refundace Stripe/PayPal jedním klikem, opětovné odeslání e-mailu), Termíny.
- **Bezpečnost plateb**:
  - Stripe webhook ověřuje `payment_status = paid` a shodu částky a měny.
  - PayPal návrat (`/paypal/return`) nově ověřuje objednávku u PayPalu (stav, `custom_id`, částka) —
    dříve šlo jednou zaplacenou objednávkou potvrdit cizí rezervaci.
  - Rezervace nelze vložit přímo do DB s vlastní cenou — jen přes atomickou RPC (migrace `hardening`).
  - `TEST_MODE` se ignoruje, pokud jsou nastavené ostré klíče.
  - Odstraněn nechráněný veřejný endpoint `rollover-check` (rollover běží jen ve Workeru přes cron / token).
  - Admin token se drží jen v `sessionStorage`.
- **Platba**: stránka po platbě skutečně ověřuje stav rezervace (dřív vždy hlásila úspěch);
  bez nastaveného Workeru se rezervace nevytvoří „naoko“.
- **Kapacity**: cron každou hodinu uvolní nezaplacené rezervace starší 30 minut.
- `.env` odstraněn z repozitáře, přidán `.env.example`.
- Odstraněny duplicitní webhooky v aplikaci (`/api/public/webhooks/*`) — bez nastaveného
  secretu šlo zdarma potvrdit libovolnou rezervaci. Platby zpracovává výhradně Worker.

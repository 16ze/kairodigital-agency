# Phase 1: MVP Core - Research

**Researched:** 2026-04-11
**Domain:** Multi-tenant SaaS booking (Next.js 15 + Supabase + pawaPay + Twilio) pour marché gabonais
**Confidence:** HIGH (stack et patterns), MEDIUM-HIGH (pawaPay specifics — doc v2 officielle vérifiée)

---

## Summary

Phase 1 construit le coeur MVP de Bookly : auth OTP téléphone, onboarding pro, moteur de disponibilités atomique en PostgreSQL, flux de booking client, paiements Airtel Money via pawaPay, notifications SMS Twilio, agenda Realtime, walk-ins, et dashboard CA. L'architecture est shared schema + RLS PostgreSQL avec `salon_id` injecté dans le JWT Supabase via un Custom Access Token Hook.

Les pièces les plus risquées sont (1) la contrainte anti double-booking `btree_gist` combinée à l'état `pending_payment` + slot-lock 10 min, (2) la vérification de signature RFC-9421 sur les webhooks pawaPay (ce n'est pas un simple HMAC — c'est RSASSA/ECDSA avec clé publique), (3) l'injection correcte du `salon_id` dans le JWT sans casser l'onboarding (l'utilisateur n'a pas de salon_id au premier signup), et (4) la gestion du montant XAF en string décimal côté API pawaPay alors qu'on stocke en INTEGER en DB.

**Primary recommendation:** Commencer par le schema DB + RLS + hook JWT (Plan 01-01), car c'est la fondation dont tout le reste dépend. Paralléliser auth OTP pendant que le KYB pawaPay avance. Intégrer pawaPay en dernier (Plan 01-06) pour ne pas bloquer le reste si le KYB prend 4 semaines.

---

## User Constraints

> Pas de CONTEXT.md en amont (le phase a été lancé directement en `/gsd:plan-phase 1`). Les contraintes viennent de PROJECT.md, ROADMAP.md, et de la recherche amont validée dans `.planning/research/`.

### Locked Decisions (de ROADMAP.md + research synthesis)

- **Stack frontend :** Next.js 15 (App Router, React 19, Turbopack), TypeScript strict, Tailwind CSS 3.4, shadcn/ui, React Hook Form 7, Zod 3, Zustand 5 (state agenda uniquement)
- **Backend :** Supabase Cloud région London (eu-west-2), PostgreSQL + RLS, Supabase Phone Auth (OTP), Supabase Realtime, Edge Functions (Deno) pour webhooks
- **ORM :** Prisma 5.x pour migrations UNIQUEMENT — toutes les queries runtime passent par le Supabase JS client (pour que RLS s'applique automatiquement via le JWT)
- **Server Actions :** toutes les mutations passent par des Server Actions Next.js 15 (pas d'API routes)
- **Paiement mobile money :** pawaPay pour Airtel Money Gabon (`AIRTEL_GAB`). Moov Money Gabon **NON supporté par pawaPay** (vérifié dans la doc providers) — à retirer du scope Phase 1
- **Paiement CB :** Stripe via entité française Kairo Digital (secondaire, Phase 1 minimal)
- **SMS :** Twilio uniquement (Africa's Talking ne couvre pas le Gabon)
- **Devise :** XAF (FCFA) stockée en INTEGER en DB, format d'affichage `9 900 FCFA` (espace séparateur)
- **Auth :** OTP téléphone uniquement en v1, email/password exclu
- **Walk-ins :** P0 (40-60% des visites en Afrique)
- **Timezone :** Africa/Libreville (UTC+1, pas de DST depuis 1912)
- **Multi-tenant :** shared schema + RLS, `salon_id` injecté dans JWT via Custom Access Token Hook

### Claude's Discretion (à décider pendant le planning)

- Nombre exact de Server Actions par feature et découpage fichiers
- Structure précise des composants UI (shadcn blocks vs custom)
- Choix cron : `pg_cron` (Supabase) vs Vercel Cron vs Supabase Edge Function scheduled — voir recommandation dans la section Notifications plus bas
- Format exact des templates SMS (à valider avec le contenu produit)
- Structure exacte du wizard onboarding (linéaire 4 étapes vs progressif)

### Deferred Ideas (OUT OF SCOPE Phase 1)

- PWA offline (Phase 2)
- CRM client / historique (Phase 2)
- POS caisse complet (Phase 2)
- WhatsApp notifications (Phase 2, bloqué par délai Meta 4-8 semaines)
- Fidélité / cartes cadeaux (Phase 2)
- Acomptes / prépaiements (Phase 2)
- Analytics avancés / taux d'occupation (Phase 3)
- Campagnes SMS marketing (Phase 3)
- Payout Airtel vers le pro (Phase 3)
- Moov Money (provider non supporté par pawaPay pour Gabon)
- App mobile native (Phase 4)
- Marketplace publique (Phase 4)

---

## Phase Requirements

| ID | Description | Research Support (quelle pièce technique) |
|----|-------------|---------------------------------------------|
| AUTH-01 | Pro signup via OTP SMS | Supabase Phone Auth + Twilio (Plan 01-02) |
| AUTH-02 | Client signup via OTP SMS | Supabase Phone Auth + Twilio (Plan 01-02, 01-05) |
| AUTH-03 | Login sans mot de passe | `signInWithOtp` + `verifyOtp` |
| AUTH-04 | Session persistante | JWT Supabase + cookies httpOnly (Next.js middleware) |
| AUTH-05 | Isolation RLS par salon_id | Custom Access Token Hook injecte `salon_id` dans JWT (Plan 01-01) |
| ONBOARD-01 | Créer salon | Table `salons` + Server Action |
| ONBOARD-02 | Ajouter services FCFA | Table `services` INTEGER price |
| ONBOARD-03 | Horaires par jour | Tables `staff_working_hours` / `salon_business_hours` |
| ONBOARD-04 | Collaborateurs + services assignés | Tables `staff_members` + `service_staff` (junction) |
| ONBOARD-05 | Lien partageable unique | Colonne `slug` UNIQUE sur `salons` + route `/[slug]` publique |
| BOOKING-01 | Page publique sans compte | RLS policy `is_published = true` en lecture anon |
| BOOKING-02 | Choix service/staff/créneau | `get_available_slots()` + UI 3 étapes |
| BOOKING-03 | Slots calculés temps réel | Fonction PostgreSQL `get_available_slots()` |
| BOOKING-04 | Anti double-booking DB level | EXCLUDE constraint `btree_gist` + WHERE partial |
| BOOKING-05 | SMS confirmation client | `notification_queue` + pg_cron + Twilio |
| BOOKING-06 | SMS notification pro | `notification_queue` trigger sur INSERT appointment |
| BOOKING-07 | Annulation via lien SMS | Token signé dans URL SMS + Server Action cancel |
| BOOKING-08 | Lock 10 min pending_payment | Status `pending_payment` + pg_cron cleanup 5 min |
| WALKIN-01 | Walk-in direct agenda | INSERT appointment `type='walk_in', status='confirmed'` |
| WALKIN-02 | Saisie minimale | Formulaire : nom OU téléphone |
| WALKIN-03 | Indicateur visuel | Champ `type` + styling agenda |
| WALKIN-04 | Encaissement immédiat | Bouton "Encaisser" depuis agenda → Server Action payment cash |
| AGENDA-01 | Vue jour | Server Component + query par date |
| AGENDA-02 | Vue semaine | Composant switch + query 7 jours |
| AGENDA-03 | Création manuelle RDV | Server Action réutilisée du flow booking |
| AGENDA-04 | Modifier/annuler | Server Actions update/cancel |
| AGENDA-05 | Sync Realtime | `supabase.channel('agenda:{salon_id}')` + postgres_changes |
| AGENDA-06 | Mobile responsive | Tailwind breakpoints + touch-friendly shadcn |
| PAYMENT-01 | Airtel Money via pawaPay | POST `/v2/deposits` sandbox puis prod |
| PAYMENT-02 | Cash manuel | INSERT `payments (method='cash', status='completed')` |
| PAYMENT-03 | Webhook async | Edge Function + signature RFC-9421 |
| PAYMENT-04 | Edge Function dédié par provider | `supabase/functions/pawapay-webhook/` |
| PAYMENT-05 | INTEGER FCFA | Colonne `amount INTEGER NOT NULL` |
| PAYMENT-06 | Historique transactions | Liste Server Component + filtres |
| NOTIF-01 | SMS confirmation client | notification_queue template `booking_confirmation_client` |
| NOTIF-02 | SMS rappel 24h | pg_cron scheduled job + template `booking_reminder_24h` |
| NOTIF-03 | SMS notification pro | notification_queue template `booking_notification_pro` |
| NOTIF-04 | Table notification_queue async | Trigger INSERT → queue → worker |
| NOTIF-05 | Retry 3 tentatives | Colonne `attempts` + worker logic |
| DASHBOARD-01 | CA du jour | Aggregate query `payments` status='completed' AND paid_at::date = today |
| DASHBOARD-02 | CA du mois | Aggregate query + date_trunc('month') |
| DASHBOARD-03 | Nombre RDV jour | Count appointments par status |
| DASHBOARD-04 | RDV prochains 24h | Query appointments start_time BETWEEN now AND now+24h |

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Next.js | 15.x | Framework web (App Router, RSC, Server Actions) | Stack non négociable projet — React 19, Turbopack stable |
| React | 19.x | UI runtime | Inclus Next.js 15 |
| TypeScript | 5.5+ strict | Type safety | Convention non négociable |
| Tailwind CSS | 3.4.x | Styling | Mobile-first, utility-first |
| shadcn/ui | latest | Composants Radix | Accessibilité + customisation |
| @supabase/supabase-js | 2.x | Client Supabase (Auth + DB + Realtime) | Runtime queries (JWT → RLS auto) |
| @supabase/ssr | 0.5.x+ | Helpers SSR Next.js pour cookies | **Remplace `@supabase/auth-helpers-nextjs` (déprécié)** — utilise `createServerClient` + `createBrowserClient` |
| Prisma | 5.x | Migrations schema uniquement | ORM pour `prisma migrate` — **JAMAIS utilisé en runtime** |
| Zod | 3.x | Validation entrées | Convention projet — sur tous les Server Actions |
| React Hook Form | 7.x | Formulaires | Pair avec Zod via `zodResolver` |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| date-fns | 3.x | Manipulation dates | Calcul plages horaires, format FR |
| date-fns-tz | 3.x | Timezone-aware formatting | `formatInTimeZone(utc, 'Africa/Libreville', 'HH:mm')` |
| twilio | 5.x (Node SDK) | Envoi SMS depuis Edge Function worker | Worker `notification_queue` |
| zustand | 5.x | State client agenda | Uniquement pour l'agenda (sync Realtime) — pas pour l'état global |
| libphonenumber-js | 1.11+ | Validation numéros +241 | Format gabonais `+241 XX XX XX XX` |
| nanoid | 5.x | Slugs uniques salons | `bookly.app/{slug}` |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| pg_cron (Supabase) pour worker notification_queue | Vercel Cron ou Edge Function scheduled | pg_cron = DB-level, pas de HTTP overhead, mais ne peut pas appeler Twilio directement — doit trigger une Edge Function. **Recommandé : pg_cron toutes les 60s → invoque Edge Function `process-notification-queue`** |
| `@supabase/auth-helpers-nextjs` | `@supabase/ssr` | auth-helpers est **déprécié** — ne pas utiliser sur un nouveau projet |
| Drizzle ORM | Prisma | Prisma déjà locked dans la stack projet |
| Twilio Verify (OTP managed) | Supabase Phone Auth native | Supabase Phone Auth utilise Twilio en sous-capot et gère la session + RLS — pas de double système |

**Installation (à ajouter au plan 01-01) :**
```bash
npm install next@15 react@19 react-dom@19
npm install @supabase/supabase-js @supabase/ssr
npm install zod react-hook-form @hookform/resolvers
npm install date-fns date-fns-tz libphonenumber-js nanoid zustand
npm install -D prisma typescript @types/node @types/react @types/react-dom
npm install -D tailwindcss postcss autoprefixer
```

**Version verification (à exécuter au début du Plan 01-01) :**
```bash
npm view next version          # Confirmer 15.x actuel
npm view @supabase/ssr version # Confirmer 0.5.x+
npm view prisma version        # Confirmer 5.x (pas 6 si breaking)
npm view @supabase/supabase-js version
npm view twilio version
```
Documenter les versions exactes dans `package.json`. Les versions dans ce doc sont des minimums.

---

## Architecture Patterns

### Recommended Project Structure

```
src/
├── app/
│   ├── (marketing)/           # landing, pricing public
│   ├── (auth)/
│   │   ├── login/             # OTP flow
│   │   └── verify/
│   ├── (pro)/                 # Dashboard pro (authentifié)
│   │   ├── onboarding/        # Wizard 4 étapes
│   │   ├── agenda/            # Vue jour/semaine + Realtime
│   │   ├── dashboard/         # CA + stats
│   │   ├── settings/
│   │   │   ├── services/
│   │   │   ├── staff/
│   │   │   └── hours/
│   │   └── layout.tsx         # Guard auth + check onboarding complet
│   ├── [slug]/                # Page publique salon + booking (client non authentifié)
│   │   ├── page.tsx           # Présentation salon + services
│   │   ├── book/              # Tunnel réservation
│   │   └── manage/[token]/    # Annulation via lien SMS
│   └── api/                   # Uniquement pour webhooks non migrables en Edge Function
├── lib/
│   ├── supabase/
│   │   ├── server.ts          # createServerClient (cookies)
│   │   ├── client.ts          # createBrowserClient
│   │   ├── middleware.ts      # refresh session
│   │   └── admin.ts           # service_role (Edge Functions uniquement)
│   ├── actions/               # Server Actions
│   │   ├── auth.ts
│   │   ├── salon.ts
│   │   ├── booking.ts
│   │   ├── agenda.ts
│   │   └── payment.ts
│   ├── schemas/               # Zod schemas
│   ├── pawapay/               # Client pawaPay (deposits, signature check helpers)
│   ├── twilio/                # Client Twilio (wrapper)
│   └── format/                # formatXAF, formatPhone, formatTimeLibreville
├── components/
│   ├── ui/                    # shadcn
│   ├── agenda/                # Composants agenda jour/semaine
│   ├── booking/               # Composants tunnel
│   └── onboarding/            # Composants wizard
├── hooks/                     # use-agenda-realtime, use-slot-picker
├── middleware.ts              # Auth middleware Next.js
prisma/
├── schema.prisma              # Source de vérité schema
└── migrations/                # Migrations SQL générées
supabase/
├── functions/                 # Edge Functions Deno
│   ├── pawapay-webhook/
│   └── process-notification-queue/
└── migrations/                # Migrations manuelles RLS, functions, triggers, pg_cron
```

### Pattern 1: Server Action avec Result type + Zod

```typescript
// lib/actions/booking.ts
"use server"
import { z } from "zod"
import { createServerClient } from "@/lib/supabase/server"
import { cookies } from "next/headers"

type Result<T> = { data: T; error: null } | { data: null; error: string }

const CreateBookingSchema = z.object({
  salonId: z.string().uuid(),
  serviceId: z.string().uuid(),
  staffId: z.string().uuid(),
  startTime: z.string().datetime(),
  clientPhone: z.string().regex(/^\+241\d{8}$/, "Numéro gabonais invalide"),
  clientName: z.string().min(1).max(100),
})

export async function createBooking(
  input: z.infer<typeof CreateBookingSchema>
): Promise<Result<{ appointmentId: string; paymentRequired: boolean }>> {
  const parsed = CreateBookingSchema.safeParse(input)
  if (!parsed.success) {
    return { data: null, error: parsed.error.issues[0].message }
  }

  const supabase = createServerClient(cookies())

  // RPC atomique — jamais SELECT puis INSERT en JS
  const { data, error } = await supabase.rpc("create_booking", {
    p_salon_id: parsed.data.salonId,
    p_service_id: parsed.data.serviceId,
    p_staff_id: parsed.data.staffId,
    p_start_time: parsed.data.startTime,
    p_client_phone: parsed.data.clientPhone,
    p_client_name: parsed.data.clientName,
  })

  if (error) {
    // PostgreSQL exclusion violation = double-booking tenté
    if (error.code === "23P01") {
      return { data: null, error: "Ce créneau vient d'être pris" }
    }
    return { data: null, error: "Erreur lors de la réservation" }
  }

  return { data: { appointmentId: data.id, paymentRequired: data.requires_payment }, error: null }
}
```

### Pattern 2: Supabase SSR (Next.js 15 cookies)

```typescript
// lib/supabase/server.ts
import { createServerClient as createSupabaseServerClient } from "@supabase/ssr"
import { cookies } from "next/headers"

export async function createServerClient() {
  const cookieStore = await cookies() // Next.js 15 : cookies() est async
  return createSupabaseServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookies) {
          try {
            cookies.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Server Component context — ignore
          }
        },
      },
    }
  )
}
```

**Gotcha Next.js 15 :** `cookies()`, `headers()`, `params`, `searchParams` sont désormais **asynchrones** (retournent des Promises). Oublier `await` casse le build en mode strict.

### Pattern 3: Realtime agenda avec cleanup

```typescript
// hooks/use-agenda-realtime.ts
"use client"
import { useEffect } from "react"
import { createBrowserClient } from "@/lib/supabase/client"

export function useAgendaRealtime(salonId: string, onChange: (payload: unknown) => void) {
  useEffect(() => {
    const supabase = createBrowserClient()
    // 1. Charger l'état initial DANS le composant parent AVANT ce hook
    // 2. Subscribe (ne jamais subscribe avant le fetch initial — risque d'updates manqués)
    const channel = supabase.channel(`agenda:${salonId}`)
      .on("postgres_changes", {
        event: "*",
        schema: "public",
        table: "appointments",
        filter: `salon_id=eq.${salonId}`,
      }, onChange)
      .subscribe()

    return () => {
      supabase.removeChannel(channel) // cleanup obligatoire
    }
  }, [salonId, onChange])
}
```

### Pattern 4: Custom Access Token Hook (salon_id dans JWT)

```sql
-- supabase/migrations/xxx_auth_hook.sql
CREATE OR REPLACE FUNCTION public.custom_access_token_hook(event jsonb)
RETURNS jsonb
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
  v_salon_id uuid;
  v_role text;
  v_claims jsonb;
BEGIN
  v_claims := event->'claims';

  -- Le user peut avoir plusieurs memberships (owner + staff d'un autre salon)
  -- En Phase 1 : un seul salon par user → simple SELECT
  SELECT sm.salon_id, sm.role
    INTO v_salon_id, v_role
  FROM public.salon_memberships sm
  WHERE sm.user_id = (event->>'user_id')::uuid
    AND sm.is_active = true
  ORDER BY sm.created_at ASC
  LIMIT 1;

  IF v_salon_id IS NOT NULL THEN
    v_claims := jsonb_set(v_claims, '{salon_id}', to_jsonb(v_salon_id::text));
    v_claims := jsonb_set(v_claims, '{user_role}', to_jsonb(coalesce(v_role, 'owner')));
  END IF;

  -- Pas de salon_id = user vient de signup, en cours d'onboarding
  -- Le client code check `claims.salon_id` et redirige vers /onboarding si null

  event := jsonb_set(event, '{claims}', v_claims);
  RETURN event;
END;
$$;

-- Permissions (OBLIGATOIRES sinon le hook ne s'exécute pas)
GRANT EXECUTE ON FUNCTION public.custom_access_token_hook TO supabase_auth_admin;
GRANT USAGE ON SCHEMA public TO supabase_auth_admin;
REVOKE EXECUTE ON FUNCTION public.custom_access_token_hook FROM authenticated, anon, public;

-- RLS doit permettre à supabase_auth_admin de lire salon_memberships
CREATE POLICY "auth_admin_can_read_memberships"
  ON public.salon_memberships
  FOR SELECT
  TO supabase_auth_admin
  USING (true);
```

**Enregistrement du hook** (Supabase Dashboard → Auth → Hooks → Custom Access Token → pointe vers `public.custom_access_token_hook`).

### Anti-Patterns to Avoid

- **Calcul des slots en TypeScript** → race conditions, fait exploser la complexité. Toujours via `get_available_slots()` SQL.
- **SELECT puis INSERT** pour vérifier un créneau → double-booking garanti à la concurrence. Toujours une RPC atomique + EXCLUDE constraint.
- **Appels Twilio/pawaPay depuis un DB trigger** → bloque la transaction, rien ne récupère si timeout. Toujours `notification_queue` + worker async.
- **Prisma en runtime** → bypass RLS silencieux, fuite cross-tenant. Prisma migrate-only, stop.
- **Subscribe Realtime avant fetch initial** → updates manqués entre le subscribe et le fetch. Toujours fetch puis subscribe.
- **Policy RLS `USING (EXISTS (SELECT ... FROM salons WHERE ...))`** → subquery par ligne, O(n²). Toujours `USING (salon_id = (auth.jwt() ->> 'salon_id')::uuid)` avec index.
- **`cookies()` sans `await` en Next.js 15** → erreur build strict.
- **Oublier `REPLICA IDENTITY FULL`** sur `appointments` si on veut le `old_record` dans les events Realtime UPDATE/DELETE → sinon on perd l'info pour la reconciliation.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Calcul de disponibilités | Loop JS sur les slots | `get_available_slots()` PostgreSQL | Atomicité + perf + pas de round-trips |
| Anti double-booking | Mutex applicatif / SELECT FOR UPDATE | EXCLUDE constraint `btree_gist` | Garantie DB-level, zéro code |
| OTP SMS | Table `otp_codes` + envoi manuel | Supabase Phone Auth (Twilio integrated) | Rate limits + session + RLS tout-en-un |
| Queue notifications | Worker maison + Redis | Table `notification_queue` + pg_cron → Edge Function worker | Durable, observable, pas d'infra additionnelle |
| Signature webhook pawaPay | `crypto.createHmac("sha256")` | Impl RFC-9421 avec `http-message-signatures` ou sample officiel pawaPay Node | pawaPay utilise RSA/ECDSA, PAS HMAC — c'est une signature RFC-9421 asymétrique |
| Multi-tenant isolation | `WHERE salon_id = ?` en code | RLS PostgreSQL + JWT claim | Un oubli = fuite cross-tenant |
| Validation numéros +241 | Regex maison | `libphonenumber-js` (parsePhoneNumber('GA')) | Formats variés, normalisation |
| Format FCFA | `toLocaleString('fr-FR')` bricolé | `Intl.NumberFormat('fr-FR', { style: 'currency', currency: 'XAF', maximumFractionDigits: 0 })` | Devise zéro-décimal gérée nativement |
| Slugs uniques salons | Compteur + check | `nanoid` ou `slugify + nanoid suffix` sur collision | Collision-safe, URL-safe |

**Key insight :** Tous les problèmes difficiles (concurrence, signature, queue, isolation) sont adressés par des primitives DB/SDK. Hand-roller, c'est réinventer des bugs.

---

## Common Pitfalls

### P1 — Double-booking race condition

**What goes wrong :** Deux clients cliquent le même créneau à 100ms d'intervalle. Les deux SELECT voient le créneau libre. Les deux INSERT réussissent. Double-booking.

**Why it happens :** Pas de verrou atomique entre le check et l'insert côté application.

**How to avoid :**
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE appointments
ADD CONSTRAINT no_double_booking
EXCLUDE USING gist (
  staff_id WITH =,
  tstzrange(start_time, end_time, '[)') WITH &&
) WHERE (status IN ('pending_payment', 'confirmed'));
```

La clause `WHERE` est **supportée** (confirmé via doc officielle PostgreSQL — crée un index partiel). Les RDV `cancelled`/`completed` ne bloquent plus de nouveaux slots.

**Warning signs :** Tester avec 2 navigateurs en parallèle sur le même slot en dev.

---

### P2 — pawaPay webhook : signature RFC-9421 asymétrique (PAS HMAC)

**What goes wrong :** On code un `crypto.createHmac('sha256', secret)` comme pour Stripe. Ça ne matchera jamais — pawaPay utilise des signatures **asymétriques** RFC-9421 (RSA-PSS-SHA512, RSA-PKCS-SHA256, ECDSA P-256-SHA256, ECDSA P-384-SHA384).

**Why it happens :** L'habitude Stripe/GitHub = HMAC. pawaPay suit RFC-9421 (HTTP Message Signatures).

**How to avoid :**

1. Dans le dashboard pawaPay, activer "Signed callbacks" et fournir notre **clé publique**. pawaPay signe avec leur clé privée, on vérifie avec leur clé publique (à eux).
2. En fait c'est l'inverse : **pawaPay fournit sa clé publique**, on vérifie avec. Lire les headers `Signature`, `Signature-Input`, `Content-Digest`, `Signature-Date`.
3. Utiliser un lib qui implémente RFC-9421 (ex: `http-message-signatures` npm) OU adapter le sample officiel : https://github.com/pawaPay/signatures-node-example

**Edge Function Deno skeleton :**
```typescript
// supabase/functions/pawapay-webhook/index.ts
import { serve } from "https://deno.land/std/http/server.ts"
import { createClient } from "jsr:@supabase/supabase-js"

serve(async (req) => {
  // 1. Récupérer les headers de signature
  const signature = req.headers.get("Signature")
  const signatureInput = req.headers.get("Signature-Input")
  const contentDigest = req.headers.get("Content-Digest")
  const signatureDate = req.headers.get("Signature-Date")

  const bodyText = await req.text()

  // 2. Vérifier Content-Digest (SHA-256/SHA-512 du body)
  // 3. Reconstruire la signature base selon Signature-Input
  // 4. Vérifier avec la clé publique pawaPay (stockée en env PAWAPAY_PUBLIC_KEY)
  const verified = await verifyRFC9421(signature, signatureInput, contentDigest, bodyText, signatureDate)
  if (!verified) return new Response("Invalid signature", { status: 401 })

  // 5. Parser payload
  const payload = JSON.parse(bodyText)
  // payload: { depositId, status: 'COMPLETED'|'FAILED', ... }

  // 6. IDEMPOTENCE : n'update que si status != completed
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")! // service_role pour bypass RLS
  )

  const { error } = await supabase
    .from("payments")
    .update({
      status: payload.status === "COMPLETED" ? "completed" : "failed",
      paid_at: payload.status === "COMPLETED" ? new Date().toISOString() : null,
      provider_payload: payload,
    })
    .eq("provider_transaction_id", payload.depositId)
    .neq("status", "completed") // idempotence

  if (error) return new Response("DB error", { status: 500 })

  // 7. Si COMPLETED → confirmer l'appointment (trigger DB peut aussi le faire)
  // 8. HTTP 200 obligatoire sinon pawaPay retry pendant 15 minutes
  return new Response("OK", { status: 200 })
})
```

**Warning signs :** Vérification échoue systématiquement en sandbox. Logguer header par header en dev.

---

### P3 — XAF : DB en INTEGER, API pawaPay en string décimal

**What goes wrong :** On stocke `price = 9900` (INTEGER FCFA). On envoie `amount: "9900"` à pawaPay. pawaPay attend une **string décimale**. Vérifier le comportement exact en sandbox car la doc mentionne que "not all providers support decimals" et AIRTEL_GAB est listé avec "Decimal Support: 2 places".

**How to avoid :**

```typescript
// lib/pawapay/format.ts
export function formatXAFForPawapay(integerFcfa: number): string {
  // pawaPay accepte une string ; XAF zero-decimal en pratique
  // Vérifier en sandbox avec test number COMPLETED (24174345678)
  return String(integerFcfa) // → "9900" (pas "9900.00")
}
```

**Action de plan :** Tester le premier deposit en sandbox avec les 4 formats (`"9900"`, `"9900.00"`, `"9900.0"`, `99.00`) et documenter le format accepté. Les test numbers AIRTEL_GAB sont : `24174345678` (COMPLETED), `24174345048` (FAILED insufficient_balance), `24174345068` (FAILED unspecified), `24174345128` (SUBMITTED indéfini).

---

### P4 — RLS fuite cross-tenant sur migration initiale

**What goes wrong :** Prisma migrate crée les tables mais n'active PAS RLS. Par défaut, tout user voit tout. CVE-2025-48757 a exposé 170+ apps.

**How to avoid :**
```sql
-- Dans chaque migration qui crée une table tenant-scoped :
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
ALTER TABLE appointments FORCE ROW LEVEL SECURITY; -- même pour le propriétaire de la table

CREATE POLICY "salon_isolation_read" ON appointments
  FOR SELECT TO authenticated
  USING (salon_id = (auth.jwt() ->> 'salon_id')::uuid);

CREATE POLICY "salon_isolation_write" ON appointments
  FOR ALL TO authenticated
  USING (salon_id = (auth.jwt() ->> 'salon_id')::uuid)
  WITH CHECK (salon_id = (auth.jwt() ->> 'salon_id')::uuid);

-- Anon (clients non authentifiés sur page publique) : lecture publiée seulement
CREATE POLICY "public_can_read_published" ON salons
  FOR SELECT TO anon
  USING (is_active = true AND is_published = true);
```

**CRITICAL :** Un test automatisé (voir Validation Architecture) doit vérifier qu'un user du salon A ne peut JAMAIS lire les RDV du salon B. Faire ça AVANT d'écrire du code feature.

---

### P5 — Hook JWT : user sans salon_id au signup

**What goes wrong :** Un pro s'inscrit → OTP vérifié → JWT émis → hook cherche `salon_memberships` → aucune ligne → `salon_id` null dans le JWT → toutes les queries RLS retournent vide.

**How to avoid :**

1. Le hook ne plante PAS si pas de membership — il laisse `salon_id` absent.
2. Le middleware Next.js check `claims.salon_id` :
   - Présent → vers `/agenda`
   - Absent → vers `/onboarding`
3. L'onboarding insère dans `salons` puis dans `salon_memberships` via une **Server Action utilisant le service_role** (ou une RPC SECURITY DEFINER qui check `auth.uid() = owner_id`).
4. Après création du membership, **forcer un refresh du JWT** : `supabase.auth.refreshSession()`. Sinon le JWT en cours continue sans `salon_id` jusqu'à expiration (~1h).

**Warning signs :** Le user voit un agenda vide après onboarding. Logs montrent RLS filtre tout.

---

### P6 — Realtime UPDATE sans REPLICA IDENTITY FULL

**What goes wrong :** Sur un UPDATE, Realtime envoie uniquement les colonnes modifiées + la PK. Si le client veut reconciler avec son optimistic UI et a besoin de l'état complet, il n'a que des champs partiels.

**How to avoid :**
```sql
ALTER TABLE appointments REPLICA IDENTITY FULL;
```

Attention : augmente le WAL. OK pour Phase 1 (volumes faibles), revoir en Phase 3+.

---

### P7 — pg_cron nécessite extension + invocation Edge Function HTTP

**What goes wrong :** On veut que pg_cron toutes les 60s traite `notification_queue`, mais pg_cron ne peut pas appeler Twilio directement (Deno/Node requis).

**How to avoid :**

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_net; -- pour faire HTTP depuis SQL

-- Schedule toutes les 60s : invoquer l'Edge Function
SELECT cron.schedule(
  'process-notification-queue',
  '* * * * *', -- chaque minute (pas moins possible en free tier)
  $$
  SELECT net.http_post(
    url := 'https://<project-ref>.supabase.co/functions/v1/process-notification-queue',
    headers := jsonb_build_object(
      'Authorization', 'Bearer ' || current_setting('app.edge_function_secret'),
      'Content-Type', 'application/json'
    )
  );
  $$
);

-- Pour le rappel 24h : job séparé toutes les 15 min
SELECT cron.schedule(
  'enqueue-24h-reminders',
  '*/15 * * * *',
  $$
  INSERT INTO notification_queue (type, appointment_id, salon_id, recipient_phone, template, payload)
  SELECT 'sms', a.id, a.salon_id, c.phone, 'booking_reminder_24h',
         jsonb_build_object('salon_name', s.name, 'start_time', a.start_time)
  FROM appointments a
  JOIN salons s ON s.id = a.salon_id
  JOIN customers c ON c.id = a.customer_id -- ou c.phone
  WHERE a.status = 'confirmed'
    AND a.start_time BETWEEN now() + interval '23 hours 45 minutes'
                          AND now() + interval '24 hours 15 minutes'
    AND NOT EXISTS (
      SELECT 1 FROM notification_queue nq
      WHERE nq.appointment_id = a.id AND nq.template = 'booking_reminder_24h'
    );
  $$
);
```

**Gotcha :** le plus petit interval pg_cron = 1 minute. Pour du quasi-realtime, rester à 1 min.

---

### P8 — Next.js 15 : Dynamic APIs async

**What goes wrong :** Code copié-collé d'un projet Next.js 14 utilise `cookies()`, `headers()`, `params.id` de façon synchrone. Build OK mais runtime error ou type errors.

**How to avoid :**
```typescript
// Next.js 15 : tout est async
const cookieStore = await cookies()
const h = await headers()
const { id } = await params // dans les pages
const query = await searchParams
```

---

### P9 — Page publique `/[slug]` et anon key : attention à la fuite de données

**What goes wrong :** Le client non-auth utilise l'anon key. Si une policy oublie de restreindre `salons.is_published = true`, on expose les brouillons. Pire : si `services` n'a pas de policy anon, la page plante en prod (RLS bloque).

**How to avoid :** Policies explicites pour `TO anon` :
```sql
CREATE POLICY "public_read_salons" ON salons
  FOR SELECT TO anon
  USING (is_active = true AND is_published = true);

CREATE POLICY "public_read_services" ON services
  FOR SELECT TO anon
  USING (
    is_active = true AND
    salon_id IN (SELECT id FROM salons WHERE is_published = true)
  );

CREATE POLICY "public_read_staff" ON staff_members
  FOR SELECT TO anon
  USING (
    is_active = true AND
    salon_id IN (SELECT id FROM salons WHERE is_published = true)
  );
```

`get_available_slots()` doit être `SECURITY DEFINER` pour que l'anon puisse l'appeler sans tout ouvrir.

---

### P10 — Explosion coût SMS

**What goes wrong :** 20 RDV/jour × 3 SMS (confirm client + confirm pro + rappel 24h) × 0.20$ ≈ 360$/mois. Plan Starter = ~60$/mois. Perte nette par client.

**How to avoid en Phase 1 :**
- Pro : un seul SMS par jour récapitulatif (pas par RDV) → `notification_queue` digest
- Ou : pro notifié via Realtime in-app uniquement (pas SMS) — suffit si le pro a l'agenda ouvert
- Client : confirmation immédiate + rappel 24h (2 SMS par RDV, incompressible)
- **Budget Phase 1 :** inclure 50 SMS/mois dans le plan, facturation à l'usage au-delà
- Phase 2 : migration WhatsApp obligatoire — soumettre demande Meta dès le début de Phase 1

---

## Code Examples

### Schema Prisma (extract) — migrations-only

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DIRECT_URL") // Connexion directe pour migrations
}

model Salon {
  id          String   @id @default(uuid()) @db.Uuid
  ownerId     String   @map("owner_id") @db.Uuid
  name        String
  slug        String   @unique
  phone       String?
  address     String?
  city        String   @default("Libreville")
  country     String   @default("GA")
  currency    String   @default("XAF")
  timezone    String   @default("Africa/Libreville")
  isActive    Boolean  @default(true) @map("is_active")
  isPublished Boolean  @default(false) @map("is_published")
  createdAt   DateTime @default(now()) @map("created_at") @db.Timestamptz(6)

  services      Service[]
  staff         StaffMember[]
  appointments  Appointment[]
  memberships   SalonMembership[]

  @@map("salons")
}

model Service {
  id              String   @id @default(uuid()) @db.Uuid
  salonId         String   @map("salon_id") @db.Uuid
  name            String
  durationMinutes Int      @map("duration_minutes")
  price           Int      // FCFA INTEGER
  isActive        Boolean  @default(true) @map("is_active")

  salon Salon @relation(fields: [salonId], references: [id], onDelete: Cascade)

  @@index([salonId])
  @@map("services")
}

// ... (voir ARCHITECTURE.md pour le schéma complet)
```

**Post-migration SQL manuel** (Prisma ne génère pas RLS/constraints/extensions) :
```sql
-- supabase/migrations/001_post_prisma.sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_net;

-- Activer RLS + policies sur toutes les tables
ALTER TABLE salons ENABLE ROW LEVEL SECURITY;
ALTER TABLE services ENABLE ROW LEVEL SECURITY;
-- ... etc

-- EXCLUDE constraint anti double-booking
ALTER TABLE appointments
ADD CONSTRAINT no_double_booking
EXCLUDE USING gist (
  staff_id WITH =,
  tstzrange(start_time, end_time, '[)') WITH &&
) WHERE (status IN ('pending_payment', 'confirmed'));

-- REPLICA IDENTITY pour Realtime
ALTER TABLE appointments REPLICA IDENTITY FULL;
```

### Fonction SQL `create_booking` (RPC atomique)

```sql
CREATE OR REPLACE FUNCTION public.create_booking(
  p_salon_id uuid,
  p_service_id uuid,
  p_staff_id uuid,
  p_start_time timestamptz,
  p_client_phone text,
  p_client_name text
) RETURNS TABLE (id uuid, requires_payment boolean)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_duration int;
  v_end_time timestamptz;
  v_appointment_id uuid;
  v_customer_id uuid;
  v_price int;
  v_salon_published boolean;
BEGIN
  -- 1. Vérifier salon publié (page publique)
  SELECT is_published INTO v_salon_published
  FROM salons WHERE id = p_salon_id;
  IF NOT v_salon_published THEN
    RAISE EXCEPTION 'Salon not available' USING ERRCODE = 'P0001';
  END IF;

  -- 2. Charger durée + prix service
  SELECT duration_minutes, price INTO v_duration, v_price
  FROM services WHERE id = p_service_id AND salon_id = p_salon_id AND is_active = true;
  IF v_duration IS NULL THEN
    RAISE EXCEPTION 'Service not found' USING ERRCODE = 'P0002';
  END IF;

  v_end_time := p_start_time + (v_duration || ' minutes')::interval;

  -- 3. Upsert customer par phone
  INSERT INTO customers (salon_id, phone, name)
  VALUES (p_salon_id, p_client_phone, p_client_name)
  ON CONFLICT (salon_id, phone) DO UPDATE SET name = EXCLUDED.name
  RETURNING id INTO v_customer_id;

  -- 4. INSERT appointment — la contrainte EXCLUDE lève 23P01 si conflit
  INSERT INTO appointments (
    salon_id, customer_id, staff_id, start_time, end_time,
    type, status, total_price
  )
  VALUES (
    p_salon_id, v_customer_id, p_staff_id, p_start_time, v_end_time,
    'booking', 'pending_payment', v_price
  )
  RETURNING appointments.id INTO v_appointment_id;

  -- 5. Link service
  INSERT INTO appointment_services (appointment_id, service_id, price_at_booking)
  VALUES (v_appointment_id, p_service_id, v_price);

  -- 6. Enqueue notification pro (trigger fera aussi l'affaire)
  INSERT INTO notification_queue (type, appointment_id, salon_id, template, payload, recipient_phone)
  SELECT 'sms', v_appointment_id, p_salon_id, 'booking_notification_pro',
         jsonb_build_object('customer_name', p_client_name, 'start_time', p_start_time),
         s.phone
  FROM salons s WHERE s.id = p_salon_id;

  RETURN QUERY SELECT v_appointment_id, (v_price > 0);
END;
$$;

GRANT EXECUTE ON FUNCTION public.create_booking TO anon, authenticated;
```

### Cleanup pending_payment job

```sql
SELECT cron.schedule(
  'expire-pending-payments',
  '* * * * *', -- chaque minute
  $$
  UPDATE appointments
  SET status = 'cancelled',
      cancellation_reason = 'payment_timeout',
      cancelled_at = now()
  WHERE status = 'pending_payment'
    AND created_at < now() - interval '10 minutes';
  $$
);
```

### Worker Edge Function `process-notification-queue`

```typescript
// supabase/functions/process-notification-queue/index.ts
import { serve } from "https://deno.land/std/http/server.ts"
import { createClient } from "jsr:@supabase/supabase-js"
import twilio from "npm:twilio"

const MAX_ATTEMPTS = 3
const BATCH_SIZE = 20

serve(async (_req) => {
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  )
  const tw = twilio(Deno.env.get("TWILIO_SID")!, Deno.env.get("TWILIO_TOKEN")!)
  const from = Deno.env.get("TWILIO_FROM")!

  // Lock & fetch (SELECT ... FOR UPDATE SKIP LOCKED pour concurrence)
  const { data: items } = await supabase.rpc("claim_notification_batch", {
    p_limit: BATCH_SIZE,
  })

  for (const item of items ?? []) {
    try {
      const body = renderTemplate(item.template, item.payload)
      await tw.messages.create({ from, to: item.recipient_phone, body })
      await supabase.from("notification_queue")
        .update({ status: "sent", processed_at: new Date().toISOString() })
        .eq("id", item.id)
    } catch (err) {
      const attempts = (item.attempts ?? 0) + 1
      await supabase.from("notification_queue")
        .update({
          status: attempts >= MAX_ATTEMPTS ? "failed" : "pending",
          attempts,
          last_error: String(err),
        })
        .eq("id", item.id)
    }
  }

  return new Response(JSON.stringify({ processed: items?.length ?? 0 }), { status: 200 })
})
```

Fonction SQL `claim_notification_batch` (verrou concurrent-safe) :
```sql
CREATE OR REPLACE FUNCTION public.claim_notification_batch(p_limit int)
RETURNS SETOF notification_queue
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  UPDATE notification_queue nq
  SET status = 'processing', processed_at = null
  WHERE nq.id IN (
    SELECT id FROM notification_queue
    WHERE status = 'pending' AND attempts < 3
    ORDER BY created_at
    LIMIT p_limit
    FOR UPDATE SKIP LOCKED
  )
  RETURNING nq.*;
END;
$$;
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@supabase/auth-helpers-nextjs` | `@supabase/ssr` (`createServerClient` + `createBrowserClient`) | Supabase SSR release 2024 | auth-helpers déprécié, ne plus utiliser |
| Next.js 14 sync dynamic APIs | Next.js 15 async `cookies()`, `headers()`, `params` | Next.js 15 (Oct 2024) | Refactor obligatoire sur tous les RSC qui lisent ces APIs |
| pawaPay v1 HMAC callbacks | pawaPay v2 RFC-9421 signatures asymétriques | pawaPay v2 API | Code de vérification totalement différent |
| Prisma runtime queries sur Supabase | Prisma migrations-only, Supabase JS runtime | Community consensus 2023-2024 | RLS fonctionne automatiquement |
| `generate_series` côté client pour slots | `get_available_slots()` PL/pgSQL | Booking SaaS standard | Atomicité + perf |
| Single webhook endpoint pour tous providers | Un Edge Function par provider | Observabilité | Logs/debug isolés |

**Deprecated/outdated :**
- `@supabase/auth-helpers-nextjs` → `@supabase/ssr`
- Next.js 14 → Next.js 15 (toutes les dynamic APIs async)
- Africa's Talking pour Gabon → Twilio
- CinetPay/FedaPay pour Gabon → pawaPay
- Moov Money Gabon via pawaPay → **non disponible**, retirer du scope

---

## Open Questions

1. **Format exact du champ `amount` pour AIRTEL_GAB via pawaPay**
   - What we know : doc dit `amount` est une string, regex `^([0]|([1-9][0-9]{0,17}))([.][0-9]{0,3}[1-9])?$`, "Decimal Support: 2 places" pour AIRTEL_GAB.
   - What's unclear : faut-il envoyer `"9900"` ou `"9900.00"` ? La doc dit "not all providers support decimals".
   - Recommendation : **tester en premier dans le plan 01-06 avec test number `24174345678` les deux formats** et locker le format qui retourne ACCEPTED.

2. **Hook JWT et refresh après onboarding**
   - What we know : `refreshSession()` existe.
   - What's unclear : le hook est-il ré-exécuté sur refresh, ou seulement sur signin ?
   - Recommendation : tester explicitement dans le plan 01-03 — après l'INSERT du membership, appeler `supabase.auth.refreshSession()` et vérifier que `jwt.salon_id` est peuplé.

3. **Annulation via lien SMS sans compte**
   - What we know : BOOKING-07 exige annulation via lien.
   - What's unclear : comment sécuriser sans session ? Token signé HMAC dans l'URL ?
   - Recommendation : colonne `cancellation_token text` (UUID v4) sur `appointments`, URL `/[slug]/manage/{token}`, Server Action qui lookup par token + check `start_time > now() + cancellation_window`. Invalider le token après usage.

4. **Latence Supabase London → Libreville**
   - What we know : ~120-150ms estimé.
   - What's unclear : impact UX réel sur le booking flow.
   - Recommendation : optimistic UI sur agenda (Zustand), Server Components + streaming pour la page publique.

5. **pawaPay KYB timeline et production go-live**
   - What we know : KYB requis (2-4 semaines), sandbox peut commencer sans.
   - What's unclear : quand basculer en prod.
   - Recommendation : tout Phase 1 en sandbox, go-live prod lors du lancement pilote (après validation bout-en-bout).

6. **Cleanup appointments expirés vs REPLICA IDENTITY WAL**
   - What we know : REPLICA IDENTITY FULL augmente le WAL.
   - What's unclear : impact sur le plan Supabase (gratuit/pro) en Phase 1.
   - Recommendation : activer sur `appointments` uniquement (pas toutes les tables), monitorer Supabase metrics.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | **Vitest 2.x** (unit + integration, aligné Next.js 15) + **Playwright 1.49+** (e2e) |
| Config file | `vitest.config.ts` (à créer Wave 0) + `playwright.config.ts` (à créer Wave 0) |
| Quick run command | `pnpm vitest run --reporter=dot` |
| Full suite command | `pnpm vitest run && pnpm playwright test` |
| SQL tests | **pgTAP** extension sur une DB Supabase de test (branch ou local) — pour tester les functions + RLS |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| AUTH-01 | Pro signup OTP | integration | `vitest tests/auth/signup.test.ts` | ❌ Wave 0 |
| AUTH-02 | Client signup OTP | integration | `vitest tests/auth/client-signup.test.ts` | ❌ Wave 0 |
| AUTH-03 | Login OTP flow | e2e | `playwright test tests/e2e/login.spec.ts` | ❌ Wave 0 |
| AUTH-04 | Session persistance | e2e | `playwright test tests/e2e/session.spec.ts` | ❌ Wave 0 |
| AUTH-05 | **RLS isolation cross-tenant** | integration | `vitest tests/rls/cross-tenant.test.ts` | ❌ Wave 0 (CRITIQUE) |
| ONBOARD-01..05 | Wizard salon complet | e2e | `playwright test tests/e2e/onboarding.spec.ts` | ❌ Wave 0 |
| BOOKING-01 | Page publique accessible anon | integration | `vitest tests/booking/public-access.test.ts` | ❌ Wave 0 |
| BOOKING-02 | Slots query retourne résultats corrects | unit (SQL) | `pgtap tests/sql/get_available_slots.sql` | ❌ Wave 0 |
| BOOKING-03 | Slots temps réel | integration | `vitest tests/booking/slots.test.ts` | ❌ Wave 0 |
| BOOKING-04 | **Anti double-booking** | integration | `vitest tests/booking/concurrent-booking.test.ts` (2 INSERT simultanés → 1 réussit, 1 erreur 23P01) | ❌ Wave 0 (CRITIQUE) |
| BOOKING-05 | SMS confirmation client | integration | `vitest tests/notifications/queue.test.ts` (check insertion queue) | ❌ Wave 0 |
| BOOKING-06 | SMS notification pro | integration | idem | ❌ Wave 0 |
| BOOKING-07 | Cancel via token SMS | integration | `vitest tests/booking/cancel.test.ts` | ❌ Wave 0 |
| BOOKING-08 | **Slot lock 10 min expiration** | integration | `vitest tests/booking/pending-payment-expire.test.ts` (time-travel) | ❌ Wave 0 |
| WALKIN-01..04 | Walk-in flow | e2e | `playwright test tests/e2e/walkin.spec.ts` | ❌ Wave 0 |
| AGENDA-01..04 | CRUD agenda | e2e | `playwright test tests/e2e/agenda.spec.ts` | ❌ Wave 0 |
| AGENDA-05 | Realtime sync | e2e | `playwright test tests/e2e/agenda-realtime.spec.ts` (2 onglets) | ❌ Wave 0 |
| AGENDA-06 | Responsive mobile | e2e | `playwright test --project=mobile tests/e2e/agenda.spec.ts` | ❌ Wave 0 |
| PAYMENT-01 | Airtel deposit sandbox | integration | `vitest tests/payment/pawapay-deposit.test.ts` (test number COMPLETED) | ❌ Wave 0 |
| PAYMENT-02 | Cash payment | integration | `vitest tests/payment/cash.test.ts` | ❌ Wave 0 |
| PAYMENT-03 | **Webhook idempotency** | integration | `vitest tests/payment/webhook-idempotent.test.ts` (deux webhooks identiques → un seul update) | ❌ Wave 0 (CRITIQUE) |
| PAYMENT-04 | Edge Function isolation | integration | `deno test supabase/functions/pawapay-webhook/test.ts` | ❌ Wave 0 |
| PAYMENT-05 | INTEGER FCFA | unit | `vitest tests/format/xaf.test.ts` | ❌ Wave 0 |
| PAYMENT-06 | Liste historique | e2e | `playwright test tests/e2e/payments-history.spec.ts` | ❌ Wave 0 |
| NOTIF-01..03 | Queue SMS templates | integration | `vitest tests/notifications/templates.test.ts` | ❌ Wave 0 |
| NOTIF-04 | Queue async pattern | integration | `vitest tests/notifications/worker.test.ts` | ❌ Wave 0 |
| NOTIF-05 | **Retry max 3** | integration | `vitest tests/notifications/retry.test.ts` (mock Twilio fail) | ❌ Wave 0 |
| DASHBOARD-01..04 | Aggregates | integration | `vitest tests/dashboard/metrics.test.ts` | ❌ Wave 0 |

### Tests critiques (blockers pour le merge)

1. **`tests/rls/cross-tenant.test.ts`** — deux clients Supabase avec des JWT de salons différents. Essaie de lire les appointments de l'autre salon → doit retourner tableau vide. **Doit passer avant ANY feature code merge.**

2. **`tests/booking/concurrent-booking.test.ts`** — `Promise.all([createBooking(...), createBooking(...)])` avec le même staff/slot. Vérifier : exactement 1 succès, exactement 1 erreur de code PG `23P01`.

3. **`tests/payment/webhook-idempotent.test.ts`** — rejouer le même webhook pawaPay 3 fois. Vérifier : `paid_at` ne bouge pas après le premier succès, aucun duplicate SMS, aucune double-confirmation.

4. **`tests/sql/get_available_slots.sql`** (pgTAP) — fixtures avec horaires + RDV existants + pauses + exceptions. Assertions sur les slots retournés.

5. **`tests/booking/pending-payment-expire.test.ts`** — INSERT appointment `pending_payment` il y a 11 min → run cleanup job → status devient `cancelled`, le slot redevient disponible.

### Sampling Rate

- **Per task commit :** `pnpm vitest run --changed` (tests liés aux fichiers modifiés)
- **Per wave merge :** `pnpm vitest run && pnpm playwright test --project=chromium`
- **Phase gate :** Full suite verte (vitest + playwright all projects + pgTAP) avant `/gsd:verify-work`

### Wave 0 Gaps

Tout est à créer — aucune infra de test n'existe pour l'instant.

- [ ] `vitest.config.ts` + `tests/setup.ts` (mock env, Supabase test client)
- [ ] `playwright.config.ts` + projets `chromium` + `mobile` (Pixel 5)
- [ ] `tests/helpers/supabase.ts` — factory pour créer un client avec JWT custom (pour tester RLS)
- [ ] `tests/helpers/fixtures.ts` — seed salons, services, staff, appointments
- [ ] `tests/helpers/pawapay-mock.ts` — mock endpoint deposit + fake webhook signed
- [ ] `tests/helpers/twilio-mock.ts` — intercepte les appels Twilio en test
- [ ] Framework install : `pnpm add -D vitest @vitest/ui playwright @playwright/test tsx`
- [ ] pgTAP setup : `CREATE EXTENSION pgtap;` sur branche de test + harness
- [ ] CI GitHub Actions : job `test` qui lance vitest + playwright + pgTAP

---

## Dependency Order (chemin critique Phase 1)

```
Plan 01-01 (Schema DB + RLS + Hook JWT)
    │  ← BLOQUANT pour TOUT le reste
    ▼
Plan 01-02 (Auth OTP Supabase + Twilio)     ◄─── peut paralléliser avec 01-03 une fois 01-01 fini
    │
    ├──► Plan 01-03 (Onboarding pro)
    │       │  ← produit un salon + memberships + services + horaires
    │       ▼
    │    Plan 01-04 (get_available_slots + btree_gist + pending_payment cleanup)
    │       │  ← fondation technique du booking
    │       ▼
    │    Plan 01-05 (Flux booking client public)
    │       │
    │       ▼
    │    Plan 01-06 (Paiements pawaPay)         ◄─── NÉCESSITE KYB pawaPay terminé
    │       │
    │       ▼
    │    Plan 01-07 (Walk-ins + cash)           ◄─── peut démarrer après 01-04
    │       │
    │       ▼
    │    Plan 01-08 (Notifications SMS queue + Twilio worker)
    │       │  ← peut démarrer après 01-04, templates après 01-05/06
    │       ▼
    │    Plan 01-09 (Agenda pro + Realtime)     ◄─── peut démarrer après 01-04
    │       │
    │       ▼
    └──► Plan 01-10 (Dashboard CA)               ◄─── peut démarrer après 01-06 + 01-07
```

**Parallélisation possible** (après 01-01 et 01-02) :
- Wave A : 01-03 (onboarding) + 01-04 (disponibilités)
- Wave B : 01-05 (booking) + 01-07 (walk-ins) + 01-09 (agenda) + 01-08 (notifications SMS scaffold)
- Wave C : 01-06 (pawaPay) + 01-10 (dashboard)

**Blockers externes :**
- KYB pawaPay (2-4 semaines) doit démarrer **dès Plan 01-01** — sinon 01-06 bloque
- Twilio account + numéro Sender (1-2 jours) avant 01-02

---

## Decisions for the Planner

1. **Cron worker pour notification_queue : Recommandation pg_cron + pg_net → Edge Function**
   - Alternatives : Vercel Cron (payant sur Pro), Edge Function scheduled (Supabase supporte désormais)
   - Recommandé : `pg_cron` schedule 1 min → `pg_net.http_post` → Edge Function `process-notification-queue`. Zéro infra additionnelle, observable via Supabase logs.

2. **Page publique authentification anon vs session invitée**
   - Recommandation : **anon key + RLS policies `TO anon`**. Le client entre son téléphone au moment de la réservation (pas de signup OTP pour réserver en Phase 1). L'OTP client devient obligatoire seulement pour gérer son RDV après coup (Phase 2, CRM-03).
   - Pattern : `customers` table liée à `appointments` via `customer_id`, pas via `auth.users`. Le lien SMS de gestion est basé sur `cancellation_token`.

3. **Connexion DB : Prisma DIRECT_URL vs pooled**
   - Recommandation : Prisma utilise **`DIRECT_URL`** (direct connection, port 5432) pour les migrations. Le Supabase JS client utilise le REST API (PostgREST), pas de connexion directe. Pas besoin de pooler côté Prisma en Phase 1.

4. **Un seul salon par user (Phase 1) vs multi-memberships (Phase 3+)**
   - Recommandation : Schema préparé pour multi (`salon_memberships` table), mais le hook JWT prend LIMIT 1 en Phase 1. Le refactor Phase 3 devra ajouter un selector de salon actif.

5. **Tests SQL : pgTAP vs Vitest + client Supabase**
   - Recommandation : **pgTAP** pour `get_available_slots` et RLS policies (tests SQL natifs, rapides). **Vitest** pour le reste.

6. **Structure wizard onboarding**
   - Recommandation : **4 étapes linéaires** avec progress bar, pas de back-end save avant la fin (sauvegarde locale via `zustand` persistant). Final submit = tout en une seule transaction.

7. **Template Twilio SMS — rendering**
   - Recommandation : rendering côté worker Edge Function (TypeScript template literals), pas côté DB. Les templates en dur dans le code du worker, payload JSON dans la queue = les variables uniquement.

---

## Sources

### Primary (HIGH confidence)
- **Supabase Auth Hooks doc** — https://supabase.com/docs/guides/auth/auth-hooks (custom_access_token_hook signature, GRANT requirements)
- **Supabase Realtime Postgres Changes** — https://supabase.com/docs/guides/realtime/postgres-changes (RLS behavior, filter syntax)
- **Supabase Realtime Authorization** — https://supabase.com/docs/guides/realtime/authorization (postgres_changes ≠ realtime.messages RLS)
- **PostgreSQL btree_gist doc** — https://www.postgresql.org/docs/current/btree-gist.html (GIST operator classes pour UUID/tstzrange)
- **PostgreSQL CREATE TABLE** — https://www.postgresql.org/docs/current/sql-createtable.html (EXCLUDE ... WHERE partial syntax confirmé)
- **pawaPay v2 API Reference** — https://docs.pawapay.io/v2/api-reference (POST /v2/deposits, base URLs sandbox/prod)
- **pawaPay Signatures (RFC-9421)** — https://docs.pawapay.io/v2/docs/signatures (RSA/ECDSA asymétrique, pas HMAC)
- **pawaPay Providers (Gabon)** — https://docs.pawapay.io/v2/docs/providers (AIRTEL_GAB confirmé, Moov NON supporté)
- **pawaPay test numbers** — https://docs.pawapay.io/v2/docs/test_numbers (numéros sandbox AIRTEL_GAB)
- **pawaPay signatures Node sample** — https://github.com/pawaPay/signatures-node-example

### Secondary (MEDIUM confidence — recherche amont `.planning/research/`)
- `ARCHITECTURE.md` — schema DB, patterns RLS, get_available_slots template
- `STACK.md` — versions validées, corrections Africa's Talking/CinetPay/Stripe
- `FEATURES.md` — priorisation Phase 1, graphe de dépendances
- `PITFALLS.md` — double-booking, RLS, Prisma, pending_payment, SMS cost, timezone Gabon

### Tertiary (LOW confidence — à valider sandbox)
- Format exact `amount` string pour AIRTEL_GAB (`"9900"` vs `"9900.00"`) — **à tester en Plan 01-06**
- Comportement refresh JWT après hook update — **à tester en Plan 01-03**
- Latence réelle London → Libreville — **à mesurer en conditions réelles**

---

## Metadata

**Confidence breakdown :**
- Standard stack : **HIGH** — versions vérifiées, patterns documentés par Supabase/Next.js officiel
- Architecture (RLS + JWT hook + btree_gist + notification_queue) : **HIGH** — confirmé par doc officielle Supabase + PostgreSQL
- pawaPay integration : **MEDIUM-HIGH** — endpoints v2 confirmés, format amount à valider en sandbox
- Twilio + Supabase Phone Auth integration : **HIGH** — chemin standard documenté
- Realtime + RLS pour l'agenda : **HIGH** — pattern documenté
- Validation architecture (tests) : **MEDIUM** — outillage à mettre en place Wave 0

**Research date :** 2026-04-11
**Valid until :** 2026-05-11 (30 jours — stack stable ; revalider pawaPay API si >1 mois)

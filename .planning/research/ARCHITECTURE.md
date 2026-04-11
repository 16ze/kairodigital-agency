# Architecture Research: Bookly

**Project:** Bookly — Multi-tenant SaaS booking beauté, Gabon
**Date:** 2026-04-10
**Confidence:** HIGH

---

## ⚠️ Conflits à résoudre vs PROJECT.md initial

| Sujet | Stack Research | Architecture Research | Décision |
|---|---|---|---|
| Stripe Gabon | ❌ Non disponible, utiliser via Kairo Digital France | ❌ Non disponible directement | **Utiliser via entité française Kairo Digital** |
| Paiement mobile money | **pawaPay** (Airtel Gabon confirmé), CinetPay ❌ Gabon | CinetPay suggéré | **→ pawaPay** (Stack Research plus précis sur couverture Gabon) |
| Prisma + Supabase | Prisma dans la stack | Tension Prisma/RLS | **Prisma pour migrations uniquement, Supabase client pour toutes les queries RLS** |

---

## Architecture globale

Bookly suit une architecture **shared-database, shared-schema multi-tenant** avec PostgreSQL Row-Level Security (RLS) comme mécanisme d'isolation. Tous les salons partagent les mêmes tables, différenciés par `salon_id` sur chaque ligne. Le JWT Supabase Auth embarque `salon_id` comme custom claim pour l'évaluation automatique des policies.

### Diagramme haut niveau

```
                        ┌─────────────────────────────┐
                        │         Vercel Edge          │
                        │   (Next.js 15 App Router)    │
                        └─────────────────────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
   ┌──────────┴──────────┐                      ┌────────────┴────────────┐
   │  Pro Dashboard      │                      │  Client Booking App     │
   │  (authentifié)      │                      │  (public-facing)        │
   │  Server Components  │                      │  Server Components      │
   │  + Server Actions   │                      │  + Server Actions       │
   └──────────┬──────────┘                      └────────────┬────────────┘
              │                                               │
              └───────────────────┬───────────────────────────┘
                                  │
                   ┌──────────────▼──────────────┐
                   │      Supabase Backend        │
                   │  PostgreSQL + RLS            │
                   │  Supabase Auth (OTP phone)   │
                   │  Supabase Realtime           │
                   │  Edge Functions (Deno)       │
                   │  pg_cron                     │
                   └──────────────┬──────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
   ┌──────▼──────┐       ┌────────▼──────┐       ┌───────▼──────┐
   │ Paiements   │       │ SMS           │       │ Phase 4      │
   │ pawaPay     │       │ Twilio        │       │ Expo RN App  │
   │ Stripe (FR) │       │ WhatsApp API  │       │              │
   └─────────────┘       └───────────────┘       └──────────────┘
```

---

## Limites des composants

| Composant | Responsabilité | Technologies |
|---|---|---|
| **Pro Dashboard** | Config salon, agenda, CRM, POS, analytics | Next.js 15 Server Components + Server Actions |
| **Client Booking App** | Recherche salons, réservation, gestion RDV | Next.js 15 Server Components + Server Actions |
| **Supabase Auth** | Auth pros et clients, JWT avec salon_id claims | Supabase Phone OTP + email |
| **PostgreSQL + RLS** | Toutes les données, isolation tenant, calcul slots | PostgreSQL 15+, RLS policies |
| **Supabase Realtime** | Live agenda updates, notifications réservation | Supabase Realtime (Postgres Changes) |
| **Edge Functions** | Webhooks paiement, processing async | Supabase Edge Functions (Deno) |
| **pg_cron** | Rappels SMS programmés, cleanup | pg_cron extension |
| **Payment Abstraction** | Interface unifiée pawaPay + Stripe | Strategy pattern, adapter par provider |
| **SMS Service** | Confirmations, rappels via notification_queue | Twilio Node.js SDK |
| **Expo Mobile** (Phase 4) | App native client, push notifications | Expo React Native SDK 53 |

---

## Multi-tenancy — Shared Schema + RLS

### Pourquoi cette approche

- Bookly cible des TPE beauty (1-5 employés) → centaines à milliers de petits salons
- Schema-per-tenant = complexité de migration injustifiée
- RLS = overhead 1-5% seulement + isolation niveau base de données
- Intégration native avec JWT custom claims Supabase Auth

### Identification du tenant

```
JWT (Supabase Auth) → custom claim: salon_id
                    → custom claim: role (owner | staff | client)
```

Chaque requête authentifiée porte un JWT avec `salon_id` en custom claim. Les RLS policies référencent `auth.jwt() ->> 'salon_id'` pour filtrer automatiquement.

### Patterns RLS

```sql
-- Pattern 1 : Isolation salon (appointments, services, staff)
CREATE POLICY "Salon isolation" ON appointments
  FOR ALL
  USING (salon_id = (auth.jwt() ->> 'salon_id')::uuid);

-- Pattern 2 : Owner uniquement (settings, billing)
CREATE POLICY "Owner only" ON salon_settings
  FOR ALL
  USING (
    salon_id = (auth.jwt() ->> 'salon_id')::uuid
    AND (auth.jwt() ->> 'role') = 'owner'
  );

-- Pattern 3 : Client voit ses propres RDV (multi-salons)
CREATE POLICY "Client own bookings" ON appointments
  FOR SELECT
  USING (client_id = auth.uid());

-- Pattern 4 : Lecture publique pour booking (sans auth)
CREATE POLICY "Public salon info" ON salons
  FOR SELECT
  USING (is_active = true AND is_published = true);

-- Pattern 5 : Service role pour Edge Functions (bypass RLS)
-- Utiliser service_role key uniquement dans Edge Functions
```

### Règles critiques RLS

1. **Toutes les tables tenant-scoped** doivent avoir `salon_id` indexé en composite
2. **UPDATE policies** doivent être couplées avec SELECT policies
3. **Service role key** uniquement dans Edge Functions — jamais exposé au client
4. **Index composites** sur `(salon_id, date)`, `(salon_id, staff_id, date)` obligatoires

---

## Schéma de base de données

### Relations principales

```
salons
  ├── salon_settings (1:1)
  ├── services (1:N)
  ├── staff_members (1:N)
  │     ├── staff_working_hours (1:N)
  │     ├── staff_breaks (1:N)
  │     └── staff_exceptions (1:N — congés, absences)
  ├── appointments (1:N)
  │     ├── payments (1:1)
  │     └── appointment_services (1:N — multi-prestations)
  ├── customers (1:N — fiche client salon)
  │     └── lié à auth.users via client_user_id
  └── working_hours (1:N — horaires par défaut salon)

auth.users
  ├── profiles (1:1 — metadata, type: pro|client)
  └── salon_memberships (1:N — salons + role)
```

### Tables clés

```sql
-- Salon (racine du tenant)
CREATE TABLE salons (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id    UUID REFERENCES auth.users(id) NOT NULL,
  name        TEXT NOT NULL,
  slug        TEXT UNIQUE NOT NULL,
  phone       TEXT,
  address     TEXT,
  city        TEXT DEFAULT 'Libreville',
  country     TEXT DEFAULT 'GA',
  currency    TEXT DEFAULT 'XAF',
  timezone    TEXT DEFAULT 'Africa/Libreville',
  is_active   BOOLEAN DEFAULT true,
  is_published BOOLEAN DEFAULT false,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- Services
CREATE TABLE services (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  salon_id         UUID REFERENCES salons(id) NOT NULL,
  name             TEXT NOT NULL,
  duration_minutes INTEGER NOT NULL,
  price            INTEGER NOT NULL,  -- FCFA, pas de décimales
  is_active        BOOLEAN DEFAULT true,
  sort_order       INTEGER DEFAULT 0
);

-- Collaborateurs
CREATE TABLE staff_members (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  salon_id   UUID REFERENCES salons(id) NOT NULL,
  user_id    UUID REFERENCES auth.users(id),
  first_name TEXT NOT NULL,
  last_name  TEXT NOT NULL,
  phone      TEXT,
  is_active  BOOLEAN DEFAULT true
);

-- Horaires par collaborateur (par jour de semaine)
CREATE TABLE staff_working_hours (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  staff_id     UUID REFERENCES staff_members(id) NOT NULL,
  salon_id     UUID REFERENCES salons(id) NOT NULL,
  day_of_week  INTEGER NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
  start_time   TIME NOT NULL,
  end_time     TIME NOT NULL,
  is_working   BOOLEAN DEFAULT true
);

-- Pauses (déjeuner, etc.)
CREATE TABLE staff_breaks (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  staff_id    UUID REFERENCES staff_members(id) NOT NULL,
  salon_id    UUID REFERENCES salons(id) NOT NULL,
  day_of_week INTEGER NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
  start_time  TIME NOT NULL,
  end_time    TIME NOT NULL
);

-- Exceptions (congés, absences, horaires spéciaux)
CREATE TABLE staff_exceptions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  staff_id     UUID REFERENCES staff_members(id) NOT NULL,
  salon_id     UUID REFERENCES salons(id) NOT NULL,
  date         DATE NOT NULL,
  is_available BOOLEAN DEFAULT false,
  start_time   TIME,
  end_time     TIME
);

-- Rendez-vous (entité core)
CREATE TABLE appointments (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  salon_id            UUID REFERENCES salons(id) NOT NULL,
  client_id           UUID REFERENCES auth.users(id),
  staff_id            UUID REFERENCES staff_members(id) NOT NULL,
  start_time          TIMESTAMPTZ NOT NULL,
  end_time            TIMESTAMPTZ NOT NULL,
  type                TEXT DEFAULT 'booking'
    CHECK (type IN ('booking', 'walk_in')),  -- walk-ins inclus
  status              TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'confirmed', 'cancelled', 'completed', 'no_show')),
  total_price         INTEGER NOT NULL DEFAULT 0,
  notes               TEXT,
  cancellation_reason TEXT,
  cancelled_at        TIMESTAMPTZ,
  created_at          TIMESTAMPTZ DEFAULT now(),
  updated_at          TIMESTAMPTZ DEFAULT now()
);

-- Services liés à un RDV (multi-prestations)
CREATE TABLE appointment_services (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  appointment_id   UUID REFERENCES appointments(id) NOT NULL,
  service_id       UUID REFERENCES services(id) NOT NULL,
  price_at_booking INTEGER NOT NULL
);

-- Paiements
CREATE TABLE payments (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  appointment_id        UUID REFERENCES appointments(id),
  salon_id              UUID REFERENCES salons(id) NOT NULL,
  amount                INTEGER NOT NULL,  -- FCFA
  currency              TEXT DEFAULT 'XAF',
  method                TEXT NOT NULL
    CHECK (method IN ('mobile_money', 'card', 'cash', 'gift_card')),
  provider              TEXT,  -- 'pawapay', 'stripe', 'cash'
  provider_transaction_id TEXT,
  status                TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded')),
  paid_at               TIMESTAMPTZ,
  created_at            TIMESTAMPTZ DEFAULT now()
);

-- File de notifications (async — ne JAMAIS appeler l'API externe depuis un trigger)
CREATE TABLE notification_queue (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type           TEXT NOT NULL CHECK (type IN ('sms', 'email', 'push', 'whatsapp')),
  appointment_id UUID REFERENCES appointments(id),
  salon_id       UUID REFERENCES salons(id) NOT NULL,
  recipient_phone TEXT,
  recipient_email TEXT,
  template        TEXT NOT NULL,
  payload         JSONB DEFAULT '{}',
  status          TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'processing', 'sent', 'failed')),
  attempts        INTEGER DEFAULT 0,
  last_error      TEXT,
  created_at      TIMESTAMPTZ DEFAULT now(),
  processed_at    TIMESTAMPTZ
);

-- Index critiques
CREATE INDEX idx_appointments_salon_date   ON appointments(salon_id, start_time);
CREATE INDEX idx_appointments_staff_date   ON appointments(salon_id, staff_id, start_time);
CREATE INDEX idx_appointments_client       ON appointments(client_id, start_time);
CREATE INDEX idx_services_salon            ON services(salon_id) WHERE is_active = true;
CREATE INDEX idx_staff_salon               ON staff_members(salon_id) WHERE is_active = true;
CREATE INDEX idx_staff_hours               ON staff_working_hours(staff_id, day_of_week);
CREATE INDEX idx_payments_appointment      ON payments(appointment_id);
CREATE INDEX idx_notification_queue_pending ON notification_queue(status, created_at)
  WHERE status = 'pending';
```

---

## Algorithme de disponibilité (pièce la plus critique)

**Implémenté comme fonction PostgreSQL** — jamais en code applicatif.

Raisons :
- Atomique : lit horaires, pauses, exceptions, RDV existants en une seule transaction
- Rapide : pas de round-trips entre app et DB
- Consistent : même logique web et mobile
- SECURITY DEFINER : filtre par salon_id en paramètre, pas via RLS

```sql
CREATE OR REPLACE FUNCTION get_available_slots(
  p_salon_id   UUID,
  p_date       DATE,
  p_service_id UUID,
  p_staff_id   UUID DEFAULT NULL,
  p_slot_interval INTEGER DEFAULT 15
)
RETURNS TABLE (
  slot_start       TIMESTAMPTZ,
  slot_end         TIMESTAMPTZ,
  available_staff_id UUID,
  staff_name       TEXT
) LANGUAGE plpgsql SECURITY DEFINER
AS $$
DECLARE
  v_duration   INTEGER;
  v_day_of_week INTEGER;
BEGIN
  SELECT duration_minutes INTO v_duration
  FROM services WHERE id = p_service_id AND salon_id = p_salon_id;

  v_day_of_week := EXTRACT(DOW FROM p_date);

  RETURN QUERY
  WITH eligible_staff AS (
    SELECT sm.id AS staff_id,
           sm.first_name || ' ' || sm.last_name AS name,
           swh.start_time AS work_start,
           swh.end_time   AS work_end
    FROM staff_members sm
    JOIN staff_working_hours swh ON swh.staff_id = sm.id
    WHERE sm.salon_id = p_salon_id
      AND sm.is_active = true
      AND swh.day_of_week = v_day_of_week
      AND swh.is_working = true
      AND (p_staff_id IS NULL OR sm.id = p_staff_id)
      AND NOT EXISTS (
        SELECT 1 FROM staff_exceptions se
        WHERE se.staff_id = sm.id
          AND se.date = p_date
          AND se.is_available = false
      )
  ),
  existing_bookings AS (
    SELECT a.staff_id, a.start_time, a.end_time
    FROM appointments a
    WHERE a.salon_id = p_salon_id
      AND a.start_time::date = p_date
      AND a.status IN ('pending', 'confirmed')
  ),
  staff_breaks_today AS (
    SELECT sb.staff_id,
           sb.start_time AS break_start,
           sb.end_time   AS break_end
    FROM staff_breaks sb
    WHERE sb.salon_id = p_salon_id
      AND sb.day_of_week = v_day_of_week
  )
  SELECT
    generate_series(
      (p_date + es.work_start)::timestamptz,
      (p_date + es.work_end - (v_duration || ' minutes')::interval)::timestamptz,
      (p_slot_interval || ' minutes')::interval
    ) AS slot_start,
    generate_series(
      (p_date + es.work_start)::timestamptz,
      (p_date + es.work_end - (v_duration || ' minutes')::interval)::timestamptz,
      (p_slot_interval || ' minutes')::interval
    ) + (v_duration || ' minutes')::interval AS slot_end,
    es.staff_id,
    es.name
  FROM eligible_staff es
  WHERE NOT EXISTS (
    SELECT 1 FROM existing_bookings eb
    WHERE eb.staff_id = es.staff_id
      AND eb.start_time < slot_start + (v_duration || ' minutes')::interval
      AND eb.end_time > slot_start
  )
  AND NOT EXISTS (
    SELECT 1 FROM staff_breaks_today sbt
    WHERE sbt.staff_id = es.staff_id
      AND (p_date + sbt.break_start)::timestamptz < slot_start + (v_duration || ' minutes')::interval
      AND (p_date + sbt.break_end)::timestamptz > slot_start
  )
  AND slot_start > now()
  ORDER BY slot_start, es.name;
END;
$$;
```

---

## Flux de données

### Flux 1 : Client réserve un créneau

```
1. Client → /[salon-slug] (page publique, Server Component)
2. Fetch salon, services, staff (RLS public policy)
3. Sélection service → Server Action appelle get_available_slots()
4. Sélection créneau → Server Action crée le RDV :
   BEGIN
     SELECT FOR UPDATE sur les RDV chevauchants (anti double-booking)
     INSERT appointments (status: 'pending')
     INSERT appointment_services
   COMMIT
5. Si paiement requis → redirect pawaPay
6. Webhook pawaPay → Edge Function → UPDATE payment + appointment status
7. DB trigger → notification_queue (INSERT)
8. Supabase Realtime → notifie Pro Dashboard
9. pg_cron toutes les 5min → traite notification_queue → Twilio SMS
```

### Flux 2 : Walk-in (spécifique Afrique)

```
1. Pro ouvre l'agenda
2. Clic "Walk-in" sur un créneau
3. Saisie rapide : client (optionnel) + service + durée
4. INSERT appointment (type: 'walk_in', status: 'confirmed')
5. Agenda mis à jour via Realtime
6. Encaissement direct en caisse
```

### Flux 3 : Realtime agenda

```typescript
// 1. Charger l'état initial
const { data: appointments } = await supabase
  .from('appointments')
  .select('*, staff:staff_members(*), services:appointment_services(*, service:services(*))')
  .eq('salon_id', salonId)
  .gte('start_time', startOfWeek)
  .lte('start_time', endOfWeek);

// 2. Subscribe APRÈS le chargement initial (éviter les updates manqués)
const channel = supabase.channel(`agenda:${salonId}`)
  .on('postgres_changes', {
    event: '*',
    schema: 'public',
    table: 'appointments',
    filter: `salon_id=eq.${salonId}`
  }, handleRealtimeChange)
  .subscribe();
```

### Flux 4 : Paiement mobile money (async)

```
Abstraction Layer (Strategy Pattern)
├── PaymentProvider (interface)
│   ├── initializePayment(amount, currency, metadata) → { paymentUrl, transactionId }
│   ├── verifyPayment(transactionId) → PaymentStatus
│   └── handleWebhook(payload, signature) → PaymentEvent
├── PawapayProvider (Airtel + Moov Gabon)
├── StripeProvider (CB via Kairo Digital France)
└── CashProvider (enregistrement manuel)

// Idempotence webhook (éviter double-confirmation)
UPDATE payments
SET status = 'completed', paid_at = now()
WHERE provider_transaction_id = $1
  AND status != 'completed';
```

### Flux 5 : Pipeline SMS notifications

```
Déclencheurs event-driven (immédiat via DB trigger → notification_queue) :
├── RDV confirmé → SMS client + pro
├── RDV annulé → SMS client + pro
└── Paiement reçu → SMS reçu client

Déclencheurs programmés (pg_cron toutes les 15 min) :
├── J-24h → SMS rappel client
└── No-show détection → flag appointments expirés

⚠️ RÈGLE : Ne JAMAIS appeler une API externe depuis un DB trigger
   Toujours insérer dans notification_queue et traiter en async
```

---

## Décisions d'implémentation clés

### Prisma vs Supabase client

**Décision :** Prisma pour migrations + schema management uniquement. Supabase client (JavaScript) pour toutes les queries à l'exécution.

**Raison :** Prisma se connecte directement à PostgreSQL et bypasse RLS par défaut (il faudrait `SET app.current_salon_id` à chaque query). Le Supabase client utilise le JWT de l'utilisateur connecté → RLS s'applique automatiquement.

```typescript
// ✅ Correct — RLS automatique
const { data } = await supabase
  .from('appointments')
  .select('*')  // RLS filtre par salon_id du JWT

// ❌ Dangereux — bypass RLS sans config explicite
const appointments = await prisma.appointments.findMany()
```

### Pattern Result sur tous les Server Actions

```typescript
// Toujours { data, error } — jamais de throw nu
type Result<T> = { data: T; error: null } | { data: null; error: string }

async function createAppointment(input: CreateAppointmentInput): Promise<Result<Appointment>> {
  const parsed = CreateAppointmentSchema.safeParse(input)
  if (!parsed.success) return { data: null, error: parsed.error.message }
  // ...
}
```

---

## Anti-patterns à éviter

| Anti-pattern | Conséquence | Solution |
|---|---|---|
| Calcul des slots en code applicatif | Race conditions, double-bookings | Fonction PostgreSQL `get_available_slots()` |
| Stocker les créneaux pré-générés | Données obsolètes, bloat | Calcul à la demande |
| Appels API externes dans les DB triggers | Bloque la transaction | notification_queue + Edge Function async |
| Filtrage tenant uniquement en code | Data leak si bug | RLS au niveau DB |
| Un seul endpoint webhook pour tous les providers | Impossible à déboguer | Un Edge Function par provider |

---

## Ordre de build recommandé (chemin critique)

```
Semaine 1 : Fondation
  → Schéma Supabase + toutes les migrations Prisma
  → RLS policies + index
  → Auth OTP téléphone (Supabase Phone Auth)
  → Scaffold Next.js 15 + Tailwind + shadcn/ui

Semaine 2 : Dashboard pro — configuration
  → Onboarding wizard (salon + services + staff + horaires)
  → Pages de configuration du salon

Semaine 3 : Moteur de disponibilité + réservation
  → Fonction PostgreSQL get_available_slots()
  → Transaction de booking avec SELECT FOR UPDATE
  → Tests de concurrence (double-booking)

Semaine 3-4 : Flux client booking
  → Page publique salon (/[slug])
  → Tunnel de réservation (service → créneau → confirmation)
  → OTP client + gestion de compte basique

Semaine 4 : Agenda temps réel
  → Vue agenda pro (jour/semaine)
  → Walk-ins (ajout direct)
  → Supabase Realtime subscriptions

Semaine 5 : Paiements
  → Payment abstraction layer (Strategy pattern)
  → Intégration pawaPay (Airtel + Moov)
  → Encaissement cash
  → Webhooks Edge Functions + idempotence

Semaine 5-6 : SMS + Dashboard
  → notification_queue pattern
  → Intégration Twilio
  → pg_cron rappels 24h
  → Dashboard CA (jour/mois FCFA)
```

**Dépendances critiques :**
1. Schéma + RLS + Auth → tout dépend de l'isolation tenant
2. Moteur disponibilité → précède booking client
3. Booking → précède paiements
4. Webhooks paiement → précèdent SMS confirmation
5. notification_queue pattern établi en Phase 1 → réutilisé Phase 2+ (WhatsApp, push)

---

*Research date: 2026-04-10*

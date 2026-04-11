# Pitfalls Research: Bookly

**Project:** Bookly — SaaS booking beauté, Gabon
**Date:** 2026-04-10
**Confidence:** HIGH

---

## 🚨 2 Blockers pré-Phase 1 (showstoppers)

Ces deux points invalident des hypothèses dans PROJECT.md. Ils doivent être résolus avant d'écrire la moindre ligne de code de paiement ou SMS.

### BLOCKER 1 — Stripe n'opère pas au Gabon

**Problème :** Stripe supporte 46+ pays mais le Gabon est absent. Pas de compte marchand, pas de payout FCFA, pas de traitement CB local.

**Impact :** Le PROJECT.md liste Stripe dans les Constraints et Requirements. Utiliser Stripe directement = impossible.

**Solution :**
- **pawaPay** = provider principal (Airtel Money Gabon confirmé live)
- **Moneroo** = fallback (Airtel + Moov + Visa/MC Gabon listé)
- **Stripe** = secondaire via entité française Kairo Digital (CB internationale)

**Phase à corriger :** Avant Phase 1 — décision architecture critique.

---

### BLOCKER 2 — Africa's Talking ne couvre pas le Gabon

**Problème :** Africa's Talking couvre l'Afrique de l'Est et de l'Ouest (Kenya, Nigeria, Uganda, Ghana, CI, etc.). Gabon est absent de leur liste officielle de pays supportés.

**Impact :** Toutes les features SMS (confirmation, rappel 24h, notification pro) sont bloquées si Africa's Talking est utilisé.

**Solution :**
- **Twilio** = provider SMS (couverture Gabon +241 confirmée, ~0,20$/SMS)
- **WhatsApp Business API** = canal principal (pénétration haute Gabon, ~10x moins cher que SMS)
- Déposer la demande Meta Business Manager dès Phase 1 (délai 4-8 semaines)

**Phase à corriger :** Avant Phase 1 — impact sur schema (notification_queue), architecture et env variables.

---

## Pitfalls critiques

### P1 — Double-booking race condition

**Description :** Deux clients réservent le même créneau simultanément. La vérification "créneau libre ?" réussit pour les deux avant qu'aucun ne commit.

**Signes d'alerte :**
- Deux RDV sur le même collaborateur à la même heure
- Plaintes clients après réservation confirmée

**Prévention :**

```sql
-- Activer l'extension btree_gist (obligatoire)
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Contrainte d'exclusion sur la table appointments
ALTER TABLE appointments
ADD CONSTRAINT no_double_booking
EXCLUDE USING gist (
  staff_id WITH =,
  tstzrange(start_time, end_time, '[)') WITH &&
)
WHERE (status IN ('pending', 'confirmed'));
```

```typescript
// Booking via RPC atomique — JAMAIS un SELECT séparé + INSERT depuis le client JS
const { data, error } = await supabase.rpc('create_booking', {
  p_salon_id: salonId,
  p_service_id: serviceId,
  p_staff_id: staffId,
  p_start_time: startTime,
  p_client_id: clientId,
});
```

**Phase :** Semaine 1, avant tout code de booking.

---

### P2 — Flux paiement mobile money asynchrone

**Description :** Les confirmations Airtel/Moov prennent 5-60 secondes (ou n'arrivent jamais). Si le booking attend la confirmation avant d'être créé, le créneau reste disponible et peut être repris.

**Signes d'alerte :**
- Clients qui paient mais n'ont pas de RDV confirmé
- Créneaux "fantômes" bloqués pour toujours

**Prévention :**

```sql
-- État intermédiaire dans appointments
status CHECK (status IN (
  'pending',           -- RDV créé, paiement non initié
  'pending_payment',   -- Paiement initié, en attente confirmation
  'confirmed',         -- Paiement confirmé
  'cancelled',
  'completed',
  'no_show'
))
```

```typescript
// Flux complet
// 1. Créer le RDV (status: 'pending_payment')
// 2. Bloquer le créneau pendant 10 minutes
// 3. Rediriger vers pawaPay
// 4. Webhook pawaPay → Edge Function → UPDATE status 'confirmed'
// 5. pg_cron toutes les 5min → libérer les créneaux 'pending_payment' > 10min

// Idempotence webhook (éviter double-confirmation)
UPDATE payments
SET status = 'completed', paid_at = now()
WHERE provider_transaction_id = $1
  AND status != 'completed'
RETURNING *;
```

**Phase :** Phase 1 — partie intégrante du flux de réservation.

---

### P3 — Fuite de données multi-tenant (RLS)

**Description :** CVE-2025-48757 a exposé les données de 170+ applications Supabase. Les migrations Prisma n'activent PAS RLS automatiquement. Le code généré par IA met souvent `USING (true)` par défaut.

**Signes d'alerte :**
- Table sans RLS activé = tous les salons voient toutes les données
- Policy `USING (true)` = pas d'isolation

**Prévention :**

```sql
-- 1. Activer RLS sur CHAQUE table (Prisma ne le fait pas)
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
ALTER TABLE services ENABLE ROW LEVEL SECURITY;
ALTER TABLE staff_members ENABLE ROW LEVEL SECURITY;
-- ... toutes les tables tenant-scoped

-- 2. Pattern SECURITY DEFINER pour résolution tenant
CREATE OR REPLACE FUNCTION get_current_salon_id()
RETURNS UUID LANGUAGE sql STABLE SECURITY DEFINER
AS $$
  SELECT (auth.jwt() ->> 'salon_id')::uuid;
$$;

-- 3. Policy correcte (pas USING (true) !)
CREATE POLICY "salon_isolation" ON appointments
  FOR ALL
  USING (salon_id = get_current_salon_id());
```

```typescript
// 4. Suite de tests cross-tenant obligatoire Phase 1
test('Salon A ne peut pas voir les RDV du Salon B', async () => {
  const salonAClient = createClientWithJWT({ salon_id: salonAId });
  const { data } = await salonAClient.from('appointments').select('*');
  expect(data?.every(a => a.salon_id === salonAId)).toBe(true);
});
```

**Phase :** Semaine 1 — migration initiale + tests de sécurité obligatoires.

---

### P4 — Prisma bypasse RLS

**Description :** Prisma se connecte directement à PostgreSQL avec le service_role, qui bypasse RLS. Si Prisma est utilisé pour les queries à l'exécution, les policies sont invisibles.

**Signes d'alerte :**
- Query Prisma retourne des données cross-tenant
- RLS policies jamais déclenchées en développement

**Prévention :**

```typescript
// ❌ DANGEREUX — bypasse RLS
const appointments = await prisma.appointments.findMany()

// ✅ CORRECT — RLS automatique via JWT
const { data } = await supabase
  .from('appointments')
  .select('*')

// Règle : Prisma uniquement pour migrations
// Toutes les queries runtime → Supabase JS client
// Dans les Server Actions :
const supabase = createServerClient(cookies()) // JWT de l'utilisateur connecté
```

**Phase :** Semaine 1 — établir la règle dès le scaffold.

---

### P5 — Dégradation de performance RLS à l'échelle

**Description :** Les policies RLS mal écrites peuvent rendre chaque query O(n²). Pattern typique : subquery corrélée sur chaque ligne.

**Prévention :**

```sql
-- ❌ Pattern lent (subquery évaluée par ligne)
CREATE POLICY "slow" ON appointments
  FOR SELECT USING (
    EXISTS (SELECT 1 FROM salons WHERE id = appointments.salon_id
            AND owner_id = auth.uid())
  );

-- ✅ Pattern rapide (comparison directe avec index)
CREATE POLICY "fast" ON appointments
  FOR SELECT USING (salon_id = get_current_salon_id());

-- Index obligatoires sur toutes les colonnes dans les policies
CREATE INDEX idx_appointments_salon ON appointments(salon_id);
CREATE INDEX idx_services_salon     ON services(salon_id);
-- etc.
```

**Phase :** Phase 1 (index), Phase 3 (audit performance analytics).

---

## Pitfalls modérés

### P6 — Race condition Supabase Realtime sur l'agenda

**Description :** Realtime est broadcast, pas conflict resolution. Deux pros qui modifient le même RDV simultanément peuvent se perdre des mises à jour.

**Prévention :**
- Optimistic UI + reconciliation serveur
- Colonne `updated_at` sur appointments pour détecter les conflits
- En cas de conflit → afficher un message "RDV modifié par un autre utilisateur"

**Phase :** Phase 1 — conception agenda.

---

### P7 — Abandon à l'onboarding

**Description :** Les recherches sur les SaaS africains montrent que le Time-To-First-Use (TTFU) doit être < 60 secondes. Les pros beauté sont souvent peu à l'aise avec le digital.

**Prévention :**
- Templates de services pré-remplis par catégorie (coiffeur, barbier, esthétique, ongles)
- Wizard 4 étapes avec progress bar visible
- Onboarding via WhatsApp possible (lien direct)
- Aucun champ optionnel obligatoire en Phase 1

**Phase :** Phase 1 — onboarding wizard.

---

### P8 — Connectivité / offline

**Description :** Libreville a une bonne 4G mais les micro-coupures sont fréquentes. Les coupures électriques représentent jusqu'à 31% de perte CA pour les PME en Afrique subsaharienne.

**Prévention :**
- Service Worker (PWA) pour mise en cache de l'agenda
- Retry automatique sur les mutations critiques (booking, paiement)
- États de chargement clairs + messages d'erreur réseau
- Mode offline lecture seule (voir son agenda sans internet)

**Phase :** Phase 2 — PWA offline mode.

---

### P9 — Explosion des coûts SMS

**Description :** 20 RDV/jour × 3 SMS/RDV × 0,20$ = 360$/mois en SMS. L'abonnement Bookly à 34 900 FCFA (~60$) ne couvre pas ce coût.

**Prévention :**
- WhatsApp Business API = canal principal dès Phase 2 (~0,02-0,05$/conversation)
- Soumettre la demande Meta dès Phase 1 (délai 4-8 semaines)
- SMS = fallback uniquement pour clients sans WhatsApp
- Inclure 50 SMS/mois dans l'abonnement, facturer le surplus

**Phase :** Décision à prendre en Phase 1, implémentation Phase 2.

---

## Pitfalls mineurs

### P10 — Over-engineering timezone

**Gabon = UTC+1 toute l'année, sans DST depuis 1912.** Ne pas trop complexifier.

```typescript
// Toujours stocker en UTC dans la DB
// Afficher en Africa/Libreville (UTC+1)
const localTime = formatInTimeZone(utcDate, 'Africa/Libreville', 'HH:mm');
```

**Phase :** Phase 1 — config Supabase `default_timezone = 'UTC'`.

---

### P11 — Erreurs de calcul FCFA

**FCFA = devise zéro-décimal. Ne jamais utiliser float ou number avec décimales.**

```typescript
// ❌ DANGEREUX
const price = 9900.00; // Float — erreurs d'arrondi
await supabase.from('services').insert({ price: 9900.50 }); // Décimale en DB

// ✅ CORRECT
const price = 9900; // Integer uniquement
// Colonne DB : INTEGER (pas DECIMAL, pas FLOAT)

// Affichage
const formatted = new Intl.NumberFormat('fr-FR', {
  style: 'currency',
  currency: 'XAF',
  minimumFractionDigits: 0,
  maximumFractionDigits: 0,
}).format(9900); // → "9 900 FCFA"
```

**Phase :** Phase 1 — schema initial.

---

## Récapitulatif par phase

| Phase | Pitfalls à adresser |
|---|---|
| **Pré-Phase 1** | BLOCKER 1 (pawaPay), BLOCKER 2 (Twilio + WhatsApp) |
| **Phase 1 — Semaine 1** | P3 (RLS), P4 (Prisma), P1 (btree_gist), P11 (FCFA integers) |
| **Phase 1 — Semaine 3** | P1 (RPC booking atomique), P2 (pending_payment), P5 (index) |
| **Phase 1 — Semaine 4** | P6 (Realtime), P7 (Onboarding TTFU) |
| **Phase 1 — Semaine 5** | P2 (webhooks idempotents), P10 (timezone) |
| **Phase 2** | P8 (offline PWA), P9 (WhatsApp Business API) |
| **Phase 3** | P5 (RLS performance analytics), P9 (SMS cost modeling) |
| **Phase 4** | P8 (offline Expo RN), multi-pays timezone |

---

## Questions ouvertes

1. **pawaPay vs Moneroo** — pawaPay confirme Airtel Gabon, Moneroo liste Airtel + Moov. Valider les deux via onboarding sandbox.
2. **WhatsApp API tarif Gabon** — vérifier le rate card Meta pour les messages template +241.
3. **Airtel Money direct vs agrégateur** — Airtel domine ~90% du mobile money au Gabon. Une intégration directe économise les frais d'agrégateur mais ajoute de la maintenance.
4. **Supabase Realtime comportement réseau Gabon** — latence et reconnexion à tester sur le terrain.

---

*Research date: 2026-04-10*

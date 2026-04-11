# Research Summary: Bookly

**Project:** Bookly — SaaS booking beauté, Gabon / Afrique francophone
**Date:** 2026-04-11
**Overall Confidence:** MEDIUM-HIGH

---

## Executive Summary

Bookly est un SaaS vertical multi-tenant de gestion de salon beauté ciblant le marché gabonais. Aucun concurrent direct n'existe — le vrai concurrent est le cahier papier et WhatsApp. Deux corrections critiques s'imposent vs PROJECT.md initial : Africa's Talking ne couvre pas le Gabon (utiliser Twilio), et Stripe ne peut pas être utilisé directement au Gabon (passer par Kairo Digital France). Le rail de paiement primaire est mobile money (pawaPay / Airtel Money), pas la carte bancaire. L'architecture est shared schema + RLS PostgreSQL avec `salon_id` dans le JWT.

---

## ⚠️ Corrections critiques au PROJECT.md

| Hypothèse PROJECT.md | Correction validée |
|---|---|
| Africa's Talking SMS | ❌ **Twilio obligatoire** — Africa's Talking ne couvre pas le Gabon |
| CinetPay mobile money | ❌ **pawaPay obligatoire** — CinetPay ne couvre pas le Gabon |
| Stripe natif Gabon | ❌ **Via entité française Kairo Digital** — Stripe n'opère pas au Gabon |
| Auth email/password | ❌ **OTP téléphone P0** — email secondaire uniquement |
| Walk-ins hors scope | ❌ **P0** — 40-60% des clients arrivent sans RDV en Afrique |
| Next.js 14 | → Upgrader vers **Next.js 15** (LTS, React 19, Turbopack stable) |
| FCFA avec décimales | → **INTEGER en DB**, format d'affichage `9 900 FCFA` |

---

## Stack validée

### Frontend Web
- Next.js 15 (App Router, React 19, Turbopack)
- TypeScript 5.x strict
- Tailwind CSS 3.4 + shadcn/ui
- Zustand 5.x (state agenda)
- React Hook Form 7.x + Zod 3.x

### Backend
- Supabase Cloud — région **London (eu-west-2)** (~120-150ms depuis Libreville)
- Prisma 5.x — migrations uniquement (Supabase JS client pour les queries)
- Server Actions Next.js 15 pour les mutations
- RLS activé sur toutes les tables (isolation multi-tenant par `salon_id`)
- Supabase Realtime pour sync agenda

### Paiements
- **pawaPay** — Airtel Money Gabon (confirmé) + Moov Money (à valider sandbox)
- **Stripe** via compte Kairo Digital France — CB internationale, abonnements SaaS
- Pas de Flutterwave, CinetPay ou FedaPay pour le Gabon

### SMS / Notifications
- **Twilio** — SMS OTP + rappels (Africa's Talking exclut le Gabon)
- **WhatsApp Business API** — Phase 2 (10x moins cher que SMS à l'échelle)

### Mobile (Phase 4)
- Expo SDK 53 (React Native, iOS + Android)
- Push notifications (Expo Notifications)

---

## Features prioritaires marché africain

### P0 — Adaptations africaines non négociables
1. **Auth OTP téléphone** — pas d'email/password en P0
2. **Walk-ins** — enregistrement client sans RDV préalable (40-60% des visites)
3. **Mobile money** — pawaPay Airtel Money au checkout (pas CB en premier)
4. **Offline PWA** — coupures électriques fréquentes = perte de CA si pas de cache
5. **Interface légère** — connexions 3G/4G lentes, pas de 5G au Gabon

### P1 — Différenciateurs locaux
- Paiement en espèces enregistré (POS)
- Rappels WhatsApp (Phase 2)
- Bot WhatsApp réservation (Phase 3)
- Payout Airtel Money vers le pro (Phase 3)

---

## Architecture — Décisions clés

### Multi-tenant
- **Shared schema + RLS** — un seul schéma PostgreSQL, isolation par `salon_id` via JWT
- `salon_id` dans le Supabase JWT → RLS automatique sans passer l'ID manuellement

### Schéma DB — Tables principales Phase 1
```
salons           → établissements (multi-tenant root)
profiles         → users Supabase (clients + pros + admins)
professionals    → employés du salon
services         → prestations
service_pros     → junction (prestation × employé)
business_hours   → horaires par salon
appointments     → rendez-vous (avec contrainte btree_gist anti double-booking)
appointment_items→ prestations liées à un RDV
payments         → transactions paiement
notification_queue → file d'envoi notifications async
```

### Patterns critiques
- **`get_available_slots()` = fonction PostgreSQL** — atomique, pas de round-trips
- **Contrainte `btree_gist`** sur appointments — anti double-booking au niveau DB
- **`pending_payment` status** + blocage créneau 10 min — gère l'async mobile money
- **`notification_queue` pattern** dès Phase 1 — réutilisé WhatsApp/push sans refacto
- **Un Edge Function par provider webhook** — isolé, debuggable

---

## 🚨 Blockers pré-Phase 1

| Blocker | Action requise | Délai |
|---|---|---|
| KYB pawaPay | Démarrer l'inscription sandbox + KYB business **maintenant** | 2-4 semaines |
| WhatsApp Business API | Soumettre demande Meta **maintenant** (pour Phase 2) | 4-8 semaines |

---

## Risques identifiés

| Risque | Impact | Mitigation |
|---|---|---|
| pawaPay Moov Money Gabon non confirmé | Haut | Valider sandbox avant code paiement |
| Latence Supabase London → Libreville | Moyen | Tester en conditions réelles dès le début |
| Conformité fiscale locale gabonaise | Haut (Phase 2) | Consulter comptable local avant lancement commercial |
| Apple/Google stores pour apps mobile money | Moyen (Phase 4) | Vérifier politique stores dès Phase 3 |

---

## Roadmap recommandée (4 phases)

| Phase | Durée | Objectif |
|---|---|---|
| **1** — Fondations MVP | 6 semaines | Schéma + RLS + Auth OTP + Agenda + Walk-ins + pawaPay + SMS Twilio |
| **2** — Rétention + Canaux | 4 semaines | PWA offline + WhatsApp + POS complet + fidélité |
| **3** — Growth + Monétisation | 4 semaines | Bot WhatsApp + payout pro + service à domicile |
| **4** — Mobile + Marketplace | 6 semaines | Expo client app + push + marketplace géolocalisée |

**Chemin critique Phase 1 :** schéma DB → disponibilités → booking → paiements → SMS

---

## Ready for Roadmap

Tous les fichiers de recherche disponibles dans `.planning/research/`. L'orchestrateur peut procéder à la création du ROADMAP.md.

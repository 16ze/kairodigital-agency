# Roadmap: Bookly

## Overview

Bookly livre en 4 phases : d'abord le coeur MVP qui permet à un pro d'encaisser des réservations en ligne via Airtel Money et de gérer son agenda avec les walk-ins (Phase 1) ; puis la couche rétention avec PWA offline, CRM, caisse POS et WhatsApp (Phase 2) ; puis le growth avec analytics avancés, campagnes marketing, gestion équipe et bot WhatsApp (Phase 3) ; enfin la marketplace publique et l'app mobile client iOS/Android (Phase 4).

---

## Stack validee (corrections vs PROJECT.md initial)

| Hypothese initiale | Decision finale | Raison |
|---|---|---|
| Africa's Talking (SMS) | **Twilio** | Africa's Talking ne couvre pas le Gabon |
| CinetPay (mobile money) | **pawaPay** | CinetPay ne couvre pas le Gabon |
| Stripe natif Gabon | **Stripe via Kairo Digital France** | Stripe n'opere pas directement au Gabon |
| Next.js 14 | **Next.js 15** (React 19, Turbopack) | LTS stable, Turbopack production-ready |
| Auth email/password | **OTP telephone uniquement** | Email non fiable comme identite en Afrique |
| FCFA avec decimales | **INTEGER en DB** | La devise locale n'a pas de centimes |

---

## Blockers pre-Phase 1 (demarrer maintenant)

> Ces deux demarches ont des delais incompressibles — ne pas attendre le debut du code.

| Blocker | Action requise | Delai estime |
|---|---|---|
| **pawaPay KYB** | Demarrer l'inscription sandbox + verification business sur pawaPay | 2-4 semaines |
| **WhatsApp Business API** | Soumettre la demande d'acces Meta Business (necessite pour Phase 2) | 4-8 semaines |

---

## Phases

- [ ] **Phase 1: MVP Core** - Un pro peut s'inscrire, configurer son salon, recevoir des reservations en ligne + walk-ins, et encaisser via Airtel Money ou cash
- [ ] **Phase 2: Retention + Canaux** - Le pro garde ses clients avec CRM, caisse POS complete, fidelite, et notifications WhatsApp
- [ ] **Phase 3: Growth + Monetisation** - Le pro developpe son activite avec analytics avances, campagnes marketing, gestion equipe, bot WhatsApp et payout mobile
- [ ] **Phase 4: Mobile + Marketplace** - Les clients decouvrent les salons via la marketplace, et utilisent l'app mobile iOS/Android

---

## Phase Details

### Phase 1: MVP Core

**Goal**: Un pro peut s'inscrire, configurer son salon, recevoir des reservations en ligne et des walk-ins 24/7, et encaisser via Airtel Money ou cash — sans jamais decrocher le telephone.

**Depends on**: Rien (premiere phase) — mais voir blockers pre-Phase 1 ci-dessus

**Duration**: 6 semaines

**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, ONBOARD-01, ONBOARD-02, ONBOARD-03, ONBOARD-04, ONBOARD-05, BOOKING-01, BOOKING-02, BOOKING-03, BOOKING-04, BOOKING-05, BOOKING-06, BOOKING-07, BOOKING-08, WALKIN-01, WALKIN-02, WALKIN-03, WALKIN-04, AGENDA-01, AGENDA-02, AGENDA-03, AGENDA-04, AGENDA-05, AGENDA-06, PAYMENT-01, PAYMENT-02, PAYMENT-03, PAYMENT-04, PAYMENT-05, PAYMENT-06, NOTIF-01, NOTIF-02, NOTIF-03, NOTIF-04, NOTIF-05, DASHBOARD-01, DASHBOARD-02, DASHBOARD-03, DASHBOARD-04

**Success Criteria** (what must be TRUE):
  1. Un pro peut creer son compte via OTP telephone, configurer son salon (services, horaires, collaborateurs) et obtenir son lien partageable en moins de 10 minutes
  2. Un client peut ouvrir le lien WhatsApp du salon, choisir un service et un creneau, et recevoir un SMS de confirmation — sans creer de compte
  3. Aucun double-booking n'est possible : deux clients qui reservent le meme creneau simultanement voient l'un d'eux rejete au niveau base de donnees
  4. Un client peut payer via Airtel Money au moment de la reservation, et le pro voit le paiement confirme dans son agenda
  5. Le pro peut ajouter un walk-in depuis l'agenda en moins de 30 secondes, l'encaisser en cash, et voir le CA mis a jour sur son dashboard
  6. Le client recoit un SMS de rappel 24h avant son RDV via Twilio, avec lien d'annulation fonctionnel

**Chemin critique Phase 1:**
```
Schema DB + migrations Prisma
  → RLS PostgreSQL + salon_id dans JWT
    → Auth OTP (Supabase + Twilio)
      → Onboarding pro (salon, services, horaires)
        → Moteur de disponibilites (get_available_slots() + btree_gist)
          → Flux de reservation client (page publique)
            → Paiement pawaPay (pending_payment + slot lock 10min + webhook)
              → Notifications SMS (notification_queue + Twilio)
                → Agenda pro (vue jour/semaine + Realtime)
                  → Walk-ins + encaissement cash
                    → Dashboard CA
```

**Plans**: TBD

Plans:
- [ ] 01-01: Schema DB, migrations Prisma, RLS multi-tenant
- [ ] 01-02: Auth OTP telephone (Supabase + Twilio)
- [ ] 01-03: Onboarding pro (salon, services, horaires, collaborateurs, lien partageable)
- [ ] 01-04: Moteur de disponibilites (get_available_slots, btree_gist, pending_payment)
- [ ] 01-05: Flux de reservation client (page publique, selection service/creneau)
- [ ] 01-06: Paiements Airtel Money via pawaPay (checkout + webhook Edge Function)
- [ ] 01-07: Encaissement cash + walk-ins
- [ ] 01-08: Notifications SMS (confirmation, rappel 24h, notification pro) via notification_queue
- [ ] 01-09: Agenda pro (vue jour, vue semaine, Realtime, responsive mobile)
- [ ] 01-10: Dashboard CA (jour, mois, RDV du jour, prochains RDV 24h)

---

### Phase 2: Retention + Canaux

**Goal**: Le pro retient ses clients avec une fiche CRM complete, une caisse POS sur tablette, un programme de fidelite, et des notifications WhatsApp moins couteuses que les SMS.

**Depends on**: Phase 1

**Duration**: 4 semaines

**Requirements**: CRM-01, CRM-02, CRM-03, CRM-04, POS-01, POS-02, POS-03, POS-04, FIDELITY-01, FIDELITY-02, FIDELITY-03, FIDELITY-04, COMMS-01, DEPOSIT-01, DEPOSIT-02, PWA-01, PWA-02, PWA-03

**Success Criteria** (what must be TRUE):
  1. Le pro peut ouvrir la fiche d'un client et voir l'historique complet de ses visites, le montant total depense, et les notes ajoutees manuellement
  2. Depuis l'interface caisse (POS), le pro peut encaisser un panier multi-prestations en combinant cash + Airtel Money en un seul encaissement, et envoyer le recu par SMS
  3. Un client confirme sa reservation et recoit la confirmation sur WhatsApp (pas SMS) — le pro a configure la preference de notification
  4. Un client cumule des points a chaque visite et peut les utiliser comme reduction lors de l'encaissement suivant
  5. Le pro peut exiger un acompte (pourcentage configurable) a la reservation pour les services premium
  6. L'agenda est consultable et les walk-ins enregistrables sans connexion internet (PWA offline), avec synchronisation automatique au retour du reseau

**Plans**: TBD

Plans:
- [ ] 02-01: CRM clients (fiche complete, historique, notes)
- [ ] 02-02: Interface caisse POS (panier multi-prestations, encaissement mixte, recu SMS)
- [ ] 02-03: Fidelite et cartes cadeaux
- [ ] 02-04: Notifications WhatsApp Business API (remplacement/complement SMS)
- [ ] 02-05: Acompte et prepaiement a la reservation
- [ ] 02-06: PWA offline (agenda cache, walk-ins offline, installable)

---

### Phase 3: Growth + Monetisation

**Goal**: Le pro developpe son activite avec un dashboard analytics actionnable, des campagnes SMS/WhatsApp ciblees, la gestion des plannings de son equipe, et un bot WhatsApp qui reserve automatiquement pour ses clients.

**Depends on**: Phase 2

**Duration**: 4 semaines

**Requirements**: ANALYTICS-01, ANALYTICS-02, ANALYTICS-03, TEAM-01, TEAM-02, TEAM-03, COMMS-02, COMMS-03, COMMS-04, GROWTH-01, GROWTH-02

**Success Criteria** (what must be TRUE):
  1. Le pro voit son taux d'occupation par collaborateur et par service sur les 30 derniers jours, avec un graphique d'evolution hebdomadaire
  2. Le pro peut envoyer une campagne SMS aux clients qui ne sont pas revenus depuis plus de 30 jours, avec un message personnalise
  3. Le pro peut creer des shifts hebdomadaires pour ses collaborateurs et saisir leurs absences — les creneaux indisponibles sont immediatement bloques dans le moteur de reservation
  4. Un client peut envoyer "Reserver" sur WhatsApp au salon et le bot guide la reservation jusqu'a la confirmation sans intervention du pro
  5. Le pro peut exporter un CSV de toutes les transactions du mois pour sa comptabilite
  6. Le pro peut recevoir ses fonds directement sur son wallet Airtel Money (payout)

**Plans**: TBD

Plans:
- [ ] 03-01: Analytics avances (taux d'occupation, top services/collaborateurs, graphiques CA)
- [ ] 03-02: Export comptable CSV
- [ ] 03-03: Gestion equipe (shifts, absences, performance par collaborateur)
- [ ] 03-04: Campagnes SMS/WhatsApp (ponctuelle, ciblee, automatisee)
- [ ] 03-05: Bot WhatsApp reservation (Twilio/Meta Flows)
- [ ] 03-06: Payout Airtel Money vers le pro (pawaPay disbursements)

---

### Phase 4: Mobile + Marketplace

**Goal**: Les clients potentiels decouvrent les salons via une marketplace geolocalisee, et les clients existants disposent d'une app native iOS/Android avec push notifications.

**Depends on**: Phase 3

**Duration**: 6 semaines

**Requirements**: SCALE-01, SCALE-02, SCALE-03, GROWTH-03

**Success Criteria** (what must be TRUE):
  1. Un nouveau client a Libreville peut ouvrir la marketplace, filtrer par type de salon (coiffure, barbier, nail art), voir les salons sur une carte et reserver un creneau sans avoir entendu parler du salon avant
  2. L'app mobile cliente (iOS + Android) est disponible sur l'App Store et le Play Store, avec push notifications pour les confirmations et rappels de RDV
  3. Un pro peut generer un widget de reservation et l'integrer sur son site web ou son profil Instagram via un lien ou un code iframe
  4. Un professionnel mobile (domicile) peut creer une intervention geolocalisee et le client voit l'adresse de rendez-vous confirmee dans l'app

**Plans**: TBD

Plans:
- [ ] 04-01: Marketplace publique (recherche, geolocalisation, filtres, profils salons)
- [ ] 04-02: App mobile cliente Expo React Native (iOS + Android)
- [ ] 04-03: Push notifications (Expo Notifications)
- [ ] 04-04: Widget de reservation integrable (site web, reseaux sociaux)
- [ ] 04-05: Service a domicile (interventions geolocalisees)

---

## Progress

**Execution Order:** 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. MVP Core | 0/10 | Not started | - |
| 2. Retention + Canaux | 0/6 | Not started | - |
| 3. Growth + Monetisation | 0/6 | Not started | - |
| 4. Mobile + Marketplace | 0/5 | Not started | - |

---

## Coverage

**v1 Requirements (43 total):**

| ID | Phase |
|----|-------|
| AUTH-01 | Phase 1 |
| AUTH-02 | Phase 1 |
| AUTH-03 | Phase 1 |
| AUTH-04 | Phase 1 |
| AUTH-05 | Phase 1 |
| ONBOARD-01 | Phase 1 |
| ONBOARD-02 | Phase 1 |
| ONBOARD-03 | Phase 1 |
| ONBOARD-04 | Phase 1 |
| ONBOARD-05 | Phase 1 |
| BOOKING-01 | Phase 1 |
| BOOKING-02 | Phase 1 |
| BOOKING-03 | Phase 1 |
| BOOKING-04 | Phase 1 |
| BOOKING-05 | Phase 1 |
| BOOKING-06 | Phase 1 |
| BOOKING-07 | Phase 1 |
| BOOKING-08 | Phase 1 |
| WALKIN-01 | Phase 1 |
| WALKIN-02 | Phase 1 |
| WALKIN-03 | Phase 1 |
| WALKIN-04 | Phase 1 |
| AGENDA-01 | Phase 1 |
| AGENDA-02 | Phase 1 |
| AGENDA-03 | Phase 1 |
| AGENDA-04 | Phase 1 |
| AGENDA-05 | Phase 1 |
| AGENDA-06 | Phase 1 |
| PAYMENT-01 | Phase 1 |
| PAYMENT-02 | Phase 1 |
| PAYMENT-03 | Phase 1 |
| PAYMENT-04 | Phase 1 |
| PAYMENT-05 | Phase 1 |
| PAYMENT-06 | Phase 1 |
| NOTIF-01 | Phase 1 |
| NOTIF-02 | Phase 1 |
| NOTIF-03 | Phase 1 |
| NOTIF-04 | Phase 1 |
| NOTIF-05 | Phase 1 |
| DASHBOARD-01 | Phase 1 |
| DASHBOARD-02 | Phase 1 |
| DASHBOARD-03 | Phase 1 |
| DASHBOARD-04 | Phase 1 |

**v2 Requirements (mapped to Phase 2):**
CRM-01, CRM-02, CRM-03, CRM-04, POS-01, POS-02, POS-03, POS-04, FIDELITY-01, FIDELITY-02, FIDELITY-03, FIDELITY-04, COMMS-01, DEPOSIT-01, DEPOSIT-02, PWA-01, PWA-02, PWA-03

**v3+ Requirements (mapped to Phase 3 & 4):**
ANALYTICS-01, ANALYTICS-02, ANALYTICS-03, TEAM-01, TEAM-02, TEAM-03, COMMS-02, COMMS-03, COMMS-04, GROWTH-01, GROWTH-02, GROWTH-03, SCALE-01, SCALE-02, SCALE-03

Mapped v1: 43/43
Orphaned v1: 0

---

*Created: 2026-04-11*
*Last updated: 2026-04-11*

# Requirements: Bookly

**Defined:** 2026-04-11
**Core Value:** Un professionnel beauty africain peut recevoir des réservations en ligne 24/7 et encaisser via mobile money (Airtel Money, Moov Money) sans jamais décrocher son téléphone.

---

## v1 Requirements

### Authentication (AUTH)

- [ ] **AUTH-01**: Le pro peut s'inscrire avec son numéro de téléphone et recevoir un OTP SMS
- [ ] **AUTH-02**: Le client peut s'inscrire avec son numéro de téléphone et recevoir un OTP SMS
- [ ] **AUTH-03**: L'utilisateur peut se reconnecter via OTP sans mot de passe
- [ ] **AUTH-04**: La session persiste après fermeture du navigateur (JWT Supabase)
- [ ] **AUTH-05**: L'accès est isolé par `salon_id` via RLS PostgreSQL — zéro fuite cross-tenant

### Onboarding Pro (ONBOARD)

- [ ] **ONBOARD-01**: Le pro peut créer son établissement (nom, adresse, téléphone, photo)
- [ ] **ONBOARD-02**: Le pro peut ajouter ses services avec nom, durée, prix FCFA
- [ ] **ONBOARD-03**: Le pro peut configurer ses horaires d'ouverture par jour de la semaine
- [ ] **ONBOARD-04**: Le pro peut ajouter des collaborateurs avec leurs services assignés
- [ ] **ONBOARD-05**: Le pro reçoit un lien partageable unique (ex: bookly.app/salon-nom) à partager sur WhatsApp

### Réservation en ligne (BOOKING)

- [ ] **BOOKING-01**: Le client peut accéder à la page publique du salon depuis un lien WhatsApp sans créer de compte
- [ ] **BOOKING-02**: Le client peut choisir un service, un collaborateur et un créneau disponible
- [ ] **BOOKING-03**: Les créneaux disponibles sont calculés en temps réel (fonction PostgreSQL `get_available_slots()`)
- [ ] **BOOKING-04**: Aucun double-booking n'est possible — contrainte `btree_gist` au niveau DB
- [ ] **BOOKING-05**: Le client reçoit un SMS de confirmation immédiate après réservation
- [ ] **BOOKING-06**: Le pro reçoit un SMS de notification à chaque nouvelle réservation
- [ ] **BOOKING-07**: Le client peut annuler son RDV via lien SMS (jusqu'à N heures avant)
- [ ] **BOOKING-08**: Un créneau est bloqué 10 minutes pendant le paiement en attente (`pending_payment`)

### Walk-ins (WALKIN)

- [ ] **WALKIN-01**: Le pro peut ajouter un client walk-in directement à l'agenda (sans flux de réservation)
- [ ] **WALKIN-02**: L'ajout d'un walk-in requiert uniquement le nom ou téléphone du client
- [ ] **WALKIN-03**: Les walk-ins apparaissent dans l'agenda avec un indicateur visuel distinct des RDV en ligne
- [ ] **WALKIN-04**: Le pro peut encaisser un walk-in immédiatement depuis l'agenda

### Agenda Pro (AGENDA)

- [ ] **AGENDA-01**: Le pro visualise son agenda en vue jour avec tous les RDV et walk-ins
- [ ] **AGENDA-02**: Le pro peut basculer en vue semaine
- [ ] **AGENDA-03**: Le pro peut créer manuellement un RDV pour un client existant ou nouveau
- [ ] **AGENDA-04**: Le pro peut modifier ou annuler un RDV depuis l'agenda
- [ ] **AGENDA-05**: L'agenda se synchronise en temps réel via Supabase Realtime (plusieurs onglets/appareils)
- [ ] **AGENDA-06**: L'agenda est accessible et utilisable sur mobile (responsive, touch-friendly)

### Paiements (PAYMENT)

- [ ] **PAYMENT-01**: Le client peut payer via Airtel Money (pawaPay) au moment de la réservation
- [ ] **PAYMENT-02**: Le pro peut encaisser en cash depuis l'agenda — enregistrement manuel avec montant
- [ ] **PAYMENT-03**: Le webhook pawaPay confirme ou annule le paiement de façon asynchrone
- [ ] **PAYMENT-04**: Un Edge Function Supabase dédié gère chaque webhook provider (isolé, traceable)
- [ ] **PAYMENT-05**: Tous les montants sont stockés en INTEGER (FCFA sans décimales)
- [ ] **PAYMENT-06**: L'historique des transactions est consultable par le pro

### Notifications (NOTIF)

- [ ] **NOTIF-01**: SMS de confirmation envoyé au client après réservation (Twilio)
- [ ] **NOTIF-02**: SMS de rappel envoyé au client 24h avant le RDV (Twilio)
- [ ] **NOTIF-03**: SMS de notification envoyé au pro à chaque nouvelle réservation (Twilio)
- [ ] **NOTIF-04**: Les notifications sont gérées via la table `notification_queue` (async, retry)
- [ ] **NOTIF-05**: Les SMS en échec sont réessayés automatiquement (max 3 tentatives)

### Dashboard Pro (DASHBOARD)

- [ ] **DASHBOARD-01**: Le pro voit son CA du jour en FCFA
- [ ] **DASHBOARD-02**: Le pro voit son CA du mois en FCFA
- [ ] **DASHBOARD-03**: Le pro voit le nombre de RDV du jour (confirmés / annulés / walk-ins)
- [ ] **DASHBOARD-04**: Le pro voit les RDV à venir dans les prochaines 24h

---

## v2 Requirements

### CRM & Fiches Clients (CRM)

- **CRM-01**: Le pro peut consulter la fiche complète d'un client (nom, téléphone, historique des visites)
- **CRM-02**: Le pro peut ajouter des notes sur un client
- **CRM-03**: Le client peut créer son compte pour accéder à son historique de réservations
- **CRM-04**: Le pro peut gérer une liste d'attente sur un créneau complet

### Point de Vente / Caisse (POS)

- **POS-01**: Le pro dispose d'une interface caisse complète (tablette/PC) avec panier multi-prestations
- **POS-02**: L'interface caisse accepte cash + mobile money en un seul encaissement
- **POS-03**: Le pro peut imprimer ou envoyer un reçu par SMS au client
- **POS-04**: Les frais d'annulation tardive et de no-show sont configurables par le pro

### Fidélité (FIDELITY)

- **FIDELITY-01**: Le client accumule des points de fidélité à chaque visite
- **FIDELITY-02**: Le client peut utiliser ses points en caisse
- **FIDELITY-03**: Le pro peut créer des cartes cadeaux numériques
- **FIDELITY-04**: Le client peut acheter et offrir une carte cadeau

### Communications Avancées (COMMS)

- **COMMS-01**: Les notifications de confirmation et rappel sont envoyées via WhatsApp (Phase 2)
- **COMMS-02**: Le pro peut envoyer une campagne SMS à tous ses clients
- **COMMS-03**: Le pro peut envoyer une campagne SMS ciblée (filtre par dernière visite, service)
- **COMMS-04**: Le pro peut configurer des campagnes SMS automatisées (rappel de revisite)

### Acompte & Prépaiement (DEPOSIT)

- **DEPOSIT-01**: Le pro peut exiger un acompte à la réservation (% configurable)
- **DEPOSIT-02**: Le pro peut exiger le prépaiement intégral pour certains services

### PWA Offline (PWA)

- **PWA-01**: L'agenda est consultable sans connexion internet (cache local)
- **PWA-02**: Les walk-ins peuvent être enregistrés offline et synchronisés à la reconnexion
- **PWA-03**: L'application est installable sur écran d'accueil (mobile + desktop)

---

## v3+ Requirements (Phase 3–5)

### Analytics Avancés (ANALYTICS)

- **ANALYTICS-01**: Dashboard taux d'occupation (%), top services, top collaborateurs
- **ANALYTICS-02**: Évolution CA semaine/mois/trimestre avec graphiques
- **ANALYTICS-03**: Export comptable (CSV des transactions)

### Gestion Équipe (TEAM)

- **TEAM-01**: Le pro peut créer des plannings de shifts pour ses collaborateurs
- **TEAM-02**: Le pro peut saisir les absences et congés
- **TEAM-03**: Le pro peut voir la performance de chaque collaborateur (CA, RDV)

### Bot WhatsApp & Marketplace (GROWTH)

- **GROWTH-01**: Bot WhatsApp — le client peut réserver directement depuis WhatsApp sans navigateur
- **GROWTH-02**: Payout Airtel Money — le pro reçoit ses fonds sur son wallet mobile (Phase 3)
- **GROWTH-03**: Service à domicile — un professionnel peut gérer des interventions géolocalisées

### Marketplace & App Mobile (SCALE)

- **SCALE-01**: Marketplace publique — les clients peuvent découvrir les salons par géolocalisation
- **SCALE-02**: App mobile client iOS et Android (Expo React Native) avec push notifications
- **SCALE-03**: Widget de réservation intégrable sur site web ou lien Instagram/Facebook

### Avancé (ADVANCED)

- **ADVANCED-01**: Gestion des stocks de produits avec alertes de rupture
- **ADVANCED-02**: Packages / Cures (N séances prépayées avec date d'expiration)
- **ADVANCED-03**: Multi-établissements — gérant pilote plusieurs salons depuis un compte

---

## Out of Scope

| Feature | Raison |
|---------|--------|
| Certification NF525 (caisse fiscale française) | Réglementation française — inapplicable au Gabon / en Afrique |
| Auth email/password comme méthode principale | Email non fiable comme identité en Afrique — OTP téléphone uniquement en P0 |
| Carte bancaire comme rail de paiement principal | Pénétration <10% au Gabon — mobile money + cash en premier |
| Google Calendar sync | Pas d'usage identifié sur le marché cible |
| Analytics de cohortes / rétention Phase 1 | Les pros veulent CA + nombre de RDV — pas besoin de complexité Phase 1 |
| IA réception téléphonique | Infrastructure insuffisante, trop tôt pour le marché |
| Multi-devises | FCFA uniquement pour le lancement Gabon |
| Multi-langues (EN, AR) | Français uniquement pour le lancement |
| App mobile pro native | Webapp responsive suffisante pour les pros en v1–v3 |
| SEO marketplace Phase 1 | Pas de densité suffisante de salons pour générer du trafic organique |
| Desktop-first design | Mobile-first obligatoire — majorité d'utilisateurs smartphone only |
| Tap to Pay NFC | Mobile money = sans contact natif — NFC inutile |
| Expansion multi-pays | Phase 1 Gabon uniquement, internationalisation Phase 4+ |

---

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 1 | Pending |
| AUTH-02 | Phase 1 | Pending |
| AUTH-03 | Phase 1 | Pending |
| AUTH-04 | Phase 1 | Pending |
| AUTH-05 | Phase 1 | Pending |
| ONBOARD-01 | Phase 1 | Pending |
| ONBOARD-02 | Phase 1 | Pending |
| ONBOARD-03 | Phase 1 | Pending |
| ONBOARD-04 | Phase 1 | Pending |
| ONBOARD-05 | Phase 1 | Pending |
| BOOKING-01 | Phase 1 | Pending |
| BOOKING-02 | Phase 1 | Pending |
| BOOKING-03 | Phase 1 | Pending |
| BOOKING-04 | Phase 1 | Pending |
| BOOKING-05 | Phase 1 | Pending |
| BOOKING-06 | Phase 1 | Pending |
| BOOKING-07 | Phase 1 | Pending |
| BOOKING-08 | Phase 1 | Pending |
| WALKIN-01 | Phase 1 | Pending |
| WALKIN-02 | Phase 1 | Pending |
| WALKIN-03 | Phase 1 | Pending |
| WALKIN-04 | Phase 1 | Pending |
| AGENDA-01 | Phase 1 | Pending |
| AGENDA-02 | Phase 1 | Pending |
| AGENDA-03 | Phase 1 | Pending |
| AGENDA-04 | Phase 1 | Pending |
| AGENDA-05 | Phase 1 | Pending |
| AGENDA-06 | Phase 1 | Pending |
| PAYMENT-01 | Phase 1 | Pending |
| PAYMENT-02 | Phase 1 | Pending |
| PAYMENT-03 | Phase 1 | Pending |
| PAYMENT-04 | Phase 1 | Pending |
| PAYMENT-05 | Phase 1 | Pending |
| PAYMENT-06 | Phase 1 | Pending |
| NOTIF-01 | Phase 1 | Pending |
| NOTIF-02 | Phase 1 | Pending |
| NOTIF-03 | Phase 1 | Pending |
| NOTIF-04 | Phase 1 | Pending |
| NOTIF-05 | Phase 1 | Pending |
| DASHBOARD-01 | Phase 1 | Pending |
| DASHBOARD-02 | Phase 1 | Pending |
| DASHBOARD-03 | Phase 1 | Pending |
| DASHBOARD-04 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 43 total
- Mapped to phases: 43
- Unmapped: 0 ✓

---
*Requirements defined: 2026-04-11*
*Last updated: 2026-04-11 after research synthesis*

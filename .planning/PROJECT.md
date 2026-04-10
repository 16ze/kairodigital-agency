# Bookly

## What This Is

Bookly est une plateforme SaaS de gestion de rendez-vous et d'encaissement destinée aux professionnels de la beauté en Afrique francophone (salons de coiffure, instituts de beauté, nail art, barbershops). Le produit comprend un dashboard web pro, une application mobile client (iOS + Android via Expo React Native), et une marketplace publique (Phase 4). Lancement à Libreville (Gabon), scalable sur tout le continent.

## Core Value

Un professionnel beauty africain peut recevoir des réservations en ligne 24/7 et encaisser via mobile money (Airtel Money, Moov Money) sans jamais décrocher son téléphone.

## Requirements

### Validated

(Aucun — livrer pour valider)

### Active

**Phase 1 — MVP Core Booking**
- [ ] Le pro peut créer un compte et configurer son établissement (nom, services, horaires, collaborateurs)
- [ ] Le client peut rechercher un salon et réserver un créneau en ligne
- [ ] Le pro reçoit une notification SMS à chaque nouvelle réservation
- [ ] Le client reçoit un SMS de confirmation et un rappel 24h avant le RDV
- [ ] Le pro peut voir et gérer son agenda (vue jour/semaine)
- [ ] Le client peut annuler ou déplacer son RDV
- [ ] Le pro peut encaisser via Airtel Money et Moov Money
- [ ] Le pro peut encaisser par carte bancaire (Stripe)
- [ ] Le pro peut voir son CA du jour et du mois

**Phase 2 — CRM + Caisse + Fidélité**
- [ ] Le pro dispose d'une fiche client complète avec historique des visites
- [ ] Le pro peut encaisser depuis une interface caisse (POS) sur tablette ou PC
- [ ] Le client peut acheter et utiliser des cartes cadeaux
- [ ] Le client accumule des points de fidélité utilisables en caisse
- [ ] Le client peut rejoindre une liste d'attente sur un créneau complet
- [ ] Le pro peut demander un acompte ou un prépaiement à la réservation
- [ ] Les frais d'annulation tardive et de no-show sont configurables

**Phase 3 — Analytics + Marketing + Équipe**
- [ ] Le pro dispose d'un dashboard analytics (CA, occupation, top services, top collaborateurs)
- [ ] Le pro peut envoyer des campagnes SMS ciblées (ponctuelles et automatisées)
- [ ] Le pro peut gérer les plannings et absences de ses collaborateurs (shifts, pointage)
- [ ] Le pro peut exporter ses données comptables

**Phase 4 — Marketplace + App Mobile**
- [ ] Les clients peuvent découvrir des salons via une marketplace publique (recherche + géolocalisation)
- [ ] L'application mobile client est disponible sur iOS et Android (Expo React Native)
- [ ] Le pro peut intégrer un widget de réservation sur son site web ou réseaux sociaux

**Phase 5 — Stocks + Packages + Multi-établissements**
- [ ] Le pro peut gérer ses stocks de produits avec alertes de rupture
- [ ] Le pro peut créer des packages / cures (N séances avec validité)
- [ ] Un gérant peut piloter plusieurs établissements depuis un seul compte

### Out of Scope

- Certification NF525 — réglementation française non applicable en Afrique
- IA réception téléphonique — roadmap long terme, post-v1
- App mobile pro native — webapp responsive suffisante pour les pros en v1
- Multi-langues (anglais, arabe) — lancement en français uniquement
- Intégration Google Calendar — pas prioritaire sur le marché cible
- Expansion multi-pays — Phase 1 Gabon uniquement, internationalisation Phase 4+

## Context

- **Marché cible :** TPE beauty en Afrique francophone — coiffeurs, barbiers, esthéticiennes, nail artists
- **Lancement :** Libreville, Gabon — 5 salons pilotes pour valider le MVP
- **Devise :** FCFA (XAF) — affichage et facturation en francs CFA
- **Langue :** Français (interface + communications SMS/email)
- **SMS :** Africa's Talking API — couverture Afrique subsaharienne, prix compétitifs
- **Paiements mobile money :** Airtel Money + Moov Money — pénétration élevée vs carte bancaire en Afrique centrale
- **Différence clé vs Planity :** Pas de marketplace au lancement (Phase 4) — les pros apportent leurs propres clients au départ
- **Concurrents locaux :** Aucun concurrent direct identifié sur le marché gabonais / CEMAC
- **Référence produit :** Planity (France) — couverture fonctionnelle cible ~97%, design différent, aucune référence dans le code

## Constraints

- **Stack :** Next.js 14 App Router, TypeScript strict, Tailwind CSS, shadcn/ui, Supabase (PostgreSQL + Auth + Realtime), Prisma, Stripe, Expo React Native, Africa's Talking, Vercel — non négociable
- **Convention :** Pattern Result `{ data, error }`, Zod sur toutes les entrées, RLS activé sur toutes les tables, Server Components par défaut
- **Paiements :** Intégration mobile money via API opérateurs (Airtel Money API, Moov Money API) ou agrégateur (CinetPay, FedaPay) — à valider en Phase 1
- **Devise :** FCFA — pas de décimales, arrondi à l'entier
- **Déploiement :** Vercel (frontend + API routes) + Supabase cloud
- **Mobile :** App client uniquement en React Native (Expo) — pas d'app pro native en v1

## Key Decisions

| Décision | Rationale | Outcome |
|----------|-----------|---------|
| Africa's Talking pour les SMS | Couverture Afrique, prix compétitif vs Twilio, intégration simple | — Pending |
| CinetPay ou FedaPay pour mobile money | Agrégateurs qui unifient Airtel + Moov en une seule API — à tester | — Pending |
| Marketplace en Phase 4, pas Phase 1 | Valider le produit pro d'abord avec des clients existants, construire la marketplace quand l'offre est dense | — Pending |
| App mobile client (Expo RN) dès Phase 4 | Web responsive suffit pour Phase 1 ; native apporte la valeur (push notif, offline) en Phase 4 | — Pending |
| FCFA sans décimales | La devise locale n'utilise pas de centimes — simplification UI et calculs | — Pending |

## Pricing

| Plan | Prix/mois | Cible |
|------|-----------|-------|
| Starter | 9 900 FCFA | Indépendant (1 personne) |
| Pro | 19 900 FCFA | Salon 2-5 collaborateurs |
| Business | 34 900 FCFA | Multi-collaborateurs + analytics |

- Essai gratuit 3 mois sans CB
- Zéro commission sur les réservations
- SMS supplémentaires : facturation à l'usage via Africa's Talking

---
*Last updated: 2026-04-10 after initialization*

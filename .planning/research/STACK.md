# Stack Research: Bookly

**Project:** Bookly — SaaS booking beauté, Gabon / Afrique francophone
**Date:** 2026-04-10
**Confidence:** MEDIUM-HIGH

---

## ⚠️ Corrections critiques vs PROJECT.md initial

| Hypothèse initiale | Réalité vérifiée |
|---|---|
| Africa's Talking pour SMS | ❌ **Ne couvre PAS le Gabon** — utiliser Twilio |
| CinetPay pour mobile money | ❌ **Ne couvre PAS le Gabon** (9 pays : CI, SN, CM, ML, BF, TG, CG, GN, BJ) |
| FedaPay | ❌ Pas de couverture Gabon confirmée |
| Flutterwave | ⚠️ Licence CEMAC obtenue pour Cameroun seulement (juin 2025) |
| Next.js 14 | ⚠️ Outdated — utiliser **Next.js 15** (LTS stable, React 19, Turbopack stable) |
| Stripe natif Gabon | ❌ Stripe n'opère pas au Gabon directement — utiliser via entité française (Kairo Digital) |

---

## Recommended Stack

### Frontend Web

| Lib | Version | Rationale |
|---|---|---|
| **Next.js** | 15.x (LTS) | App Router, React 19, Turbopack stable. Upgrade depuis 14 recommandé. |
| **React** | 19.x | Inclus avec Next.js 15 |
| **TypeScript** | 5.x strict | Non négociable |
| **Tailwind CSS** | 3.4.x | Stable, bien supporté |
| **shadcn/ui** | Latest | Composants accessibles basés sur Radix UI |
| **Zustand** | 5.x | State client léger pour agenda temps réel |
| **React Hook Form** | 7.x | Formulaires + validation Zod |
| **Zod** | 3.x | Validation schémas côté client et serveur |

### Backend / Database

| Lib | Version | Rationale |
|---|---|---|
| **Supabase** | Cloud (hosted) | PostgreSQL + Auth + Realtime + Storage. Region : **London (eu-west-2)** — plus proche Afrique (~120-150ms depuis Libreville). |
| **Prisma** | 5.x | ORM type-safe. Utiliser Prisma pour les migrations + Supabase client pour Realtime. |
| **Server Actions** | Next.js 15 natif | Mutations server-side, pas d'API routes séparées |
| **Supabase RLS** | Obligatoire | Isolation multi-tenant par salon. Toutes les tables. |
| **Supabase Realtime** | Natif | Channels pour sync agenda en temps réel |

### Mobile (Client App — Phase 4)

| Lib | Version | Rationale |
|---|---|---|
| **Expo** | SDK 53 | React Native géré, OTA updates, EAS Build |
| **Expo Router** | 4.x | Navigation file-based (aligné Next.js) |
| **React Native Reanimated** | 3.x | Animations fluides agenda |
| **Expo Notifications** | SDK 53 | Push notifications iOS + Android |
| **MMKV** | Latest | Storage local rapide (vs AsyncStorage) |

### Paiements

#### Mobile Money — Gabon

| Provider | Couverture Gabon | Recommandation |
|---|---|---|
| **pawaPay** | ✅ Airtel Money Gabon (confirmé live) | **PRIMARY** — intégrer Phase 1 |
| **Moneroo** | ✅ Airtel + Moov Gabon (listé) | **FALLBACK** — valider Phase 1 |
| CinetPay | ❌ Non disponible Gabon | Exclure |
| FedaPay | ❌ Non confirmé Gabon | Exclure |
| Flutterwave | ⚠️ CEMAC en cours | Surveiller |

**Stratégie recommandée :** Intégrer pawaPay en priorité (Airtel Money). Ajouter Moneroo comme fallback pour Moov Money. Valider les deux lors de l'onboarding Phase 1 — risque le plus élevé du projet.

#### Carte bancaire

| Provider | Status | Notes |
|---|---|---|
| **Stripe** | ✅ Via entité française (Kairo Digital) | XAF = devise zéro décimal, supportée nativement. Paiements reçus sur compte français. |

**Note :** Mobile money = rail primaire au Gabon. Stripe = secondaire pour clients avec CB internationale.

### SMS / Communications

| Phase | Outil | Rationale |
|---|---|---|
| **Phase 1-3** | **Twilio** | ✅ Couverture Gabon confirmée. ~0,20$/SMS (0,18-0,26$ selon opérateur). |
| **Phase 2+** | **WhatsApp Business API** | Coût 60-70% inférieur au SMS. Essentiel à l'échelle. Meta Business Manager requis. |
| Emails | **Resend + React Email** | Transactionnel (confirmation, rappels, factures) |

**Coût SMS Phase 1 estimé :** ~0,60$/réservation (confirmation + rappel 24h + post-RDV). Budget mensuel avec 500 RDV/mois : ~300$/mois. → WhatsApp devient critique dès Phase 2.

### Infrastructure

| Composant | Outil | Notes |
|---|---|---|
| **Hosting** | Vercel | Edge functions, déploiement continu |
| **Database** | Supabase Cloud (London) | Pas de région Afrique disponible |
| **Storage** | Supabase Storage | Photos salons, avatars |
| **Monitoring** | Vercel Analytics + Sentry | |
| **CI/CD** | GitHub Actions + Vercel | Deploy preview sur chaque PR |

---

## What NOT to Use

| Technologie | Raison |
|---|---|
| Africa's Talking | ❌ Ne couvre pas le Gabon |
| CinetPay / FedaPay | ❌ Pas de couverture Gabon confirmée |
| tRPC | Complexité inutile avec Server Actions |
| Prisma Accelerate | Coût additionnel injustifié en Phase 1 |
| Redis / Upstash | Pas nécessaire en Phase 1, ajouter si perf problème |
| Next.js 14 | Outdated, utiliser 15 |

---

## Africa-Specific Considerations

### Connectivité

- **Latence Supabase → Libreville :** ~120-150ms (London). Utiliser optimistic UI côté agenda pour masquer la latence.
- **Connectivité instable :** Prévoir des états de chargement clairs, retry logic sur les mutations critiques (paiement, réservation).
- **Mobile-first :** La majorité des utilisateurs en Afrique = smartphone uniquement. Interface responsive critique.

### Paiements mobile money

- **Webhooks async :** Les confirmations Airtel/Moov peuvent prendre 10-30 secondes. Afficher un état "En attente de confirmation" et mettre à jour via webhook.
- **Numéros locaux :** Format gabonais = +241 XX XX XX XX. Validation de numéros de téléphone adaptée.
- **Onboarding pawaPay :** Processus KYB (Know Your Business) requis. Prévoir 2-4 semaines. Démarrer immédiatement en Phase 1.

### FCFA (XAF)

- Devise zéro-décimal (pas de centimes). Stripe supporte XAF nativement.
- Tous les montants en entiers. Pas de `toFixed(2)` dans l'UI.
- Format d'affichage : `9 900 FCFA` (espace comme séparateur de milliers, pas de virgule).

### SMS

- Twilio Gabon : ~0,20$/SMS. Prévoir budget SMS dans le pricing (inclus dans l'abonnement ou facturation à l'usage).
- WhatsApp Business API : Soumettre la demande Meta dès Phase 1 (délai d'approbation 4-8 semaines).

---

## Confidence Levels

| Domaine | Niveau | Raison |
|---|---|---|
| Stack frontend (Next.js 15 + Tailwind + shadcn) | HIGH | Versions vérifiées, patterns éprouvés |
| Backend (Supabase + Prisma) | HIGH | Compatibilité vérifiée, patterns SaaS documentés |
| Paiements (pawaPay) | MEDIUM-HIGH | Live Gabon confirmé (Airtel). Moov Money à valider. |
| Stripe via Kairo Digital | HIGH | XAF confirmé, entité française viable |
| SMS (Twilio) | MEDIUM-HIGH | Couverture Gabon confirmée, tarifs vérifiés |
| Mobile (Expo SDK 53) | HIGH | Version stable vérifiée |
| Latence London → Libreville | MEDIUM | Estimations théoriques, test terrain requis |

---

## Open Questions (à valider Phase 1)

1. **pawaPay Moov Money Gabon** — Airtel confirmé, Moov à valider lors de l'onboarding
2. **Twilio tarif exact Gabon** — fourchette 0,18-0,26$/SMS selon opérateur
3. **Moneroo maturité API** — moins battle-tested que pawaPay, tester en sandbox
4. **Latence réelle Libreville → Supabase London** — mesurer avec Lighthouse depuis Gabon
5. **WhatsApp Business API délai approbation** — soumettre dès Phase 1
6. **pawaPay KYB timeline** — démarrer le processus en semaine 1

---

*Research date: 2026-04-10*

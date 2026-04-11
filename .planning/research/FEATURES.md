# Features Research: Bookly

**Project:** Bookly — SaaS booking beauté, Gabon / Afrique francophone
**Date:** 2026-04-10
**Confidence:** MEDIUM-HIGH

---

## ⚠️ Corrections critiques vs PROJECT.md initial

| Hypothèse initiale | Réalité marché |
|---|---|
| Auth email/password | ❌ → **OTP téléphone** — l'identité = numéro de téléphone en Afrique |
| Planity = concurrent | ❌ → **Le vrai concurrent = WhatsApp + cahier papier** |
| Marketplace Phase 4 correcte | ✅ Confirmé — pas de densité suffisante au départ |
| Walk-ins pas mentionnés | ❌ → **40-60% walk-ins dans les salons africains** — à intégrer Phase 1 |
| Email comme canal comm | ❌ → **WhatsApp est LE canal**, SMS pour transactionnel |
| Offline hors scope | ❌ → **PWA offline = avantage concurrentiel** (pannes fréquentes, data cher) |

---

## Concurrents identifiés

| Concurrent | Type | Gabon ? | Verdict |
|---|---|---|---|
| **WhatsApp + cahier papier** | Non-logiciel | ✅ | **Vrai concurrent** — ce que les salons font aujourd'hui |
| Zenaba (CI) | Marketplace stylistes à domicile | ❌ | Pas concurrent direct |
| Fresha | SaaS booking (UK) | ❌ | Interface anglaise, pas de mobile money |
| Booksy | SaaS booking (US/EU) | ❌ | Pas adapté Afrique |
| Wala | POS Afrique | Partiel | POS uniquement, pas de booking |
| **Planity** | SaaS booking (France) | ❌ | Référence fonctionnelle, pas concurrent |

**Conclusion : Whitespace réel.** Aucun concurrent direct sur le marché gabonais / CEMAC pour la gestion de salon beauté avec mobile money intégré.

---

## 1. Table Stakes — Ce qui doit exister ou les pros partent

### Réservation & Agenda

| Feature | Priorité | Note Afrique |
|---|---|---|
| Réservation en ligne 24/7 | P0 | Lien partageable sur WhatsApp = canal principal |
| Agenda pro (vue jour/semaine) | P0 | Interface tactile tablette |
| **Gestion des walk-ins** | **P0** | ⚠️ 40-60% des clients arrivent sans RDV — ABSENT du PROJECT.md initial |
| Création manuelle de RDV par le pro | P0 | |
| Annulation / déplacement | P0 | |
| Confirmation immédiate | P0 | Via SMS (pas email) |

### Authentification

| Feature | Priorité | Note Afrique |
|---|---|---|
| **Auth OTP par téléphone** | **P0** | ⚠️ Identité = numéro de téléphone. Email non fiable. Supabase Phone Auth. |
| Auth email/password | P1 | Secondaire, pour les pros tech-savvy |
| Session persistante | P0 | |

### Paiements

| Feature | Priorité | Note Afrique |
|---|---|---|
| **Airtel Money** | **P0** | Via pawaPay — rail primaire Gabon |
| **Moov Money** | **P0** | Via pawaPay ou Moneroo |
| **Encaissement cash** | **P0** | ⚠️ 30-50% des transactions. Suivi manuel + enregistrement obligatoire. |
| Carte bancaire (Stripe) | P1 | Secondaire, <10% pénétration |
| Webhook paiement async | P0 | Confirmation mobile money peut prendre 10-30s |

### Communications

| Feature | Priorité | Note Afrique |
|---|---|---|
| **SMS confirmation + rappel** | **P0** | Via Twilio (Africa's Talking ❌ Gabon) |
| **WhatsApp lien de réservation** | **P0** | Partage du lien sur WhatsApp = acquisition principale |
| **WhatsApp notifications** | **P1** | Dès Phase 2 — canal préféré des clients |
| Email | P2 | Faiblement utilisé |

### Gestion pro basique

| Feature | Priorité | Note Afrique |
|---|---|---|
| Fiche client (nom, téléphone) | P0 | Téléphone = identifiant principal |
| Services et tarifs (FCFA) | P0 | Pas de décimales |
| Horaires d'ouverture | P0 | |
| Gestion collaborateurs | P0 | |

---

## 2. Différenciants — Avantage concurrentiel en Afrique

| Feature | Valeur | Phase |
|---|---|---|
| **POS cash + mobile money intégré** | Le seul outil qui gère les 3 modes de paiement du marché | 2 |
| **Mode PWA / offline** | Pannes fréquentes, data cher → agenda consulable sans internet | 2 |
| **Carte de fidélité digitale (punch card)** | Simple, visuel, compréhensible sans explication | 2 |
| **Bot WhatsApp de réservation** | Le client réserve sans app ni browser, depuis WhatsApp | 3 |
| **Payout instantané sur mobile wallet** | Le pro reçoit son argent sur Airtel Money le jour même | 3 |
| **Service à domicile** | Stylistes indépendants qui se déplacent — marché significant | 3 |
| **Interface ultra-simple** | Pros souvent peu à l'aise avec le digital — onboarding < 10 min | P0 |
| **Support en français gabonais** | Terminologie locale (« faire les cheveux », « défrisage », etc.) | P0 |

---

## 3. Anti-Features — Ce qu'on ne construit PAS pour ce marché

| Feature | Raison |
|---|---|
| Certification NF525 | Réglementation française — inapplicable en Afrique |
| Auth email-first | L'email n'est pas l'identité en Afrique — OTP téléphone |
| Paiement CB en premier | <10% pénétration Gabon — mobile money d'abord |
| Google Calendar sync | Pas d'usage identifié sur le marché cible |
| Analytics complexes Phase 1 | Pros veulent CA + nombre de RDV — pas de cohortes |
| Rapport Z / conformité fiscale FR | Cadre légal différent (à adapter localement) |
| IA réception téléphonique | Trop tôt, infrastructure insuffisante |
| Multi-devises | FCFA uniquement en v1 |
| SEO marketplace | Pas de densité de salons suffisante pour Phase 1 |
| Desktop-first design | Mobile-first obligatoire — majorité = smartphone only |

---

## 4. Adaptations Planity → Bookly Afrique

| Feature Planity | Adaptation Bookly |
|---|---|
| Marketplace publique | Phase 4 — WhatsApp sharing link en attendant |
| Auth email/password | OTP téléphone en priorité |
| Stripe seul | pawaPay (mobile money) + Stripe secondaire |
| SMS via Twilio FR | Twilio (Africa's Talking ❌ Gabon) |
| NF525 caisse | Caisse simple conforme local (pas NF525) |
| Email comme canal | SMS + WhatsApp comme canaux principaux |
| Agenda "réservations only" | **Agenda walk-ins intégré** — ajout obligatoire |
| Prix en € avec décimales | Prix FCFA entiers, format `9 900 FCFA` |
| Avis Google / Meditrust | Avis internes Bookly + partage WhatsApp |
| Tap to Pay (iPhone) | Pas pertinent — mobile money = sans contact natif |

---

## 5. Recommandation MVP Phase 1 (10 features clés)

1. Auth OTP téléphone (Supabase Phone Auth)
2. Onboarding pro wizard (services + horaires + collaborateurs)
3. Page publique salon + lien réservation partageable WhatsApp
4. Tunnel de réservation client (service → créneau → confirmation SMS)
5. Gestion des walk-ins (ajout direct à l'agenda sans flux de réservation)
6. Agenda pro (vue jour/semaine) avec RDV + walk-ins
7. Encaissement mobile money (Airtel + Moov via pawaPay)
8. Encaissement cash (enregistrement manuel)
9. Rappels SMS automatiques 24h avant
10. Dashboard CA basique (jour/mois, FCFA)

**Features à déférer (hors Phase 1) :**
- Fiche client complète / CRM → Phase 2
- Carte de fidélité → Phase 2
- WhatsApp notifications → Phase 2
- Mode PWA offline → Phase 2
- POS caisse complet → Phase 2
- App mobile native → Phase 4

---

## 6. Graphe de dépendances

```
Auth OTP
  └── Onboarding pro
        ├── Gestion services + tarifs
        ├── Gestion collaborateurs
        └── Horaires d'ouverture
              └── Agenda pro
                    ├── Walk-ins (ajout direct)
                    ├── Réservation en ligne
                    │     ├── Page publique salon
                    │     ├── Tunnel réservation
                    │     └── Confirmation SMS
                    └── Encaissement
                          ├── Mobile money (pawaPay)
                          └── Cash (manuel)
                                └── Dashboard CA
```

---

## 7. Contexte marché

### Réalité des paiements au Gabon
- Mobile money : ~60% des transactions (Airtel Money dominant)
- Cash : ~30-50% des transactions
- Carte bancaire : <10% de pénétration
- → Le POS doit traiter **cash + mobile money en premier**, carte en option

### Réalité connectivité
- 3G dominant, 4G disponible Libreville mais cher
- Coût data : ~2,4% du revenu mensuel moyen pour 1GB
- Pannes électriques fréquentes (jusqu'à 31% de perte CA pour les PME)
- → Mode offline (PWA) = avantage concurrentiel réel

### Réalité culturelle
- Identité = numéro de téléphone (pas email)
- Canal comm préféré = WhatsApp (pas SMS, pas email)
- Confiance = bouche à oreille (pas Google Reviews)
- Walk-ins = norme, pas l'exception

---

*Research date: 2026-04-10*

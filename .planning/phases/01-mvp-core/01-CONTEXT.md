# Phase 1: MVP Core - Context

**Gathered:** 2026-04-12
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 1 livre le cœur opérationnel de Bookly :
- Auth OTP téléphone (pro + client)
- Onboarding pro (salon, services, horaires, collaborateurs)
- Découverte publique par recherche texte (nom/service + ville) — **NOTE: scope élargi vs ROADMAP initial**
- Page publique du salon (SEO-ready)
- Flux de réservation client (3 étapes + OTP inline + multi-services)
- Paiements pawaPay (Airtel Money) + cash + acompte configurable par service
- Agenda pro (vue jour/semaine, Realtime, responsive mobile+desktop)
- Walk-ins depuis l'agenda (tap sur créneau vide)
- Système d'avis clients (SMS post-RDV, étoiles 1-5 + commentaire)
- Notifications SMS Twilio (confirmation, rappel 24h+adresse, notification pro)
- Dashboard CA (KPIs + historique transactions)
- Espace client (/mon-compte) : voir ses RDV + annuler
- Collaborateurs avec leur propre login OTP (géré par le pro)
- Structure de plans (Plan Agenda / Plan Pro) avec quota SMS tracé

**Hors scope Phase 1 :** Stripe, stocks, cartes cadeaux, export comptable, PWA offline, WhatsApp, campagnes SMS, CRM avancé, app mobile native, widget iframe externe, remboursements automatiques pawaPay (aucun refund — Phase 2), sections marketing homepage (témoignages, tarifs, "comment ça marche").

</domain>

<decisions>
## Implementation Decisions

### Flux réservation client (page publique)

- Navigation en **3 écrans séparés** : Écran 1 → choisir service(s). Écran 2 → choisir collaborateur (+ option "Premier disponible"). Écran 3 → saisie infos client + OTP.
- **Multi-services** : le client peut ajouter plusieurs services à la suite (comme Planity).
- Chaque service affiche : **nom + durée + prix FCFA + description courte** (1 ligne, optionnelle).
- Option "Premier disponible" en plus du choix collaborateur. Étape collaborateur **masquée si le salon a 1 seul collaborateur** (sauter l'étape).
- **Compte client créé silencieusement** : à l'étape 3, le client saisit téléphone + email → reçoit OTP → compte créé. L'espace client `/mon-compte` est accessible ensuite.
- Le client peut **voir + annuler** ses RDV depuis `/mon-compte`. Phase 1 : annulation uniquement (pas de replanification directe — annuler + re-réserver).
- Si le service ne requiert **pas de paiement en ligne** : confirmation immédiate après soumission infos + OTP. Page de confirmation avec résumé (salon, service, date/heure, prix).
- Si paiement en ligne requis : flux pawaPay. Si paiement échoue ou expire (10 min) → **page d'échec + bouton "Retry"**.
- **Acompte** : montant affiché uniquement à l'étape paiement (pas sur la carte service).
- **Remboursement** : pas de remboursement automatique en Phase 1. Si annulation dans le délai → remboursement manuel par le pro.

### Recherche publique (scope élargi Phase 1)

- Barre de recherche sur bookly.app : le client tape **nom du salon** ou **service + ville** (ex: "barber libreville").
- Résultats : liste de salons correspondants avec page bookable.
- Pas de géolocalisation GPS en Phase 1 (texte uniquement).
- **Référence Planity** : UX de la page salon et du flux booking identique à Planity (voir planity.com).

### Onboarding pro

- Wizard **linéaire à 4 étapes** avec barre de progression : 1 Profil salon → 2 Services → 3 Horaires → 4 Équipe (optionnel).
- **Lien disponible dès la fin de l'étape 1** (slug créé au signup). Page publique visible uniquement quand salon + ≥1 service + horaires configurés (page "en cours de configuration" sinon).
- Horaires : **semaine type récurrente** par jour (Planity-style). Chaque jour : heure ouverture/fermeture + pause déjeuner optionnelle. Plusieurs plages horaires par jour supportées. Chaque collaborateur peut avoir ses propres horaires.
- Après onboarding : redirection vers **agenda vue jour + bandeau "Votre salon est prêt — partagez votre lien"**.
- **Pas de landing page marketing** en Phase 1 pour les pros. Signup pro direct : bookly.app/pro/signup.

### Collaborateurs (Plan Pro)

- Le propriétaire crée les collaborateurs dans **Paramètres > Équipe** : saisit le téléphone du collaborateur → le collaborateur reçoit OTP pour se connecter.
- Le propriétaire peut activer/désactiver l'accès d'un collaborateur.
- Accès collaborateur : **Agenda uniquement** par défaut. Le propriétaire peut configurer des droits supplémentaires par collaborateur (permissions configurables).
- Collaborateur n'a PAS accès à : Dashboard CA, Paramètres globaux, gestion des services/prix par défaut.
- Menu principal collaborateur : **Agenda** uniquement (le reste selon permissions pro).

### Agenda pro

- Vue par défaut : **Planity-style** (à vérifier sur planity.com — probablement vue semaine desktop / vue jour mobile).
- Création walk-in : **tap sur créneau vide → modal rapide** (nom/téléphone + service + durée). 30 secondes max.
- Distinction visuelle walk-in vs RDV en ligne : **Planity-style** (couleur distincte + indicateur visuel).
- Slots `pending_payment` : visibles avec statut "En attente de paiement" **uniquement si le service a un paiement en ligne configuré**. Sinon invisible.
- Bouton **"Encaisser"** sur chaque RDV/walk-in → choix Cash ou Airtel Money (si actif) → confirmation mise à jour CA.
- Agenda synchronisé en **temps réel** via Supabase Realtime (multi-onglets, multi-appareils).

### Paiements (configuration par service)

- Configuration **par service** (pas globale) : chaque service peut avoir :
  - Paiement sur place (aucun paiement en ligne requis)
  - Paiement intégral en ligne (pawaPay)
  - Acompte % configurable (ex: 30% à la réservation, solde sur place)
- Interface de config dans la section **Services** de l'app pro (champ par service).
- **Pas de Stripe en Phase 1** — reporté Phase 2.

### Annulation et remboursement

- Délai d'annulation : **configurable par le pro** (ex: 24h avant = annulation libre, après = pas de remboursement). Valeur par défaut : 24h.
- Si annulation dans le délai → remboursement **manuel** par le pro (pas de remboursement automatique pawaPay Phase 1).
- Client annule depuis `/mon-compte` bouton "Annuler ce RDV".

### Dashboard CA

- **Page de démarrage** après connexion pro : **Agenda vue jour** (pas le dashboard).
- Navigation : Agenda → Dashboard → Services → Clients → Paramètres.
- Dashboard : **4 KPIs uniquement** (pas de graphiques) : CA jour, CA mois, RDV du jour (confirmés/annulés/walk-ins), RDV à venir 24h.
- CA = **tous paiements confirmés** (Airtel + Cash), **exclut les annulations**.
- Section **Historique transactions** sous les KPIs : date, client, service, montant, méthode.
- Pas de graphiques en Phase 1 (Phase 3 Analytics).

### Clients (liste basique)

- Section **Clients** dans la nav principale : liste avec nom, téléphone, nombre de RDV passés.
- Pas de fiche CRM complète en Phase 1 (Phase 2).

### Notifications SMS

- **SMS confirmation client** : "Votre RDV chez [Salon] : [Service(s)] le [Jour] [Date] à [Heure]. Annuler : bookly.app/cancel/[token]"
- **SMS rappel 24h** : "Rappel : RDV chez [Salon] demain à [Heure] ([Service]). Adresse : [Adresse salon]. Annuler : bookly.app/cancel/[token]"
- **SMS notification pro** : "Nouveau RDV 📍 [Prénom Client] [Nom] : [Service] le [Date] à [Heure]. Voir agenda : bookly.app/agenda"
- **SMS demande d'avis** : envoyé automatiquement 1h après la fin du RDV.

### Système d'avis

- SMS automatique 1h après fin du RDV → lien vers page d'avis `/review/[booking-token]`.
- Notation : **étoiles 1-5 + commentaire optionnel**.
- Avis publiés sur la page publique du salon (pas de modération manuelle Phase 1).

### Homepage client (bookly.app)

- **Layout hero 100vh** avec barre de recherche centrée (inspiré Planity, pas identique).
- **Navbar fixe en haut** :
  - Logo Bookly (gauche)
  - Raccourcis catégories au centre : Barbier · Manucure · Coiffeur · Institut de beauté · Bien-être (liens vers recherche pré-filtrée)
  - Droite : bouton "Je suis professionnel" (→ /pro/signup) + bouton "Mon compte" (→ /connexion client)
- **Hero section** : fond clair, headline courte, barre de recherche unique au centre (input texte "Quel service ou salon ?") + bouton de recherche. Pas de géolocalisation GPS Phase 1 — l'utilisateur tape ville manuellement.
- Design **proche de Planity mais distinct** : palette monochrome Bookly (#FFFFFF/#1A1A1A), pas de bleu Planity.
- Page **indexée par Google** : metadata title/description, og:image, structured data.
- Pas de section "comment ça marche" / témoignages / prix en Phase 1 — homepage minimaliste axée recherche.

### Page publique du salon

- Contenu : **Planity-style** — photo, nom, adresse, horaires, liste services (nom + durée + prix + description), note moyenne + avis clients, bouton "Réserver".
- **Indexée par Google** : Next.js 15 `generateMetadata`, og:image, structured data `LocalBusiness`.
- **Light mode uniquement** en Phase 1.

### Plans et quotas

- Tous les pros sur **Plan Agenda** (gratuit) en Phase 1.
- Colonne `plan` en DB dès Phase 1 (valeur `agenda`).
- **Quota 300 SMS de rappel/mois** (NOTIF-02 uniquement, pas les confirmations ni notifications pro). Tracé en DB, **non bloqué en Phase 1**.
- Quota se réinitialise le **1er de chaque mois calendaire**.
- Stripe Billing + feature gating entre plans → Phase 2.

### Identité visuelle

- **Palette monochrome** :
  - Background général : `#FFFFFF`
  - Titres/éléments importants : `#1A1A1A`
  - Texte paragraphes/descriptions : `#333333`
  - Texte léger (dates, infos secondaires) : `#666666`
  - Fonds cartes/sections : `#F5F5F5` avec bordures `#E0E0E0`
  - Icônes : `#1A1A1A` ou `#333333`
  - Boutons primaires : fond `#1A1A1A` + texte `#FFFFFF`
  - Boutons secondaires : fond `#F5F5F5` + bordure `#E0E0E0` + texte `#1A1A1A`
- **Accent** : vert (`#10B981` ou équivalent) pour statuts positifs uniquement (confirmé, payé, actif).
- **Police** : Inter (via `next/font/google`).
- **Border radius** : 8px (Tailwind `rounded-lg`).
- **Animations** : minimales — transitions CSS standards shadcn/ui uniquement. Pas de Framer Motion Phase 1.
- **Responsive** : mobile + desktop avec même priorité. Sidebar fixe desktop, bottom nav mobile (Planity-style).
- **Light mode uniquement** — dark mode Phase 2+.

### Claude's Discretion

- Implémentation exacte des composants shadcn/ui (quels blocks utiliser)
- Structure des fichiers et découpage des Server Actions
- Choix du mécanisme cron pour rappels SMS (pg_cron vs Vercel Cron)
- Structure exacte du schéma de routes Next.js
- Format exact du slug salon (slug automatique ou saisi par le pro)
- Gestion des conflits de slugs
- Loading skeletons et états de chargement
- Logique exacte de "premier disponible" (random vs round-robin vs premier slot)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Projet et exigences
- `.planning/PROJECT.md` — Vision, stack contrainte, contexte marché Gabon, tarification
- `.planning/REQUIREMENTS.md` — 43 exigences v1 (AUTH, ONBOARD, BOOKING, WALKIN, AGENDA, PAYMENT, NOTIF, DASHBOARD)
- `.planning/ROADMAP.md` — Phases, chemin critique Phase 1, dépendances entre plans
- `.planning/STATE.md` — Décisions pré-Phase 1 verrouillées (Twilio, pawaPay, Next.js 15, JWT hook)

### Recherche technique Phase 1
- `.planning/phases/01-mvp-core/01-RESEARCH.md` — Architecture multi-tenant RLS, schéma DB, btree_gist, Custom Access Token Hook, pawaPay v2, Twilio, notification_queue

### Référence produit
- planity.com — Référence UX/UI pour : navigation pro, vue agenda, page publique salon, flux booking client, horaires pro

### Validation
- `.planning/phases/01-mvp-core/01-VALIDATION.md` — Stratégie de tests par plan (Vitest + Playwright + pgTAP)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Aucun code existant — Phase 1 from scratch.

### Established Patterns
- Stack : Next.js 15 App Router, TypeScript strict, Tailwind CSS 3.4, shadcn/ui, Supabase, Prisma (migrations uniquement), Twilio, pawaPay
- Pattern Result : `{ data, error }` sur toutes les Server Actions
- Zod sur toutes les entrées
- RLS PostgreSQL activé sur toutes les tables
- Queries runtime via Supabase JS client (pas Prisma) pour que RLS s'applique

### Integration Points
- Supabase Cloud région London — toutes les tables + Auth + Realtime + Edge Functions
- Twilio — SMS via notification_queue (async, retry 3x)
- pawaPay — Airtel Money Gabon (`AIRTEL_GAB`) + refund API Phase 1 (annulations remboursables)
- Vercel — déploiement frontend + API routes
- Timezone : Africa/Libreville (UTC+1, pas de DST)

</code_context>

<specifics>
## Specific Ideas

- Planity comme référence produit principale — UX du flux booking, page salon, agenda, navigation pro
- Design monochrome pur (noir/blanc/gris), accent vert pour statuts positifs uniquement
- Police Inter, border radius 8px, animations minimales
- "Bookly" — le nom de la plateforme. Domaine cible : bookly.app
- Marché cible : Libreville, Gabon. Langue : français. Devise : FCFA (XAF, INTEGER, format "9 900 FCFA")
- Modèle économique : SaaS 2 plans (Plan Agenda ~9 900 FCFA/mois, Plan Pro ~19 900 FCFA/mois) — facturation en Phase 2

</specifics>

<deferred>
## Deferred Ideas

- **Remboursements automatiques pawaPay** → Phase 2 (pawaPay Refund API)
- **Replanification directe** (changer créneau sans annuler) → Phase 2
- **Stripe Billing + feature gating par plan** → Phase 2
- **Stripe paiement CB** → Phase 2
- **PWA offline** → Phase 2
- **Notifications WhatsApp** → Phase 2 (délai Meta 4-8 semaines)
- **Moov Money** → non supporté pawaPay Gabon — à surveiller
- **CRM fiches clients complètes** → Phase 2
- **Interface caisse POS complète** → Phase 2
- **Cartes cadeaux** → Phase 2
- **Gestion des stocks** → Phase 2
- **Export comptable CSV** → Phase 2
- **Dark mode** → Phase 2
- **Framer Motion animations** → Phase 2
- **Géolocalisation GPS dans la recherche** → Phase 2+
- **Widget iframe intégrable** → Phase 4
- **App mobile native** → Phase 4
- **Marketplace complète** → Phase 4 (Phase 1 = recherche texte basique)
- **Acomptes remboursables automatiques** → Phase 2
- **Délai d'annulation configurable** → déjà en Phase 1 mais sans remboursement auto

</deferred>

---

*Phase: 01-mvp-core*
*Context gathered: 2026-04-12*

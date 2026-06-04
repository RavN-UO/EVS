# CLAUDE.md — EVS (« Nos primes, notre priorité »)

Document de référence pour reprendre le projet dans une nouvelle conversation.
Il décrit le fonctionnement complet de l'application, son architecture, les
règles de calcul, et l'historique des travaux réalisés.

---

## 1. Vue d'ensemble

Application web **mono-fichier** (`index.html`) pour un usage syndical/transport
(IPSO-UPTX-SUD). Elle est utilisée principalement en **PWA sur iPhone** (ajout à
l'écran d'accueil), donc **tout est pensé mobile-first**.

Deux onglets :

1. **EVS** — calculateur de primes (taux × quantité), total, résumé, export PDF.
2. **Simulateur de déplacement** — saisie d'amplitudes de service jour par jour,
   calcul du temps de repos entre jours, avec une **vue calendrier**.

Aucune dépendance, aucun build : **HTML + CSS + JavaScript vanilla** dans un seul
fichier. Persistance via `localStorage`. Fonctionne hors-ligne une fois chargée.

### Fichiers du dépôt
- `index.html` — toute l'application (le seul fichier à modifier en pratique).
- `manifest.webmanifest` — manifest PWA (nom « EVS », thème `#8c2a52`).
- `favicon-32.png`, `apple-touch-icon.png`, `icon-192.png`, `icon-512.png`,
  `icon-maskable-512.png` (+ quelques doublons sans tiret type `favicon32.png`).
- `CLAUDE.md` — ce document.

---

## 2. Structure de `index.html`

1. `<head>` : meta PWA (apple-touch-icon, manifest, theme-color, web-app-capable…).
2. `<style>` : CSS organisé en sections commentées
   (Variables → Base → En-tête → Onglets → Champs → Cartes → Total → Résumé →
   Boutons → Simulateur → Calendrier → Responsive → **Impression `@media print`**).
3. `<body>` : en-tête (cadre rayé + slogan), nav onglets, section `#tab-evs`,
   section `#tab-deplacement`, footer.
4. `<script>` : **trois IIFE indépendantes** (`(function(){ "use strict"; … })()`):
   - **IIFE 1 — Navigation par onglets.**
   - **IIFE 2 — Onglet EVS** (calculateur de primes).
   - **IIFE 3 — Onglet Simulateur** (amplitude / repos / calendrier).

Les trois closures n'exposent **aucune variable globale**.

### Variables CSS (`:root`)
`--accent: #8c2a52` (bordeaux), `--accent-dark`, `--ok` (vert), `--bad` (rouge),
`--rest` (teal pour RP), `--muted`, `--border`, etc.

---

## 3. Onglet EVS (IIFE 2)

### Données
- `STORAGE_KEY = "calcul-evs-v1"`.
- `state = { period, residence, primes: [...] }`.
- Chaque prime : `{ id, name, rate, qty, fixedQty, custom }`.
- `DEFAULT_PRIMES` : Traitement (`fixedQty:true`, quantité bloquée à 1),
  Indemnité local UPTX, Tx IDF, Repas, Conduite (taux A), Week-ends, CASTOR,
  Déplacements, Tunnel. L'utilisateur peut **ajouter des lignes personnalisées**
  (bouton « ＋ Ajouter une ligne ») et les supprimer (croix).

### Calculs
- `lineTotal(p) = rate × qty`. `grandTotal()` = somme.
- **Taux horaire** (carte Traitement uniquement) =
  `(traitement + indemnité de résidence) / 140` (`HOURLY_DIVISOR = 140`),
  affiché seulement si l'indemnité de résidence est renseignée.
- **Résumé** (`SUMMARY`) : agrège certaines primes en libellés « officiels »
  (IND JOURNALIERE TRANSPORT PERSONNEL, ALLOC DE DEPLACEMENT…, INDEMNITE LOCALE…).

### Persistance / migration
`loadState()` reconstruit toujours dans l'ordre de `DEFAULT_PRIMES` (pour faire
apparaître d'éventuelles nouvelles primes) puis ré-applique les valeurs
sauvegardées et ré-ajoute les primes `custom`. Robuste aux anciens enregistrements.

### Export PDF (impression)
- Bouton « Exporter en PDF » → `window.print()`.
- **Réservé au navigateur ordinateur** : sur mobile/tablette le bouton est retiré
  (`isMobileDevice()` = user-agent mobile **ou** `pointer:coarse && hover:none`).
- `@media print` : `@page { margin:0 }` (supprime en-tête/pied URL du navigateur),
  masque les onglets / le simulateur / les boutons, met chaque prime sur **une
  ligne compacte** à largeurs fixes, conserve l'**en-tête (cadre rayé) aux mêmes
  proportions que le site**.
- **Ligne Traitement à l'impression** : la colonne Taux affiche le **taux horaire
  sans le symbole €** (cale `.print-qty-spacer` pour aligner avec les autres
  taux) ; le **montant du traitement** reste à droite dans Total. Voir
  `.print-rate-h`, `#printTauxHoraire`, formateur `decimal2`.

---

## 4. Onglet Simulateur (IIFE 3) — cœur du projet

### Modèle « jours consécutifs » (IMPORTANT)
Le simulateur suppose des **jours consécutifs** à partir d'une seule date
`state.firstDate`. `dayDateObj(i) = firstDate + i jours`. Les minutes absolues
utilisent `i * DAY` (`DAY = 24*60`). **Ne pas** introduire de dates arbitraires
non consécutives sans réécrire toute l'arithmétique (hors périmètre).

### Données
- `STORAGE_KEY = "deplacement-v1"` — **ne jamais renommer** (effacerait les
  données utilisateurs). Tout nouveau champ doit être **additif** avec valeur par
  défaut sûre dans `loadState()`.
- `state = { firstDate, days: [...] }`.
- Un jour : `{ s, e, aS, rE, rp, night, coupure }`
  - `s`/`e` = début/fin de **Service** (format « 07h30 »).
  - `aS` = heure de départ (Trajet **aller**) ; `rE` = heure de retour (Trajet **retour**).
  - `rp` = jour de repos (RP, non travaillé).
  - `night` = service de nuit (seuil de repos 14 h au lieu de 12 h).
  - `coupure` = déduit 1 h de l'amplitude.
  - (Le champ `collapsed` a existé puis a été retiré : plus de formulaire dépliable.)

### Helpers horaires
- `formatTime("0730") -> "07h30"` (insère « h » après 2 chiffres, max 4).
- `toMinutes("07h30") -> 450` (null si incomplet/invalide).
- `span(start,end)` : durée, `+DAY` si on passe minuit.
- `fmtDuration(min) -> "9h15"`.

### Calculs (NE PAS réécrire, seulement appeler)
- `dayAmplitude(day)` = service + trajet aller + trajet retour − (coupure ? 60 : 0).
- `dayStart`/`dayEnd`/`startAbs(i)`/`endAbs(i)` : minutes absolues, gestion du
  passage de minuit (`+DAY` si fin < début).
- `prevWorked(i)` / `nextWorked(i)` : jour travaillé précédent / suivant (sautent les RP).
- `restBefore(i)` = `startAbs(i) − endAbs(prevWorked(i))` (englobe les RP).
- `restThreshold(prevDay)` = 14 h si `night`, sinon 12 h.
- `restStatusInto(i)` -> `"ok" | "bad" | ""` : statut du repos entrant en i
  (neutralise les RP intercalés : `effective = rest − numRP*DAY`).
- `compute()` : met à jour les amplitudes (`#dayAmp-i`), les pastilles de repos
  (`#rest-i`, `#rest-icon-i`) **et** appelle `renderCalendar()`.

> **Contrat d'IDs** : `compute()` cible les éléments par `id` (`dayAmp-i`,
> `rest-i`, `rest-icon-i`). Toute UI affichant amplitude/repos doit réutiliser
> `renderDays()` + `compute()` ou conserver ce contrat. Après toute mutation de
> `state` : `saveState()` puis `renderDays()` (ou `compute()` pour les saisies
> horaires, afin de **ne pas perdre le focus** du champ en cours).

### Rendu d'un jour (`buildDay`) — format actuel (compact, non dépliable)
Chaque carte jour = 3 blocs :
1. **En-tête** (`.label-row`, flex) :
   - À gauche (`.day-head-left`) : **switch Jour/Nuit** (☀️/🌙, jours travaillés)
     **à gauche de la date**, puis la **date** (`dayLabel`), le **badge nuit**
     (`nightPair`, ex. « DI/LU ») et la **case RP**.
   - À droite (`.day-head-right`) : **amplitude** (pastille `#dayAmp-i`) et un
     bouton **réinitialiser le jour** (`↻`, discret, gris) qui vide les saisies.
2. **Ligne éditable** (`buildDayEdit`, `.day-edit`, grille 5 colonnes) :
   **Aller · Début · Coupure · Fin · Retour**. On **clique sur la valeur** pour la
   saisir (inputs `.de-input`, format auto, `inputmode=numeric`). La **Coupure est
   un switch** (`.coupure-switch`) avec le libellé « Coupure » en dessous. Cellules
   en colonne (valeur au-dessus, libellé en dessous), volontairement compactes.
3. **Ligne d'actions** (`.day-actions`, si un jour travaillé précède) :
   - **« ⧉ Service J-1 »** : recopie `s`, `e`, `coupure` du dernier jour travaillé.
   - **« ⧉ Trajet J-1 »** : recopie `aS`, `rE` du dernier jour travaillé.

Un **jour RP** n'affiche que l'en-tête (pas de saisie).

### Compteur de repos dans la liste (`buildRest` / `#rest-i`)
Inséré entre deux jours travaillés **uniquement s'il a une valeur**
(`restBefore(i+1) !== null`). Affiche « Temps de repos » + durée + une icône :
**flèche verte `↓`** (repos suffisant) ou **croix rouge `✕`** (insuffisant).

### Vue calendrier (`renderCalendar`, `#calendar` / `.cal`)
Insérée une fois au-dessus de la liste des jours. **Grille 7 colonnes
Lu→Di** ; chaque jour saisi se place **sous son vrai jour de semaine** (la 1ʳᵉ
case reçoit `gridColumnStart = mondayIndex(date)` ; les suivantes suivent et
passent à la ligne). Chaque case : numéro du jour (ou « J1 » sans date),
amplitude (ou « RP »). **Tap** sur une case = défile jusqu'à la carte du jour
(`scrollIntoView` + surbrillance `.flash`).

Repères de repos **dans le calendrier** :
- **Entrant** (bord gauche de la case du jour travaillé qui reprend) : `→` verte
  (repos suffisant) ou `✕` rouge (insuffisant). Englobe les RP intercalés.
- **Sortant** (bord droit, classe `.cal-mark.leave`) : `→` **verte uniquement**,
  posée sur le **jour qui précède une période RP** quand le repos qui suit est
  suffisant. Ex. `Lu → Ma(RP) → Me` en vert : flèche verte qui part de **Lu**
  *et* flèche verte qui arrive à **Me**. En **rouge**, on ne met **que** la croix
  rouge avant la reprise (pas de marque sortante).

### Ajout / retrait de jours (depuis le calendrier)
- **« + »** : case en fin de calendrier (`.cal-add`), toujours sur le **créneau
  suivant**. `addDay()` ajoute un `newDay()`.
- **« − »** : petit rond **sous la case du dernier jour** (`.cal-rem`).
  `removeLastDay()` retire le dernier jour (si plus d'un). `stopPropagation` pour
  ne pas déclencher le scroll de la case.
- Il n'y a plus de gros bouton « Ajouter un jour », ni de croix de suppression par
  carte : le modèle est séquentiel (on agrandit/réduit par la fin).
- Bouton **« Réinitialiser »** global (`#resetDep`) : remet 5 jours vides.

---

## 5. Conventions & contraintes à respecter

- **Travailler dans la bonne IIFE.** Ne pas toucher aux deux autres sans raison.
- **Ne pas renommer** les clés `localStorage` (`calcul-evs-v1`, `deplacement-v1`).
  Tout nouveau champ de jour/prime = **additif** + défaut sûr dans `loadState()`.
- **Modèle jours consécutifs** : le calendrier est une couche d'affichage par
  dessus `state.days`, pas un sélecteur de dates arbitraires.
- **Flux de mutation** : muter `state` → `saveState()` → `renderDays()` (ou
  `compute()` pour les saisies horaires, pour préserver le focus).
- **`@media print`** ne concerne que l'onglet EVS ; le simulateur est masqué à
  l'impression. Ne pas l'y faire apparaître.
- **Mobile-first** : cibles tactiles confortables, pas de `prompt()`, l'URL ne
  doit pas apparaître à l'export PDF.

---

## 6. Historique des travaux (du plus ancien au plus récent)

1. **Export PDF** : cadre rayé du titre aux mêmes proportions que le site ; ligne
   Traitement → taux horaire aligné dans la colonne Taux, montant dans Total.
   Export identique web/mobile, URL masquée (`@page{margin:0}`).
2. **Simulateur** : case RP éloignée de la croix ; **Coupure** déplacée puis,
   finalement, transformée en **switch** dans la ligne éditable.
3. **Champ date** : affiche « Calendrier » tant que vide, sinon la date formatée ;
   le texte natif `jj/mm/aaaa` est **totalement masqué** (couleur transparente +
   `::-webkit-datetime-edit { opacity:0 }`), champ pleine largeur, tout le champ
   ouvre le sélecteur (`showPicker`).
4. **Libellés EVS** : « Indemnité de résidence - sans complément (€) » ; mention
   « sans complément » retirée de l'indication du taux horaire.
5. **Taux horaire à l'impression** : sans le symbole €.
6. **Export PDF** retiré sur mobile (réservé ordinateur).
7. **Saisie accélérée** : bouton « Service J-1 » (duplique service+coupure) et
   « Trajet J-1 » (duplique les trajets).
8. **Vue calendrier 7 colonnes** (Lu→Di) avec amplitude, repères de repos, tap
   pour défiler.
9. **Repères de repos** : flèche verte / croix rouge entre les jours, y compris la
   **flèche verte sortante** du jour précédant une période RP (repos vert).
10. **Refonte saisie mobile** : suppression du formulaire dépliable ; chaque jour
    = en-tête + ligne éditable compacte + ligne d'actions ; switch Jour/Nuit à
    gauche de la date ; reset = `↻` discret ; +/− gérés dans le calendrier.
11. **Nettoyage** : suppression du code mort (ancien formulaire `seg-*`,
    `timeInput`, `buildDaySummary`, champ `collapsed`, CSS associé), corrections
    de cohérence.

---

## 7. Workflow de développement

- **Pas de build** : ouvrir `index.html` dans un navigateur (ou via la PWA).
- Branche de travail Git dédiée ; commits petits et descriptifs ; push sur la
  branche, **pas de PR sauf demande explicite**.
- **Vérifications rapides** avant commit :
  - JS : extraire le contenu de `<script>` et `node --check`.
  - CSS : compter l'équilibre des accolades `{` / `}` dans `<style>`.
  - Idéalement, tester en conditions réelles (Safari iOS, PWA plein écran).
- **PWA** : après une mise à jour, forcer le rechargement / réinstaller le
  raccourci écran d'accueil pour récupérer la nouvelle version (cache).

---

## 8. Pistes / points ouverts

- 7 colonnes sur iPhone restent compactes (~40–60 px/colonne) : ajustable.
- Sans date saisie, le calendrier retombe sur des libellés « J1, J2… »
  (alignement par jour de semaine impossible tant qu'aucune date n'est choisie).
- « Dates arbitraires non consécutives » = hors périmètre (imposerait une refonte
  complète de l'arithmétique de temps absolu + migration `deplacement-v2`).

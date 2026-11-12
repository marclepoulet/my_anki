# CLAUDE.md

Guide de développement et contexte système pour la création et la maintenance de la Progressive Web App (PWA) de révision hors-ligne (clone d'Anki) pour iPhone.

---

## 1. Vue d'ensemble du projet

Un système de répétition espacée (SRS) sans dépendance et 100 % local, conçu strictement pour un usage personnel sur iPhone via le mode Standalone de Safari iOS.

- **Appareil cible :** iPhone (iOS 16+)
- **Modèle d'exécution :** PWA Standalone (plein écran, sans interface de navigateur ni barre d'URL)
- **Environnement d'exécution :** 100 % hors-ligne via Service Worker (`Cache-First`)
- **Persistance :** Base de données locale du navigateur (`IndexedDB`) avec sauvegarde/restauration manuelle en JSON
- **Système de build :** Zéro-build (ES6+ vanilla, fichiers statiques, sans npm ni bundler)

---

## 2. Structure du dépôt

```
.
├── index.html       # Interface principale, état de l'application, événements DOM, wrapper IndexedDB
├── sw.js            # Cycle de vie du Service Worker, pré-mise en cache des fichiers, interception réseau
├── manifest.json    # Web App Manifest (affichage standalone, couleurs du thème, icônes)
├── icon-180.png     # apple-touch-icon (écran d'accueil iOS)
├── icon-192.png     # Icône manifest
├── icon-512.png     # Icône manifest (aussi « maskable »)
├── examples/
│   └── cartes.csv   # Exemple de fichier CSV importable
└── CLAUDE.md        # Spécifications techniques et directives du dépôt
```

L'ancienne version Expo / React Native est archivée sur la branche git `expo-legacy`.

### Fonctionnalités
- **Accueil :** nombre de cartes à réviser, objectif quotidien, série de jours, révision par sujet, rappel de sauvegarde si la dernière date de plus de 7 jours. Met à jour la pastille de l'icône (`navigator.setAppBadge`, iOS 16.4+, nécessite l'autorisation des notifications).
- **Cartes :** hiérarchie **Sujet → Thème → Cartes** (création, renommage, suppression en cascade, recherche). Image facultative par carte (affichée avec le recto, réduite à 800 px en JPEG). Suspension d'une carte (exclue des révisions et du quiz). Sens de révision par thème : recto→verso, verso→recto ou au hasard (une seule planification par carte). **Texte à trous** : un recto contenant `{{mot}}` affiche un blanc, le verso devient facultatif ; ces cartes posent toujours le recto. Création d'une carte au recto déjà existant : confirmation demandée. **Sélection multiple** dans un thème (« Choisir ») : déplacer, suspendre, réactiver, supprimer. L'éditeur affiche la progression et les 10 dernières réponses de la carte.
- **Révision :** file SM-2 (cartes dues + nouvelles cartes dans la limite quotidienne), 4 boutons avec aperçu de l'intervalle, glisser à gauche = À revoir / à droite = Bon. Les cartes notées « À revoir » reviennent en fin de séance (liste en mémoire ; ces réponses sont enregistrées mais ne replanifient pas). « Annuler la dernière réponse » restaure la carte et supprime la révision. Lecture à voix haute (`speechSynthesis`, voix japonaise si le texte contient des kana/kanji).
- **Révision libre :** toujours disponible (accueil, sujet, thème), sans fin. Tire une carte au hasard proportionnellement à son poids `weight` (jamais deux fois de suite la même). Chaque note multiplie le poids : À revoir ×2, Difficile ×1,5, Bon ×0,8, Facile ×0,5 (bornes 0,1–10). Les notes de la révision SM-2 mettent aussi le poids à jour. La révision libre **ne touche pas** la planification SM-2.
- **Quiz :** QCM à 4 choix ou réponse tapée (comparaison sans casse/accents/ponctuation, « a, b / c » accepte chaque partie, bouton « J'avais bon ») (distracteurs pris dans le même thème, puis le même sujet, puis partout), sens recto→verso ou inverse, « rejouer les erreurs ». Mode **à l'écoute** : la question est lue (`speechSynthesis`) et affichée seulement après la réponse. Texte à trous : question = phrase avec blancs, réponse = mots cachés. **N'influence pas** la planification SM-2.
- **Stats :** totaux, réussite sur 7 jours, calendrier d'activité sur 17 semaines, activité sur 14 jours, prévision sur 7 jours, cartes difficiles.
- **Réglages :** objectif quotidien, nouvelles cartes/jour, heure de début de journée, lecture automatique du recto, sauvegarde JSON, import/export CSV, effacement total.
- **Mises à jour :** quand un nouveau Service Worker prend le contrôle, la page se recharge seule (sauf pendant une révision ou un quiz).

---

## 3. Workflows de développement et de vérification

Les Service Workers exigent un contexte sécurisé (`HTTPS`) ou `localhost`. L'exécution directe via `file://` n'est pas prise en charge.

### Exécution en local
```bash
# Option A : Python 3
python3 -m http.server 8080

# Option B : Node.js (via npx, aucune installation requise)
npx serve . -l 8080

# Option C : PHP CLI
php -S localhost:8080
```

### Déploiement et test sur iOS
1. Pousser les fichiers du dépôt sur un hébergeur statique supportant le HTTPS (ex. GitHub Pages, Cloudflare Pages, Vercel).
2. Ouvrir l'URL de production dans Safari sur l'iPhone.
3. Toucher **Partager** -> **Sur l'écran d'accueil**.
4. Lancer l'application exclusivement depuis l'icône ajoutée à l'écran d'accueil.
5. Activer le **Mode Avion** et relancer l'application pour vérifier l'exécution complète hors-ligne depuis le cache.

---

## 4. Exigences pour le mode Standalone iOS Safari

Toute modification apportée à `index.html` ou au CSS interne doit respecter le comportement standalone de Safari sur iOS :

### Viewport et zones de sécurité (Safe Areas)
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Anki">
```
- Prendre en compte la Dynamic Island, l'encoche et la barre d'accueil via les variables d'environnement CSS :
  ```css
  padding-top: calc(env(safe-area-inset-top) + 12px);
  padding-bottom: calc(env(safe-area-inset-bottom) + 16px);
  padding-left: calc(env(safe-area-inset-left) + 16px);
  padding-right: calc(env(safe-area-inset-right) + 16px);
  ```

### Ergonomie tactile et gestuelle
- Appliquer `touch-action: manipulation;` sur les cibles interactives et les conteneurs racines pour supprimer le délai de zoom de 300ms au double-tap.
- Appliquer `user-select: none;` (`-webkit-user-select: none;`) globalement sur les cartes de révision, les libellés et les boutons d'action pour éviter la sélection accidentelle de texte.
- Réactiver explicitement la sélection pour les champs de saisie :
  ```css
  input, textarea {
    user-select: text;
    -webkit-user-select: text;
  }
  ```

---

## 5. Architecture de stockage (`IndexedDB`)

Ne pas utiliser `localStorage` ou `sessionStorage` (limités à ~5 Mo et vulnérables à la purge automatique du cache d'iOS en cas de stockage faible).

### Spécifications de la base de données
- **Nom de la base :** `AnkiLocalDB`
- **Version du schéma :** `1` (toute modification de schéma = incrémenter `DB_VERSION` et migrer dans `onupgradeneeded`)

| Object Store | Clé primaire | Index (non-uniques) |
| :--- | :--- | :--- |
| `subjects` | `id` (`autoIncrement`) | — |
| `themes` | `id` (`autoIncrement`) | `subjectId` |
| `cards` | `id` (`autoIncrement`) | `dueDate` (cartes dues ≤ `Date.now()`), `themeId` |
| `reviews` | `id` (`autoIncrement`) | `createdAt`, `cardId` |
| `settings` | `key` | — |

### Modèle de données
```typescript
interface Subject { id?: number; name: string; createdAt: number; updatedAt: number; }
interface Theme   { id?: number; subjectId: number; name: string; direction?: 'front' | 'back' | 'mixed'; createdAt: number; updatedAt: number; }

interface Flashcard {
  id?: number;              // Clé primaire auto-générée
  themeId: number;          // Thème parent
  front: string;            // Texte de la question / recto
  back: string;             // Texte de la réponse / verso
  notes: string;            // Infos complémentaires (facultatif, '' par défaut)
  interval: number;         // Intervalle d'espacement actuel en jours
  repetitions: number;      // Nombre de rappels réussis consécutifs
  easeFactor: number;       // Multiplicateur (initial : 2.5, plancher : 1.3)
  dueDate: number;          // Timestamp Unix en millisecondes
  createdAt: number;
  lastReview: number | null; // null = carte nouvelle (jamais révisée)
  lapses: number;           // Nombre de « À revoir »
  suspended?: boolean;      // true = exclue des révisions et du quiz
  weight?: number;          // poids de tirage en révision libre (0.1–10) ; absent = (1 + 0.5 × lapses) × 2.5 / easeFactor
  image?: string;           // data:image/... (JPEG ≤ 800 px) ; seules les URL data:image sont acceptées à l'import
}

interface Review {          // Historique : révisions SM-2, révision libre ET réponses de quiz
  id?: number;
  cardId: number;
  kind: 'review' | 'free' | 'quiz';
  rating: 1 | 2 | 3 | 4 | null; // null pour un quiz
  correct: boolean;
  wasNew: boolean;          // sert à appliquer la limite « nouvelles cartes / jour »
  createdAt: number;
}

// settings : { key: 'dailyGoal', value: 20 }, { key: 'newPerDay', value: 20 }, { key: 'lastBackup', value: 0 } (timestamp du dernier export JSON),
//            { key: 'dayStartHour', value: 4 } (0–23), { key: 'autoplay', value: false }
// Les champs facultatifs ci-dessus ne nécessitent pas de migration (pas de nouvel index) : DB_VERSION reste 1.
```

- Supprimer un sujet supprime ses thèmes et leurs cartes ; supprimer un thème supprime ses cartes (dans une seule transaction). L'historique `reviews` est conservé.

### Règles de transaction
- Encapsuler les opérations d'écriture dans des transactions explicites `readwrite`.
- Toujours attacher des gestionnaires `.onerror` et `.onsuccess`.
- Fermer les modales et mettre à jour l'état de l'interface dans le callback `tx.oncomplete` pour garantir que les données sont bien écrites sur le disque.

---

## 6. Algorithme de répétition espacée (SM-2)

Le moteur implémente l'algorithme standard SuperMemo 2 avec 4 niveaux d'évaluation :

### Correspondance des notes
| Action | Note ($q$) | Comportement |
| :--- | :---: | :--- |
| **À revoir (Again)** | `1` | Réinitialisation : `repetitions = 0`, `interval = 1` |
| **Difficile (Hard)** | `2` | Progression minimale : facteur d'intervalle réduit, baisse du facteur d'aisance |
| **Bon (Good)** | `3` | Progression normale : `interval = round(interval * easeFactor)` |
| **Facile (Easy)** | `4` | Progression accélérée : multiplicateur bonus supplémentaire sur l'intervalle |

### Formules
1. **Ajustement du facteur d'aisance ($EF'$) :**
   $$EF' = EF + (0.1 - (5 - g) \times (0.08 + (5 - g) \times 0.02)), \quad g = q + 1$$
   Les boutons 1–4 correspondent aux notes SM-2 2–5 : À revoir −0,32, Difficile −0,14, Bon ±0, Facile +0,1.
   *Plancher :* Si $EF' < 1.3$, définir à $1.3$.

2. **Calcul de l'intervalle ($I$ en jours) :**
   - Si $q = 1$ (À revoir) :
     $$I = 1, \quad \text{repetitions} = 0$$
   - Si $q \ge 2$ :
     - Si $\text{repetitions} = 0$ : $I = 1$
     - Si $\text{repetitions} = 1$ : $I = 6$
     - Si $\text{repetitions} > 1$ : $I = \text{round}(I_{\text{précédent}} \times EF)$
     - Incrémenter les répétitions : $\text{repetitions} = \text{repetitions} + 1$
   - Multiplicateur bonus $1.3\times$ appliqué lorsque $q = 4$ (avec au minimum $I + 1$, pour que « Facile » dépasse toujours « Bon »).
   - Pour $q = 2$ et $\text{repetitions} > 1$ : $I = \text{round}(I_{\text{précédent}} \times 1.2)$ au lieu de $\times EF$.

3. **Calcul de la date d'échéance (`dueDate`) :**
   $$\text{dueDate} = \text{startOfDay}(\text{Date.now}() + I \times 86{,}400{,}000)$$
   `startOfDay` renvoie le début de la « journée d'étude », qui commence à `dayStartHour` (4 h par défaut) : une carte notée à 21 h est due dès le lendemain matin.

4. **Cartes « sangsues » :** à la 8e erreur (`lapses ≥ LEECH`), la carte est suspendue automatiquement pour être reformulée.

---

## 7. Cycle de vie du Service Worker (`sw.js`)

- **Stratégie :** Cache-First avec repli sur le réseau (network fallback).
- **Nom du cache :** Incrémenter la version à chaque modification des fichiers (ex. `anki-local-v2`).
- L'événement `activate` doit parcourir les clés de cache existantes et supprimer toute entrée ne correspondant pas à `CACHE_NAME`.
- Toutes les ressources de l'application (`./`, `./index.html`, `./manifest.json`, `./sw.js`, `./icon-180.png`, `./icon-192.png`, `./icon-512.png`) doivent être pré-mises en cache lors de l'événement `install`.

---

## 8. Sauvegarde et portabilité des données

- **Exportation JSON :** Lire tous les stores et produire `{ app: 'anki-local', version: 1, exportedAt, subjects, themes, cards, reviews, settings }` dans un `Blob`, partagé via la feuille de partage iOS (`navigator.share`) ou téléchargé (`anki_backup_YYYY-MM-DD.json`).
- **Importation JSON :** Lire le fichier via `FileReader`. Supprimer les `id` existants pour éviter les collisions, puis insérer sujets → thèmes → cartes → historique **dans une transaction atomique unique**, en re-mappant `subjectId`, `themeId` et `cardId` vers les nouvelles clés. L'utilisateur choisit **Ajouter** (aux données existantes) ou **Remplacer** : les 4 stores sont vidés dans la même transaction, et les réglages du fichier sont appliqués (clés connues et de type valide uniquement, sauf `lastBackup`). Un ancien export au format « tableau de `Flashcard` » est accepté et rangé dans le sujet « Importé › Général ».
- **CSV :** séparateur `,`, `;` ou tabulation détecté automatiquement. En-têtes reconnus : `subject`/`sujet`, `theme`, `front`/`recto`/`question`, `back`/`verso`/`reponse`, `notes`. Les colonnes `kana`, `romaji`, `kanji`, `translation`, `example` sont ajoutées aux notes. Sans en-tête reconnu : `front, back, notes`. Les sujets/thèmes manquants sont créés par nom. Les cartes dont recto + verso (normalisés) existent déjà, en base ou plus haut dans le fichier, sont ignorées et comptées comme doublons.

---

## 9. Directives de codage pour l'IA et les contributeurs

- **Aucun outil de build :** Ne pas ajouter Webpack, Vite, des paquets npm ou Babel. Les fichiers doivent s'exécuter directement dans un navigateur standard.
- **Source unique de vérité :** Ne pas maintenir de tableaux d'état parallèles désynchronisés d'`IndexedDB`. Valider d'abord les écritures en base, puis rafraîchir la file d'attente depuis la base de données.
- **Aucune dépendance CDN externe :** Éviter les polices ou scripts externes susceptibles d'échouer hors-ligne. Utiliser les polices système natives (`-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto`).
- **Organisation du fichier :** Maintenir le code structuré en sections fonctionnelles distinctes :
  1. Initialisation et migration IndexedDB
  2. Gestion de la file d'attente de révision
  3. Logique SM-2
  4. Gestionnaires DOM et contrôleurs de modales
  5. Logique de sauvegarde / restauration
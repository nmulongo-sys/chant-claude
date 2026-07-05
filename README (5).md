# Chant & Justesse

Application web autonome d'entraînement au déchiffrage chanté et à la justesse, conçue pour adapter automatiquement des exercices de formation musicale à une tessiture grave, et généralisée à tout chanteur via un kit vocal personnel.

**En ligne** : ouvrir `index.html` dans un navigateur (aucun serveur requis).
**Statut** : révision initiale (2026-07-05) — onglets Partition et Écoute fonctionnels ; Chant, Studio, Bibliothèque en attente d'implémentation. Fichier HTML unique, bibliothèques embarquées, fonctionne hors ligne.

## Utilisation

Fichier unique, aucune installation, hors ligne après le premier chargement, pensé mobile-first (Android) et compatible desktop.

**Charger un exercice** (onglet Partition), par trois voies :
- fichier MusicXML (`.musicxml` / `.xml`) ou MusicXML compressé (`.mxl`) — typiquement exporté depuis PlayScore 2 ;
- fichier ABC (`.abc` / `.txt`) ;
- saisie ou collage direct dans la zone ABC, éditable (sert aussi à corriger les erreurs d'OCR musical ou de transcription).

**Adapter à ma voix** : régler la tessiture (note grave / note aiguë). L'app transpose l'exercice par octaves pour le faire entrer dans la tessiture, puis bascule la clé (sol / fa) pour rester lisible. Les deux décisions sont automatiques et débrayables (Auto / 8va↓ / 8va↑ ; Auto / Sol / Fa). L'**Échelle** visualise la tessiture (bande dorée) et l'ambitus de l'exercice après adaptation — vert s'il tient, corail s'il déborde, avec le nombre de demi-tons de débordement.

**Écouter** (onglet Écoute) : lecture à la voix de synthèse, tempo 30–200 BPM réglable, mode continu ou note à note, nom des notes énoncé en français, curseur synchronisé sur la partition.

## Architecture & conventions

**Format pivot : ABC.** Tout passe par la notation ABC, rendue par `abcjs`. Les fichiers MusicXML sont convertis en ABC **sur l'appareil** (aucun réseau).

**Bibliothèques embarquées** (inline dans le HTML, pour le hors-ligne intégral) :
- `jQuery` + `xml2abc.js` (port JS de Willem Vree) : conversion MusicXML → ABC. `xml2abc` dépend de jQuery et sa fonction `vertaal()` attend un document **déjà parsé en jQuery** — l'appel correct est `vertaal($(doc), {})`, pas une chaîne brute.
- `abcjs` : rendu de la partition, modèle de données (extraction de l'ambitus), et `TimingCallbacks` pour le curseur de lecture.
- `fflate` : dézippage des `.mxl` (le `.mxl` est un ZIP ; le XML principal est repéré via `META-INF/container.xml`, sinon le premier `.xml` hors `META-INF`).

**Structure du fichier** : HTML (header + onglets + 5 panneaux) → CSS (thème par variables) → scripts des bibliothèques inline → script applicatif unique en IIFE.

**Calcul du MIDI (déterministe, hors-ligne).** Le MIDI de chaque note est calculé à la main depuis le modèle abcjs, sans dépendre du moteur audio d'abcjs :
- unités de hauteur abcjs : `0 = do central`, en degrés diatoniques (do=0, ré=1, … si=6) ; octave = `floor(pitch/7)+4`, demi-tons du degré `[0,2,4,5,7,9,11]`.
- altérations : altération explicite portée par la note (`sharp`/`flat`/`natural`/double), sinon altération locale à la mesure (réinitialisée à chaque barre), sinon altération d'armure (`tune.getKeySignature().accidentals`).
- ligne mélodique : sur un accord, la note la plus haute est retenue comme note chantée.

**Moteur de transposition.** Décalage d'octave = multiple de 12 demi-tons appliqué via `visualTranspose` d'abcjs (le texte ABC source n'est jamais modifié). En mode auto, l'app centre l'ambitus de l'exercice sur la médiane de la tessiture. La bascule de clé injecte `clef=bass`/`clef=treble` dans l'en-tête `K:` d'un ABC de rendu dérivé (la source reste intacte) ; seuil auto ≈ si3. L'affichage audio et l'affichage visuel partagent le même décalage, donc restent synchrones.

**Cascade audio (3 niveaux prévus).** Ordre de résolution par note : kit vocal utilisateur (à venir) → kit embarqué (à venir) → **synthèse**. La synthèse est un timbre vocal simplifié : deux oscillateurs `sawtooth` filtrés par deux passe-bande (~600 / ~1200 Hz) avec enveloppe. La hauteur vient de `midiToFreq = 440·2^((m−69)/12)` : la synthèse n'a **aucune limite d'octave** (elle calcule une fréquence, elle n'étire pas un échantillon). Le nom de la note est dit via `speechSynthesis` (débrayable). Le séquenceur ordonnance les notes sur l'horloge du contexte audio ; le curseur suit via `TimingCallbacks`.

**Convention de nom de note.**
- **Nom chanté** (dit à voix haute et affiché comme note à chanter) : **syllabe seule, sans octave** — « mi », « do dièse ». On ne chante pas le numéro d'octave.
- **Registre** (sélecteurs de tessiture, axe de l'Échelle, bornes d'ambitus) : syllabe **+ numéro d'octave**, car là l'octave situe une hauteur et ne se chante pas.
- Convention d'octave : **scientifique, do central = do4** (MIDI 60). Tessiture par défaut E2 → C4 (MIDI 40–60).

**Persistance.**
- Tessiture : `localStorage`, clé `cj_tess`, schéma `{ min, max }` (numéros MIDI).
- Kit vocal (à venir) : `IndexedDB`, base `ChantJustesseDB`, store `samples`, clé primaire = numéro MIDI ; valeur prévue `{ blob, mime, cents_offset, date }`.

**Hors périmètre v1** : OCR musical embarqué (la lecture de PDF reste externe, en amont) ; polyphonie chantée ; reconnaissance du nom chanté (seule la hauteur sera évaluée).

## Journal de développement

### 2026-07-05 — Révision initiale (Partition + Écoute)
- Mise en place du fichier unique, thème et navigation à 5 onglets ; Chant / Studio / Bibliothèque en placeholders explicites.
- Décision : **ABC comme format pivot unique** (un seul moteur de rendu), au lieu d'OSMD + abcjs en parallèle.
- Import MusicXML `.xml` / `.musicxml` / `.mxl` → ABC via jQuery + `xml2abc` (`vertaal($(doc), {})`), dézippage `.mxl` via `fflate`. Pipeline vérifié en exécution.
- Calcul MIDI déterministe depuis le modèle abcjs (degré diatonique + altérations locales + armure), indépendant du moteur audio d'abcjs.
- Moteur de transposition par octaves (`visualTranspose`) centré sur la tessiture, + bascule de clé auto par réécriture de l'en-tête `K:` d'un ABC de rendu dérivé. Débrayable (octave et clé). Testé : auto, ±8va manuel, clé forcée.
- Élément signature « Échelle » : visualisation tessiture vs ambitus, code couleur tient / déborde.
- Moteur audio : synthèse à formants + oscillateurs, **sans limite d'octave** (fréquence calculée, pas d'échantillon étiré) ; séquenceur sur l'horloge audio ; curseur via `TimingCallbacks` ; nom des notes par `speechSynthesis` ; modes continu et note à note.
- Décision **zéro pitch-shift** pour le futur kit vocal (un échantillon natif par note) : étirer un échantillon au-delà de ~2–3 demi-tons dégrade le timbre.
- Correctif nomenclature : le nom **chanté** est la syllabe seule, sans octave ; l'octave n'est conservé que pour les affichages de registre. Convention scientifique do central = do4 confirmée.
- Bibliothèques embarquées inline pour garantir le fonctionnement 100 % hors-ligne. Taille du livrable ≈ 674 Ko.

## Contributions

Signalements de bugs et suggestions bienvenus via les *issues*. Les *pull requests* sont relues et intégrées au cas par cas.

## Licence

MIT — réutilisation libre, l'auteur reste crédité.

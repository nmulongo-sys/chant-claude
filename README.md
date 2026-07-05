# Chant & Justesse

Application web autonome d'entraînement au déchiffrage chanté et à la justesse, conçue pour adapter automatiquement des exercices de formation musicale à une tessiture grave, et généralisée à tout chanteur via un kit vocal personnel.

**En ligne** : ouvrir `index.html` dans un navigateur (aucun serveur requis) ; déployé sur GitHub Pages.
**Statut** : 2026-07-05 — onglets Partition, Écoute, Chant (accordeur **et** mode exercice noté), Studio (kit vocal) fonctionnels ; onglet Bibliothèque à venir. Fichier HTML unique, bibliothèques embarquées, fonctionne hors ligne.

## Utilisation

Fichier unique, aucune installation, hors ligne après le premier chargement, pensé mobile-first (Android) et compatible desktop. **Le micro (onglets Chant et Studio) exige un contexte sécurisé** : servir la page en HTTPS (GitHub Pages convient) ; en ouverture `file://` locale, l'accès micro peut être bloqué selon le navigateur.

**Charger un exercice** (onglet Partition), par trois voies :
- fichier MusicXML (`.musicxml` / `.xml`) ou MusicXML compressé (`.mxl`) — typiquement exporté depuis PlayScore 2 ;
- fichier ABC (`.abc` / `.txt`) ;
- saisie ou collage direct dans la zone ABC, éditable (sert aussi à corriger les erreurs d'OCR musical ou de transcription).

**Adapter à ma voix** : régler la tessiture (note grave / note aiguë). L'app transpose l'exercice par octaves pour le faire entrer dans la tessiture, puis bascule la clé (sol / fa) pour rester lisible. Les deux décisions sont automatiques et débrayables (Auto / 8va↓ / 8va↑ ; Auto / Sol / Fa). L'**Échelle** visualise la tessiture (bande dorée) et l'ambitus de l'exercice après adaptation — vert s'il tient, corail s'il déborde, avec le nombre de demi-tons de débordement.

**Écouter** (onglet Écoute) : lecture audio, tempo 30–200 BPM réglable, mode continu ou note à note, nom des notes énoncé en français, curseur synchronisé sur la partition. La lecture utilise les échantillons du kit vocal quand ils existent, sinon la synthèse.

**S'accorder** (onglet Chant, mode **Accordeur**) : activer le micro et chanter une note tenue ; la jauge affiche la note détectée (syllabe + registre) et l'écart en cents (±50), avec une zone verte de tolérance. Bascule « Grave » (buffer élargi) pour fiabiliser la détection sous do3. Rien n'est enregistré ni transmis : l'analyse est locale.

**S'exercer** (onglet Chant, mode **Exercice**) : la partition adaptée défile à un tempo réglable après un décompte de 4 temps ; on chante chaque note, la jauge affiche en direct l'écart à la **cible** (et non au demi-tempéré le plus proche). À la fin, un **score** : pourcentage de notes justes, écart moyen en cents, décompte trop haut / trop bas / bonne note-autre octave / non chanté, et une pastille par note colorée selon le verdict. Métronome et note-guide sonore débrayables. Chaque tentative est archivée dans un **historique par exercice** (chiffres seulement ; le micro n'est jamais enregistré). Bouton « Rejouer » pour recommencer.

**Enregistrer sa voix** (onglet Studio) : la liste des notes découle de la tessiture. Pour chaque note, Enregistrer → la jauge valide en direct → à l'arrêt, l'app compare la hauteur médiane chantée à la cible et **auto-accepte si c'est juste** (tolérance ±30 cents, réglable 10–50), sinon propose de reprendre ou de garder quand même. Export/import du kit en JSON, bouton pour vider le kit.

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

**Détection de hauteur (YIN).** Module partagé par l'accordeur (Chant), le mode exercice et la validation des prises (Studio). Autocorrélation type YIN : fonction de différence → différence moyenne cumulée normalisée → premier creux sous seuil (0,12) → interpolation parabolique sub-échantillon. Entrée : `AnalyserNode.getFloatTimeDomainData`. `fftSize` = 2048 (plage normale) ou 4096 (plage « grave », plancher fiable ~65 Hz, couvre mi2 à 82 Hz). Filtrage : porte RMS (>0,008) + seuil de `clarity` (>0,90) pour rejeter le bruit/non-voisé ; lissage par médiane glissante sur 5 trames. Sortie : fréquence → MIDI + cents (`freqToMidiCents`). Validé en Node au cent près de mi2 à la4. Un bus d'abonnement (`onPitch`/`emitPitch`) diffuse chaque détection à tous les consommateurs (accordeur, exercice, Studio) sans les coupler.

**Jauge.** Composant réutilisé (Chant, exercice et Studio) : aiguille positionnée sur ±50 cents, zone verte = tolérance, note (syllabe) + registre + écart signé en cents ; vert dans la tolérance, sinon corail (trop haut) / ambre (trop bas). En mode exercice, l'aiguille exprime l'écart **à la cible** (voir ci-dessous), pas au demi-tempéré le plus proche.

**Mode exercice + notation.** Le moteur d'exercice réutilise le pipeline d'adaptation (même décalage/clé que Partition et Écoute) et le module YIN partagé. Séquençage :
- construction d'une **timeline** à partir de `state.events` : pour chaque événement, fenêtre `[on, off]` en secondes au tempo de l'exercice (`durSec@90 × 90/BPM`), cible = `midi + décalage` (les silences ont une cible nulle et comptent comme des pauses).
- toute l'horloge est celle de l'`AudioContext` ; décompte de 4 temps (bips), puis boucle `requestAnimationFrame` qui repère l'événement courant, surligne la note (`#score3`) et route les détections YIN vers le tampon de la note en cours (fenêtre de notation = 30 %→100 % de la durée, pour ignorer l'attaque/transition).
- **notation par note** (`exoGradeWindow`) : médiane des hauteurs détectées → écart brut en cents **arrondi à l'entier** (robustesse au flottant aux bornes) `raw = round((médiane − cible)×100)` ; repli d'octave `fold = raw − 1200·round(raw/1200)` (∈ [−600, 600]). Verdicts : `juste` si `|raw| ≤ tol` ; `octave` si la classe de hauteur est juste (`|fold| ≤ tol`) mais l'octave est fausse ; `haut`/`bas` selon le signe de `fold` si la classe est fausse ; `muet` si moins de 2 détections. Tolérance réglable ±15…60 cents (défaut 35).
- **score final** : `% justes = nJustes / nNotes` ; écart moyen = moyenne des `|fold|` sur les notes chantées ; décomptes par catégorie ; une pastille colorée par note.
- métronome et note-guide sonore optionnels ; le nom de la cible n'est **pas** énoncé à voix haute (déchiffrage à vue). À l'arrêt, le micro partagé est relâché (`pitchStop`) et l'état de l'accordeur remis à zéro.

**Cascade audio (résolution par note en lecture).** Ordre : kit vocal utilisateur (IndexedDB, échantillons décodés en mémoire) → synthèse (le niveau « kit embarqué » n'est pas encore présent). Les échantillons du kit sont joués **à leur hauteur native** (zéro pitch-shift), coupés à la durée de la note avec court fondu. La synthèse est un timbre vocal simplifié : deux oscillateurs `sawtooth` filtrés par deux passe-bande (~600 / ~1200 Hz) avec enveloppe ; hauteur via `midiToFreq = 440·2^((m−69)/12)`, donc **aucune limite d'octave** (fréquence calculée, pas d'échantillon étiré). Le nom de la note est dit via `speechSynthesis` (débrayable). Séquenceur sur l'horloge du contexte audio ; curseur via `TimingCallbacks`.

**Enregistrement et traitement (Studio).** Capture `MediaRecorder` sur le flux micro (`audio/webm;codecs=opus`, repli `audio/mp4` pour Safari ; type MIME testé via `isTypeSupported`). À l'arrêt : décodage `decodeAudioData` → sommation mono → **trim** du silence de tête/queue (seuil ~0,01, pré-roll 5 ms, traîne 20 ms) → **normalisation** crête −1 dBFS → **fondu de sortie 10 ms sans fondu d'attaque** (transitoire préservé). Stockage en **WAV PCM 16 bits mono** (format universellement redécodable, contrôle DSP exact). Validé en Node : 400 ms → ~224 ms, crête 0,8913, fin à zéro.

**Convention de nom de note.**
- **Nom chanté** (dit à voix haute et affiché comme note à chanter) : **syllabe seule, sans octave** — « mi », « do dièse ». On ne chante pas le numéro d'octave.
- **Registre** (sélecteurs de tessiture, axe de l'Échelle, bornes d'ambitus, jauge) : syllabe **+ numéro d'octave**, car là l'octave situe une hauteur et ne se chante pas.
- Convention d'octave : **scientifique, do central = do4** (MIDI 60). Tessiture par défaut mi2 → do4 (MIDI 40–60).

**Persistance.**
- Tessiture : `localStorage`, clé `cj_tess`, schéma `{ min, max }` (numéros MIDI).
- Kit vocal : `IndexedDB`, base `ChantJustesseDB` (**v2**), store `samples`, clé primaire = numéro MIDI ; enregistrement `{ midi, wav: Blob, mime: "audio/wav", date }`. Au démarrage et après chaque prise/import, les WAV sont décodés en `AudioBuffer` en mémoire pour la lecture. Export/import du kit en JSON (échantillons encodés base64).
- Scores d'exercice : même base `ChantJustesseDB` (v2), store `scores`, clé `id` auto-incrémentée, index `ex` (empreinte de l'ABC source) ; enregistrement `{ ex, title, date, pct, meanCents, nNotes, nJuste, tol, bpm }`. La migration v1→v2 crée le store `scores` sans toucher aux échantillons existants.

**Hors périmètre à ce stade** : OCR musical embarqué (la lecture de PDF reste externe, en amont) ; polyphonie chantée ; reconnaissance du nom chanté (seule la hauteur est évaluée) ; niveau « kit embarqué » de la cascade ; onglet Bibliothèque (les scores sont pour l'instant listés dans la carte exercice).

## Reste à faire (feuille de route)

Dans l'ordre de développement :
1. **Moteur audio v2** (passe unique de rendu audio, groupée pour ne toucher le séquenceur qu'une fois) :
   - enveloppe attaque/tenue/relâche **proportionnée à la durée** de la note (attaque brève partout, tenue/relâche qui s'étirent avec la note longue) ;
   - gestion des **liaisons** : lecture des *ties* (prolongation, même hauteur → fusionner en une note tenue) et *slurs* (expression, hauteurs différentes → legato sans ré-attaque, portamento éventuel) depuis le modèle ABC ;
   - **presets de synthèse voix homme / femme** (jeux de formants distincts sur les passe-bande) ; n'affecte que le repli synthèse, pas les échantillons enregistrés.
   - *Réserve* : la fusion des *ties* de prolongation est une correction (aujourd'hui deux notes liées de même hauteur sont ré-attaquées) ; à remonter avant si elle gêne le mode exercice.
2. **Onglet Bibliothèque.** Exercices sauvegardés (titre + ABC) et historique des scores consolidé (le store `scores` existe déjà), en IndexedDB.
3. **Kit embarqué.** Figer un kit vocal par défaut en base64 dans le HTML, comme niveau intermédiaire de la cascade (entre kit utilisateur et synthèse).

## Journal de développement

### 2026-07-05 — Mode Chant « exercice » + score
- Onglet **Chant** dédoublé : sélecteur **Accordeur / Exercice**. L'accordeur live existant est préservé intact.
- **Mode exercice** : la partition adaptée défile à l'horloge de l'`AudioContext` après un décompte de 4 temps ; timeline construite depuis `state.events` (fenêtre par note au tempo réglable) ; surlignage de la note courante sur `#score3` ; collecte des détections YIN partagées via `onPitch` sur la fenêtre 30 %→100 % de chaque note.
- **Notation par note** : médiane des hauteurs → écart en cents **arrondi** (robuste au flottant), repli d'octave `fold`. Verdicts `juste` / `octave` (bonne classe, mauvaise octave) / `haut` / `bas` / `muet`. Jauge live exprimant l'écart **à la cible**. Tolérance ±15–60 cents (défaut 35), tempo 30–140 BPM (défaut 66), métronome et note-guide débrayables.
- **Score final** : % de notes justes, écart moyen en cents, décomptes par catégorie, pastille colorée par note, bouton Rejouer.
- **Historique par exercice** : base `ChantJustesseDB` passée **v1→v2**, nouveau store `scores` (clé auto, index `ex` = empreinte de l'ABC). Migration validée en Node (fake-indexeddb) : les échantillons du kit v1 survivent, l'index filtre bien par exercice. Sauvegarde auto en fin d'exercice, tentatives récentes affichées dans la carte.
- Vérifications : logique de notation testée en Node (juste/haut/bas/octave/muet, bornes de tolérance), migration DB v2 et aller-retour des scores en fake-indexeddb, structure du fichier en jsdom (aucun id dupliqué, éléments présents, accordeur intact, dépendances définies) — 0 erreur.
- Taille du livrable ≈ 713 Ko.

### 2026-07-05 — Détection YIN, accordeur, Studio + IndexedDB
- Module de détection de hauteur **YIN** (autocorrélation + interpolation parabolique), validé en Node au cent près de mi2 (82 Hz) à la4 ; porte RMS + seuil de `clarity` + médiane glissante 5 trames ; plage « grave » à `fftSize` 4096.
- **Jauge** de justesse réutilisable (±50 cents, zone de tolérance, note + registre + cents).
- Onglet **Chant** transformé en accordeur live (micro `getUserMedia`, analyse locale, rien n'est enregistré).
- Onglet **Studio** : liste des notes issue de la tessiture, enregistrement guidé `MediaRecorder`, validation par la jauge (auto-accept si dans la tolérance ±30 cents réglable), traitement mono + trim + normalisation −1 dBFS + fondu de sortie 10 ms (sans fondu d'attaque), stockage **WAV 16 bits** en **IndexedDB** par numéro MIDI. Export/import du kit en JSON, vidage.
- **Cascade audio** recâblée : la lecture (Écoute, note à note) résout kit IndexedDB → synthèse ; échantillons joués à hauteur native (zéro pitch-shift).
- DSP (trim/fade/WAV) et couche IndexedDB validés en Node ; boot complet, persistance, lecture cascade et activation micro validés en environnement simulé (0 erreur).
- Décision de feuille de route : voix synthétique homme/femme et enveloppe/liaisons regroupées en une passe « moteur audio v2 », placée après le mode Chant exercice et avant le kit embarqué.
- Taille du livrable ≈ 700 Ko.

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

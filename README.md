# gladys-tuya-medion

Fork personnel, à usage strictement privé, de [Terdious/gladys-tuya](https://github.com/Terdious/gladys-tuya)
(intégration externe Tuya pour Gladys Assistant), avec des correctifs
spécifiques à un unique appareil : une climatisation de camping vendue sous
la marque **MEDION**.

**Ce fork n'est pas destiné à être publié dans le store Gladys ni réutilisé
par d'autres personnes** — les correctifs ci-dessous sont câblés en dur pour
un seul modèle précis et casseraient potentiellement le comportement
attendu sur d'autres appareils Tuya. Si vous tombez sur ce dépôt sans avoir
exactement le même appareil, utilisez plutôt l'intégration Tuya officielle
de Gladys.

## Appareil concerné

| | |
|---|---|
| **Nom commercial** | MEDION Smart mobile camping AC P502 |
| **Référence** | MD37735 |
| **Nom produit côté Tuya** | MEDION Camping AIR Conditioner P502 |
| **Catégorie Tuya** | `kt` (climatiseur) |
| **App associée** | MEDION Life+ (marque blanche Tuya) |

## Pourquoi ce fork existe

L'intégration Tuya officielle de Gladys fonctionne très bien pour la
majorité des climatiseurs Tuya, mais le firmware de ce modèle précis
déclare des choses incohérentes ou non-standard dans son schéma Tuya, ce
qui cassait plusieurs fonctionnalités une fois branché dans Gladys. Chaque
problème a été diagnostiqué en inspectant le schéma réel de l'appareil via
**Tuya IoT Platform → Devices → Device Debugging → Device Debugging (onglet)
→ Standard Status Set / DP Instruction**, puis comparé au comportement du
code de l'intégration.

## Les 5 correctifs, en détail

### 1. Température affichée à tort divisée par 10

**Symptôme :** la température ambiante affichait ex. `2.7°C` au lieu de
`27°C`. La consigne semblait correcte par coïncidence.

**Cause :** `src/devices/airConditioner.js` déclarait `scale: 1` par défaut
pour `temp_set` et `temp_current` (comportement standard chez beaucoup de
climatiseurs Tuya, qui stockent 27,0°C sous la forme entière `270`). Le
panneau **Device Debugging → Standard Status Set** de cet appareil précis
montre au contraire `"scale": 0` pour les deux — le firmware envoie déjà la
valeur réelle, sans multiplication par 10.

**Fichier modifié :** `src/devices/airConditioner.js` (`cloudMapping.temp_set.scale` et `cloudMapping.temp_current.scale`, passés de `1` à `0`).

### 2. Vitesse de ventilation : liste à 8 choix au lieu de 2, et écriture qui ne fait rien

**Symptôme :** le sélecteur de vitesse proposait 8 options (Auto, Faible,
Faible-Moyen, Moyen, Moyen-Fort, Fort, Silencieux, Turbo) alors que la clim
n'en a physiquement que 2. Sélectionner n'importe laquelle ne changeait
rien sur l'appareil.

**Cause :** le DP `windspeed` de cet appareil déclare un enum brut à seulement
deux valeurs numériques littérales, `"1"` et `"2"` (pas de libellés textuels
comme `"low"`/`"high"` que le code sait reconnaître nativement). Le
vocabulaire de traduction ne connaissait donc ni comment interpréter ces
valeurs en lecture, ni comment écrire dans ce format en commande.

**Fichier modifié :** `src/tuya/device/tuya.deviceMapping.js`
- Lecture (`TUYA_AC_FAN_SPEED_TO_GLADYS`) : ajout de `1: LOW` et `2: HIGH`.
- Écriture (`GLADYS_AC_FAN_SPEED_TO_TUYA`) : `LOW` envoie désormais la
  chaîne littérale `'1'` (au lieu de `'low'`), `HIGH` envoie `'2'` (au lieu
  de `'high'`).

⚠️ Ce correctif d'écriture est **global au fichier**, pas isolé par modèle
d'appareil (le code ne le permettait pas facilement sans réécriture plus
large) — c'est acceptable uniquement parce que ce fork ne sert qu'à cet
unique appareil.

### 3. Modes (Clim/Ventilation/Déshumidification/Chauffage) : aucun ne répondait

**Symptôme :** la clim démarrait et restait bloquée en mode Clim, quel que
soit le mode sélectionné dans Gladys.

**Cause :** le DP `mode` de cet appareil envoie ses valeurs en
**MAJUSCULES** (`COOL`, `FAN`, `DRY`, `HEAT`, `SLEEP`), confirmé via Device
Debugging. Le code d'origine utilisait des clés en minuscules
(`cold`/`heat`/`wet`/`fan`) pour la traduction — les clés d'un objet
JavaScript étant sensibles à la casse, aucune ne correspondait jamais,
aussi bien en lecture qu'en écriture.

**Fichier modifié :** `src/tuya/device/tuya.deviceMapping.js`
- Lecture (`TUYA_AC_MODE_TO_GLADYS`) : ajout de `COOL`, `FAN`, `DRY`, `HEAT`
  en majuscules.
- Écriture (`GLADYS_AC_MODE_TO_TUYA`) : envoie désormais `'COOL'`, `'HEAT'`,
  `'DRY'`, `'FAN'` (majuscules) au lieu des mots anglais génériques
  minuscules d'origine.

**Limitation connue, non liée au logiciel :** le mode Chauffage peut ne
jamais réchauffer réellement l'air, même une fois la commande correctement
envoyée — beaucoup de climatiseurs de camping bon marché déclarent un mode
`HEAT` dans leur logiciel sans avoir de vraie résistance chauffante
(fonctionnement uniquement par compresseur froid). Vérifiez physiquement
avant de considérer ça comme un bug.

**Mode `SLEEP` non intégré ici** : voir point 5.

### 4. Swing (balancier du clapet de sortie d'air) totalement absent

**Symptôme :** aucune option pour faire osciller le volet de sortie d'air,
alors que la télécommande physique de l'appareil le permet.

**Cause :** le DP nommé `Shake` dans le schéma Tuya (un simple booléen
on/off) n'était référencé nulle part dans le code d'origine — ni dans la
liste des codes gérés, ni dans le mapping local par numéro de DP.

**Numéro de DP retenu : 8.** Ce numéro n'a **pas** été confirmé via une
lecture brute directe du protocole local (aucun outil de type `tinytuya`
n'a été utilisé pour le vérifier expérimentalement) — il a été déduit par
recoupement :
1. l'ordre d'affichage des codes dans le tableau Tuya (`Power, temp_set,
   temp_current, mode, windspeed, C_F, Timer, Shake`) suggère une
   numérotation séquentielle 1 à 8 ;
2. le fichier d'origine listait déjà `'8'` dans ses DP "à ignorer"
   (`ignoredDps`), signe que ce numéro est un DP connu et récurrent sur
   plusieurs climatiseurs `kt` similaires, jamais exploité jusqu'ici.

**Vérifié positivement en usage réel** : le clapet bouge bien physiquement
quand on actionne l'interrupteur "Swing (balancier)" dans Gladys — la
déduction s'est révélée correcte pour cet appareil précis.

**Fichiers modifiés :**
- `src/devices/airConditioner.js` : ajout du code `shake` dans
  `AIR_CONDITIONER_CODES`, dans `cloudMapping` (catégorie `SWITCH`, type
  `BINARY`), et `dps.shake = 8` dans `localMapping` (avec retrait du `'8'`
  de `ignoredDps`).

### 5. Mode nuit (SLEEP) : valeur invisible dans l'interface Gladys

**Symptôme :** impossible d'activer le mode SLEEP de la clim depuis
Gladys, alors que l'appareil le supporte (`SLEEP` fait partie des valeurs
possibles du DP `mode`, confirmé via Device Debugging).

**Cause, structurelle et non contournable simplement :** le type de
fonctionnalité `AIR_CONDITIONING.MODE` du **cœur de Gladys lui-même**
(pas de cette intégration) ne connaît que 5 valeurs fixes : `AUTO`,
`COOLING`, `HEATING`, `DRYING`, `FAN`. Il n'existe aucun emplacement
numérique pour une 6ᵉ valeur "Sleep" dans l'interface native — l'ajouter
directement dans le sélecteur de mode aurait affiché un bouton sans
libellé et un comportement non garanti.

**Solution retenue :** exposer le mode nuit comme un **interrupteur
séparé** ("Mode nuit (Sleep)"), distinct du sélecteur de mode normal,
plutôt que comme une 6ᵉ option de ce sélecteur :
- **Activé** → envoie littéralement `SLEEP` sur le DP `mode`.
- **Désactivé** → envoie `COOL` (retour au mode Clim par défaut — pas de
  notion de "mode précédent" mémorisé, c'est une simplification assumée).

Techniquement, cette fonctionnalité est **synthétique** : elle ne
correspond à aucun DP Tuya déclaré séparément, elle réutilise le DP 4
(`mode`) sous un nom de code différent (`sleep_toggle`), avec sa propre
logique de lecture/écriture dédiée (voir ci-dessous) qui contourne le
tableau de traduction générique (partagé avec le vrai interrupteur
marche/arrêt et le swing) pour éviter tout conflit.

**Fichiers modifiés :**
- `src/tuya/device/tuya.convertDevice.js` : après la construction normale
  des fonctionnalités, injection d'une fonctionnalité `sleep_toggle`
  supplémentaire — **uniquement** si `product_name` ou `model` contient la
  chaîne `P502` (protection pour ne jamais ajouter ce bouton sur un autre
  climatiseur Tuya que celui-ci, même si le code venait à être réutilisé
  par erreur).
- `src/devices/airConditioner.js` : `dps.sleep_toggle = 4` dans
  `localMapping` (même DP que `mode`).
- `src/tuya/tuya.poll.js` (`getFeatureReader`) : cas spécial pour le code
  `sleep_toggle` — lit la valeur brute du DP `mode` et retourne `1` si elle
  vaut exactement `'SLEEP'`, sinon `0`.
- `src/tuya/tuya.setValue.js` (`setValue`) : cas spécial pour le code
  `sleep_toggle` — contourne le tableau de traduction générique (partagé
  avec les vrais interrupteurs SWITCH/BINARY) et envoie directement
  `'SLEEP'` ou `'COOL'` selon la valeur booléenne reçue.

## Comment appliquer ces correctifs à un autre appareil similaire

Si vous avez un climatiseur Tuya différent avec des symptômes similaires :

1. Allez sur **Tuya IoT Platform → Cloud → votre projet → Devices → Device
   Debugging**, sélectionnez votre appareil, onglet **"Standard Status
   Set"** pour voir le schéma déclaré (scale, min/max, enum range) de
   chaque code, et l'écran **"Configure Control Instruction Mode" →
   "DP Instruction"** (accessible depuis Product Details) pour voir la
   correspondance entre codes et numéros de DP bruts.
2. Comparez avec les valeurs codées en dur dans
   `src/devices/airConditioner.js` et `src/tuya/device/tuya.deviceMapping.js`.
3. Adaptez chaque correctif à vos propres valeurs observées — **ne
   réutilisez jamais telles quelles** les valeurs de ce fork (scale, DP 8,
   chaînes `COOL`/`1`/`2`...), elles sont spécifiques au MEDION P502.

## Comment déployer une modification (rappel opérationnel)

```bash
# 1. Modifier le code, puis valider la syntaxe
node --check src/devices/airConditioner.js   # (et tout autre fichier modifié)

# 2. Lancer les tests (voir section "Tests" ci-dessous)
npm install && npm test

# 3. Reconstruire l'image AVANT de toucher à Gladys
docker build -t ghcr.io/vv69004/gladys-tuya-medion:1.12.0 .

# 4. Committer et pousser
git add -A && git commit -m "..."
git push

# 5. Dans Gladys : désinstaller complètement "Tuya (patch MEDION P502)",
#    PUIS relancer l'étape 3 (rebuild) juste avant de réinstaller
#    (désinstaller supprime l'image locale : sans rebuild juste avant,
#    la réinstallation échoue avec une erreur "registry: denied").
```

### ⚠️ Piège n°1 : `docker restart` ne suffit JAMAIS pour appliquer un changement de code

Un conteneur déjà créé reste figé sur l'image telle qu'elle était **au
moment de sa création**. Reconstruire l'image avec le même tag ne le met
pas à jour tant qu'il n'est pas recréé de zéro :

```bash
docker stop gladys-ext-dev-tuya-patch-medion-p502
docker rm gladys-ext-dev-tuya-patch-medion-p502
```

Puis rouvrir l'écran de configuration de l'intégration dans Gladys (ou
sauvegarder sans rien changer) pour qu'il recrée le conteneur depuis
l'image fraîche.

### ⚠️ Piège n°2 : désinstaller l'intégration supprime l'image Docker locale

Si vous **désinstallez complètement** l'intégration depuis Gladys (pas
juste stop/rm le conteneur), Gladys supprime aussi l'image Docker locale
associée. Une réinstallation immédiate après tentera alors de re-télécharger
l'image depuis `ghcr.io` (qui n'existe pas là-bas, cette image n'a jamais
été publiée) et échouera avec `registry: denied`. **Reconstruisez toujours
l'image juste avant de réinstaller**, jamais après.

### ⚠️ Piège n°3 : la liste des actions (`actions` du manifeste) est figée à l'installation

Contrairement au code JS (qui se met à jour dès que le conteneur est
recréé), la liste des actions personnalisées affichées dans l'onglet
Configuration semble être mise en cache par Gladys au moment de
l'installation. Modifier `actions` dans le manifeste nécessite de
désinstaller/réinstaller complètement l'intégration, pas juste de recréer
le conteneur.

## Tests

```bash
npm install
npm test
```

**4 tests échouent, et c'est normal et attendu** — ils font partie de la
suite d'origine du projet Terdious/gladys-tuya et vérifient le comportement
**générique** (scale=1 par défaut, mots anglais minuscules pour les modes)
que ce fork modifie **volontairement** pour ce modèle précis :

- `poll restores the scale lost by Gladys persistence (cloud read)`
- `poll restores the scale lost by Gladys persistence (local read)`
- `setValue restores the scale lost by Gladys persistence`
- `setValue writes the AC mode and the scaled setpoint to the cloud`

Les 2 tests dédiés au type d'appareil AC (`convertDevice maps the supported
AC features...`) passent bien, grâce à la protection par `product_name`
qui empêche le correctif "Mode nuit" de s'appliquer aux appareils
génériques utilisés par ces tests.

## Configuration Gladys

| Champ | Valeur |
|---|---|
| Endpoint | Central Europe |
| Client ID / Secret | depuis votre projet Tuya IoT Cloud (`iot.tuya.com`) |
| App account UID | UID du compte Smart Life/Tuya lié au projet (onglet Devices → Link App Account) |

⚠️ Le data center du projet Tuya IoT Cloud doit **impérativement**
correspondre à celui de votre compte Smart Life/Tuya (France = Central
Europe), sinon la liaison de compte échoue avec l'erreur "Data centers
inconsistency".

## Licence

Comme le projet d'origine [Terdious/gladys-tuya](https://github.com/Terdious/gladys-tuya).

# WLED en 256 × 64 (fork)

Fork de [WLED](https://github.com/wled/WLED) qui accepte une matrice 2D de **256 pixels** de large, par exemple **deux panneaux HUB75 de 128 × 64 chaînés**.

> **In English:** this fork accepts a 2D matrix dimension of 256 pixels (e.g. two chained 128×64 HUB75 panels). Upstream rejects any dimension above 255 and falls back to a 1D strip. Build the `wide-matrix` branch with `pio run -e esp32s3dev_16MB_opi_hub75`. The maximum is deliberately 256 and only one ESP32-S3 board was tested. Developed with AI assistance.

📖 **[Page du projet : explications et photos](https://kenny3231.github.io/WLED/)**
🔀 Proposition au projet WLED : [pull request wled/WLED#5996](https://github.com/wled/WLED/pull/5996), ouverte et à relire par les mainteneurs

*Fork personnel, sans lien officiel avec le projet WLED. La documentation de WLED d'origine reste dans [`readme.md`](../readme.md).*

## Pourquoi ce fork

> Je voulais mettre en place deux panneaux LED de 128 × 64. Je n'ai pas pu aller plus loin : les réglages m'interdisaient de saisir 256 et, quand j'y parvenais, la valeur n'était pas prise en compte après un redémarrage. J'en ai déduit que certaines valeurs étaient codées sur 8 bits.

Un nombre sur 8 bits va de 0 à 255 : assez pour numéroter 256 colonnes, pas pour écrire « 256 ». WLED avait en plus un contrôle qui refusait toute dimension supérieure à 255 et repassait en mode 1D.

## Ce qui change

Deux commits de code ([voir le diff complet](https://github.com/wled/WLED/compare/main...Kenny3231:WLED:matrix-256)) : 6 fichiers, +66 / −25 lignes. Le second corrige des points relevés par la relecture automatique de la pull request (CodeRabbit). Aucune fonction de WLED n'est retirée.

| Fichier | Changement |
|---|---|
| `wled00/FX_2Dfcn.cpp` | Limite de la matrice 2D portée de 255 à 256 |
| `wled00/FX.h` | Taille des panneaux et zone de découpe verticale des transitions en 16 bits |
| `wled00/FX_fcn.cpp` | Même découpe en 16 bits ; taille lue dans un ledmap plafonnée à 256 ; ledmap trop grand pour un compteur 16 bits refusé, la matrice des panneaux est conservée |
| `wled00/FX_2Dfcn.cpp` (second commit) | Des valeurs de panneau absurdes ne peuvent plus contourner le contrôle et écrire hors de la mémoire réservée |
| `wled00/data/settings_2D.htm` | La page 2D accepte 256 |
| `wled00/FX.cpp` | Effet Octopus : échelle jamais nulle au-delà de 180 pixels, rayon plafonné à 255 |
| `wled00/ws.cpp` | Aperçu en direct : dimensions inférieures à 256 et conformes aux données envoyées, quelle que soit la taille |

Hormis le symptôme décrit ci-dessus, ces défauts ont été repérés en lisant le code.

## Installation

**Matériel testé :** ESP32-S3, 16 Mo de flash, 8 Mo de PSRAM octale, brochage HUB75 « MoonHub75 », deux panneaux 128 × 64 chaînés.

> ⚠️ Alimentez les panneaux avec une alimentation 5 V adaptée, pas par la carte ESP32.

Prérequis : [PlatformIO](https://platformio.org/) et Node.js. Sous Windows, compilez depuis PowerShell ou l'invite de commandes, pas depuis Git Bash.

```bash
git clone -b wide-matrix https://github.com/Kenny3231/WLED.git
cd WLED
npm ci
pio run -e esp32s3dev_16MB_opi_hub75 -t upload
```

**Utilisez un dossier PlatformIO séparé.** La plateforme ESP32 de WLED a cassé, lors des essais, une installation PlatformIO existante dans le dossier par défaut. Définissez la variable `PLATFORMIO_CORE_DIR` avant `pio run` :

```powershell
$env:PLATFORMIO_CORE_DIR = "C:\pio-wled"
```

```bash
export PLATFORMIO_CORE_DIR="$HOME/.pio-wled"   # Linux, macOS (non testé)
```

Les essais ont été faits sous Windows avec un tel dossier séparé.

Sauvegardez vos réglages et préréglages avant de flasher une carte déjà configurée.

### Réglages

1. **LED & Hardware** : type **HUB75 (Half Scan)** (selon le balayage de vos panneaux), taille d'un panneau `128 × 64`, `2` panneaux, `1` ligne × `2` colonnes.
2. **2D Configuration** : **un seul** panneau 2D de `256 × 64`, décalage `0 × 0`.
3. Enregistrez, puis **redémarrez**.

> Après un enregistrement sur la page 2D, la couleur du segment peut devenir noire ; un redémarrage l'a rétablie lors des tests. Observé aussi en 128 × 64.

## Tests

- Matrice 256 × 64 conservée après redémarrage ; image correcte constatée sur les deux panneaux.
- Après le second commit : aperçu en direct vérifié au pixel près (256 × 64, 256 × 3, 63 × 130) ; valeurs de panneau absurdes refusées sans plantage ; ledmap 256 × 64 accepté ; ledmaps trop grands refusés en conservant la matrice 256 × 64 ; 8 préréglages lancés sans redémarrage.
- Sur la première version publiée, avant le second commit : 218 effets lancés à la suite en 256 × 64 avec des transitions variées, sans redémarrage ni erreur (mémoire libre minimale : environ 102 Kio). Trois effets sont restés noirs dans l’aperçu (Image, Copy Segment, Sunrise) ; la cause probable (fichier image, second segment, démarrage sombre) n’a pas été vérifiée. Oscillate ne remplissait qu’une moitié sur la mesure courte ; mesuré plus longtemps sur une version antérieure, il couvrait les deux.
- `npm test` : 16 tests réussis sur 16.
- Réglages logiciels essayés sans changer les panneaux : 128 × 64, puis 64 × 256, 256 × 16 et 256 × 4 (ces trois derniers sur une version antérieure, différente d'une ligne dans Octopus).

## Limites connues

- Une seule carte et un seul environnement de compilation testés.
- Le balayage des effets détecte un écran noir ou coupé en deux, pas un rendu déformé.
- Non testés : plusieurs panneaux 2D séparés, dalle physique de 256 de haut, pleine luminosité sur une longue durée.
- Certains effets lourds sont peu fluides (environ 8 à 10 images par seconde, mesuré sur une version antérieure).
- La limite est volontairement de 256.

## Branches

| Branche | Rôle |
|---|---|
| `wide-matrix` | Branche à utiliser : WLED, le correctif et ce README |
| `matrix-256` | Le correctif seul, proposé au projet WLED |
| `gh-pages` | La page web et les photos |
| `main` | Copie de WLED, sans ce correctif |

### Mettre à jour le fork avec les nouveautés de WLED

```bash
git remote add upstream https://github.com/wled/WLED.git   # une seule fois
git fetch upstream
git switch wide-matrix
git merge upstream/main
```

Recompilez et retestez avant de flasher.

## Transparence sur l'IA

Le besoin, le symptôme et la vérification sur les panneaux viennent de l'auteur du fork. L'analyse du code, les modifications, les tests automatisés et cette documentation ont été réalisés avec l'aide d'un assistant IA (Claude Code).

## Licence

WLED est distribué sous licence [EUPL-1.2](../LICENSE), qui s'applique aussi à ce fork. Tout le mérite de WLED revient à ses auteurs et à sa communauté.

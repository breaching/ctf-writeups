# Tortoise Say - Walkthrough

## Énoncé

> Après Shrimp Say, voici Tortoise Say !
>
> Tortoise Say est un système embarqué révolutionnaire utilisant comme base la technologie e-Ink. Une tortue dessinée s'empresse de vous afficher un secret.
>
> Nous vous donnons une capture effectuée à l'analyseur logique de la communication avec l'écran lorsque la tortue était en train d'afficher le flag. L'écran utilisé est un Waveshare 2.9inch e-paper V2.
>
> Les broches mesurées sur l'analyseur logique sont :
> - D0 : DIN
> - D1 : CLK
> - D2 : CS
> - D3 : DC
> - D4 : RST
> - D5 : BUSY
>
> Arriverez-vous à retrouver le secret de la tortue ?

**Points :** 429  
**Catégorie :** Hardware / Forensique

## Fichier fourni

- `tortoise-say.vcd` : capture d'analyseur logique au format Value Change Dump (VCD)

## Analyse

### Format VCD

Le fichier `.vcd` est un format standard pour les captures d'analyseur logique. Il enregistre les changements d'état de chaque signal au fil du temps.

La capture contient 6 signaux SPI pour communiquer avec l'écran e-paper :
- **DIN** (D0) : données série (MOSI)
- **CLK** (D1) : horloge SPI
- **CS** (D2) : chip select (actif bas)
- **DC** (D3) : Data/Command selector (0=commande, 1=données)
- **RST** (D4) : reset
- **BUSY** (D5) : signal BUSY de l'écran

### Protocole SPI

Le Waveshare 2.9inch e-paper V2 utilise SPI mode 0 (CPOL=0, CPHA=0) :
- Données échantillonnées sur le front montant de CLK
- MSB en premier
- Le signal DC distingue les octets de commande (DC=0) des données (DC=1)

**Commandes clés identifiées :**
- `0x11` : Data Entry Mode (0x03 = X increment, Y increment)
- `0x24` : Write Black RAM (données d'image noir/blanc)
- `0x26` : Write Red RAM (deuxième buffer)
- `0x20` / `0x22` : Master Activation / Display Update Control 2
- `0x12` : Soft Reset
- `0x21` : Display Update Control 1

### Résolution de l'écran

L'écran Waveshare 2.9inch V2 a une résolution de **128×296 pixels** en orientation portrait :
- Largeur : 128 pixels → 16 octets par ligne
- Hauteur : 296 pixels → 296 lignes
- Taille totale d'une frame : 128×296/8 = 4736 octets

### Extraction des frames

En parsant le VCD et en décodant le SPI, on obtient **8 occurrences** de la commande `0x24` (Write Black RAM), chacune avec 4736 octets. Ces 8 frames correspondent aux rafraîchissements successifs de l'écran (affichage d'un texte défilant).

Frame 0 est entièrement blanche (initialisation), les frames 1-7 contiennent l'image.

### Orientation de l'image

L'écran est monté en **mode paysage** (physiquement tourné de 90°), mais les données sont stockées en portrait (128×296). Pour lire le texte correctement, il faut appliquer une rotation de **90° dans le sens anti-horaire** sur chaque frame.

Après rotation : 296×128 pixels avec le texte en orientation normale.

### Contenu des frames

Les 7 frames non-vides (après rotation 90° CCW) affichent un texte défilant à raison de **11 caractères par frame** :

| Frame | Texte affiché |
|-------|---------------|
| 1 | `Hello! I'm ` |
| 2 | `a tortoise!` |
| 3 | `FCSC{49a3ef` |
| 4 | `e9bbf4f610b` |
| 5 | `05a133ad615` |
| 6 | `6ba7080c35}` |
| 7 | ` Good Luck!` |

Le message complet est : **"Hello! I'm a tortoise!FCSC{...} Good Luck!"**

### Décodage du bitmap

L'écran utilise une police bitmap serif à taille variable (~7-9 pixels de largeur, ~10 pixels de hauteur). Les caractères ambigus ont été vérifiés manuellement :

- `3` vs `s` : deux arcs s'ouvrant vers la **gauche** = `3` (chiffre hexadécimal valide)
- `9` vs `g` : boucle fermée en haut avec queue courbée vers la gauche = `9` (font serif)
- `f` vs `t` : curl en haut à droite + 2 barres horizontales (barre + empattement) = `f`
- `1` : flag angled en haut gauche + empattement large en bas = `1`

Tous les caractères à l'intérieur des accolades sont des **chiffres hexadécimaux valides** (`0-9`, `a-f`).

## Flag

```
FCSC{49a3efe9bbf4f610b05a133ad6156ba7080c35}
```

## Scripts de résolution

### `parse_vcd.py`
Parse le fichier VCD et extrait les transactions SPI avec le signal DC pour distinguer commandes et données.

### `render_epaper.py`
Décode le flux SPI, extrait les frames de la RAM `0x24`, et génère des images PNG 128×296. Il faut ensuite faire pivoter les images de 90° CCW pour les lire correctement.

```python
from PIL import Image
import os

# ... (voir render_epaper.py)

# Après génération des frames :
for frame_idx in range(1, 8):
    img = Image.open(f'frame24_{frame_idx:02d}.png')
    img_rot = img.rotate(90, expand=True)  # 90° CCW
    img_rot.save(f'frame24_{frame_idx:02d}_rotated.png')
```

L'OCR (Tesseract) et le décodage manuel du bitmap permettent d'extraire le flag des frames 3 à 6.

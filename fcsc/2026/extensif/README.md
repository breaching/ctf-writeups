# Extensif - Walkthrough

## Énoncé

> Mais que se passe-t-il avec mes registres ? Je ne comprends plus
>
> **479 points** - Reverse

Fichier fourni : `extensif.bin`

---

## Analyse

### Identification du fichier

```
$ xxd -l 16 extensif.bin
00000000: e906 0210 e811 0840 ee00 0000 0000 0000  .......@........
```

- Magic `0xe9` → image flash ESP32
- Entry point : `0x400811e8`
- Architecture : **Xtensa LX6** (ESP32), little-endian

### Segments

| Segment | Chargement | Taille | Contenu |
|---------|-----------|--------|---------|
| DROM    | `0x3f400020` | `0x9aa8` | chaînes, données RO |
| DRAM    | `0x3ffb0000` | `0x23e8` | BSS/data RW |
| IROM1   | `0x400d0020` | `0x13834` | code applicatif |
| IROM2   | `0x40084158` | `0x8e58` | libc/ROM |

### Chaînes intéressantes (DROM)

```
0x3f4036a4  "Bienvenue"
0x3f4036b0  "Entrez le flag (16 chars): "
0x3f4036cc  "Erreur de lecture"
0x3f4036e0  "\n[-] Essaie encore\n"
0x3f4036f8  "\n[+] FLAG CORRECT! Vous pouvez valider l'épreuve avec FCSC{%s}\n"
0x3f407b04  (16 octets binaires = clé XOR)
```

---

## Vulnérabilités / mécanisme

### Désassemblage avec Capstone 6 (CS_ARCH_XTENSA)

La fonction principale (`0x400d6000`) effectue les opérations suivantes :

#### 1. Pré-chargement du buffer d'entrée (`0x3ffb0368`)

La boucle `0x400d6011-0x400d603a` écrit `"FCSCFCSCFCSCFCSC"` dans les 16 octets du buffer :

```asm
0x400d6011  slli  a9, a10, 2        ; a9 = 4*a10
0x400d6014  l32r  a8, [input_buf]   ; a8 = 0x3ffb0368
0x400d6017  addx4 a11, a10, a8      ; a11 = input_buf + 4*a10
0x400d601a  movi.n a12, 'F'         ; s8i 'F' → [a11+0]
0x400d601c  s8i   a12, a11, 0
...
0x400d603a  blti  a10, 4, . -0x29   ; boucle 4 fois
```

#### 2. Remplissage basé sur les **adresses de retour Xtensa** (le cœur du challenge)

La fonction `0x400d60e4` appelle une fonction récursive `0x400d60f4` avec `a10 = 0x4a ('J')` et `a11 = input_buf` :

```asm
; 0x400d60f4 - fonction récursive
entry a1, 0x10
bbci  a2, 3, retw.n       ; si bit3(a2) == 0 → stop (a2 = 0x50 = 'P')
s8i   a2, a3, 0           ; buf[pos]   = a2  ('J'..'O')
s8i   a0, a3, 1           ; buf[pos+1] = low_byte(a0)  ← RETURN ADDRESS!
addi.n a11, a3, 2
addi   a10, a2, 1
call8  . (→ 0x400d60f4)   ; appel récursif
retw.n
```

**Mécanisme clé :** `a0` contient l'adresse de retour courante (convention d'appel Xtensa windowed). `s8i a0, a3, 1` stocke l'octet de poids faible de cette adresse dans le buffer.

| Itération | `a2` | `buf[pos]` | Adresse de retour | `buf[pos+1]` |
|-----------|------|-----------|-------------------|-------------|
| 1 | `0x4A` ('J') | `0x4A` | `0x400d60ef` (après `call8` à `0x400d60ec`) | `0xEF` |
| 2 | `0x4B` ('K') | `0x4B` | `0x400d6108` (après `call8` à `0x400d6105`) | `0x08` |
| 3 | `0x4C` ('L') | `0x4C` | `0x400d6108` | `0x08` |
| 4 | `0x4D` ('M') | `0x4D` | `0x400d6108` | `0x08` |
| 5 | `0x4E` ('N') | `0x4E` | `0x400d6108` | `0x08` |
| 6 | `0x4F` ('O') | `0x4F` | `0x400d6108` | `0x08` |
| 7 | `0x50` ('P') | *(stop - bit3=0)* | - | - |

Résultat : `input_buf = 4A EF 4B 08 4C 08 4D 08 4E 08 4F 08 46 43 53 43`  
(les 4 derniers octets `46 43 53 43` = `FCSC` sont issus du pré-chargement)

#### 3. Boucle XOR (`0x400d6045-0x400d6064`)

```asm
l32r  a9, [expected_ptr]  ; a9 = 0x3f407b04
l8ui  a10, a9+a8, 0       ; a10 = expected[i]
l32r  a9, [input_buf]
l8ui  a11, a9+a8, 0       ; a11 = input_buf[i]
xor   a10, a10, a11       ; a10 = expected[i] ^ input_buf[i]
s8i   a10, SP+0x20+a8, 0  ; stack[i] = résultat
```

Ce résultat est stocké sur la pile **avant** que l'utilisateur saisisse quoi que ce soit.

#### 4. Lecture et vérification

La fonction de vérification (`0x400d8048`) compare ensuite l'entrée utilisateur avec le résultat XOR stocké sur la pile. Si l'entrée correspond, le flag est affiché.

---

## Calcul du flag

```
expected (DROM 0x3f407b04) : 79 8D 2D 3E 2E 6E 79 6D 28 38 2D 38 77 75 60 21
input_buf (programme)      : 4A EF 4B 08 4C 08 4D 08 4E 08 4F 08 46 43 53 43
                             XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR XOR
flag content               : 33 62 66 36 62 66 34 65 66 30 62 30 31 36 33 62
                           = "3bf6bf4ef0b0163b"
```

---

## Exploit

```bash
cd extensif
python3 solve.py
```

Sortie :
```
input_buf (programme) : 4aef4b084c084d084e084f0846435343
expected (DROM)       : 798d2d3e2e6e796d28382d3877756021
XOR result (flag)     : 33626636626634656630623031363362

[+] FLAG : FCSC{3bf6bf4ef0b0163b}
```

---

## Flag

```
FCSC{3bf6bf4ef0b0163b}
```

---

## Résumé

Le challenge exploite le mécanisme de **fenêtres de registres Xtensa** (`a0` = adresse de retour avec bits de rotation de fenêtre encodés). La fonction récursive `0x400d60f4` n'utilise pas le registre `a0` comme simple valeur : elle en extrait l'octet de poids faible comme partie du "flag attendu", ce qui change d'une adresse de retour à l'autre selon l'emplacement des instructions `call8` dans le firmware. C'est le sens du titre **Extensif** (jeu de mots sur **Xtensa**) et de la phrase *"Mais que se passe-t-il avec mes registres ?"*.

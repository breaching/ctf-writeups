# RemoBLE Control - Walkthrough

## Énoncé

> Armé de votre fidèle nRF52840 et du projet WHAD, vous avez capturé les échanges Bluetooth Low Energy entre un clavier sans-fil et un ordinateur. Que se sont-ils raconté ?
>
> La ligne de commande utilisée pour chaque capture est la suivante :
> `wsniff --interface uart0 --format=raw --no-metadata ble -f`
>
> Note : Le clavier utilisé est configuré en AZERTY.

**Catégorie :** Hardware / Bluetooth  
**Points :** 498  
**Fichiers :** `session1.txt`, `session2.txt`

---

## Cheminement de pensée

### 1. Identifier le format des fichiers

Les fichiers sont deux captures BLE brutes au format WHAD `--format=raw --no-metadata`. Chaque ligne est un paquet BLE encodé en hexadécimal, sans timestamp ni métadonnées. Le format est :

```
[Access Address (4 bytes)] [PDU Header (1)] [Length (1)] [Payload (length bytes)] [CRC (3)]
```

On reconnaît d'abord les premiers paquets de `session1.txt` :
- **L1** (`hdr=0x25`) : paquet de type `ADV`/`CONNECT_IND` - on voit les adresses InitA et AdvA dans le payload
- **L3-L8** : échanges `L2CAP SMP` (CID=0x0006) - paquets Pairing Request/Response
- **L10-L13** : `SMP Pairing Confirm` puis `SMP Pairing Random`

→ C'est un appairage BLE complet, suivi d'une phase chiffrée.

### 2. Reconnaître le mode d'appairage : Just Works

Le `Pairing Request` (L6) et le `Pairing Response` (L8) contiennent les capacités d'I/O. En les parsant :
- `IO Capability = 0x03` (NoInputNoOutput)
- Pas de MITM demandé

→ Mode **Just Works** → **Temporary Key (TK) = 0x00000000...00** (16 zéros).

C'est la vulnérabilité centrale : avec TK=0, toute la hiérarchie de clés est calculable.

### 3. Calculer la STK (Short Term Key)

La spec BLE LE Legacy Pairing définit :
```
STK = s1(TK, Srand[0:8] || Mrand[0:8])
```
où `s1(k, r) = AES_128(k, r)` avec un ordre d'octets little-endian (fonction `e`).

On extrait Mrand (L12) et Srand (L13) des paquets `SMP Pairing Random`, et on calcule la STK.

### 4. Dériver la SK de session 1

Le paquet `LL_ENC_REQ` (L14, opcode LL Control 0x03) contient `SKDm` et `IVm`.  
Le paquet `LL_ENC_RSP` (L15, opcode 0x04) contient `SKDs` et `IVs`.

La clé de session BLE-CCM est :
```
SK = AES_128(STK, SKDm || SKDs)   # little-endian interne (fonction e)
IV = IVm || IVs
```

### 5. Déchiffrer session1 → extraire le LTK

Avec SK et IV, on peut déchiffrer les paquets BLE-CCM de session1. Le chiffrement est AES-CCM avec :
- Nonce 13 octets : `[counter_LE(5 bytes, msb = direction)] [IV(8 bytes)]`
- AAD : 1 octet = `header & 0x07`
- MIC : 4 octets à la fin du payload chiffré

En brute-forçant les compteurs (incrémentés séparément par direction M→S et S→M), on trouve les paquets SMP de distribution des clés :
- **SMP Encryption Information** (opcode 0x06, S→M) : contient le **LTK** sur 16 octets

**LTK extrait : `ccc8f759678234ece3467a0b1dad9b07`**

### 6. Dériver la SK de session 2 et déchiffrer le HID

`session2.txt` est une session ultérieure utilisant le LTK (device déjà bondé). Le processus est identique :
- LL_ENC_REQ (L12) : `SKDm2`, `IVm2`
- LL_ENC_RSP (L13) : `SKDs2`, `IVs2`
- `SK2 = AES_128(LTK, SKDm2 || SKDs2)`

En déchiffrant les paquets S→M valides (MIC OK), on trouve des notifications ATT (`opcode 0x1b`) sur le handle `0x0011` = handle HID keyboard report.

### 7. Décoder les keycodes AZERTY

Un rapport HID clavier boot (8 octets) :
```
[modifier(1)] [reserved(1)] [keycode1..6(6)]
```

Le clavier est AZERTY donc les keycodes HID (positions physiques US QWERTY) correspondent à des touches différentes. Exemples critiques pour le flag :
- `0x09` + `modifier=0x02` (Left Shift) → `F` (position physique `f` sur AZERTY = key `F` shiftée)
- `0x06` + `modifier=0x02` → `C`
- `0x16` + `modifier=0x02` → `S`
- `0x21` + `modifier=0x40` (AltGr) → `{` (touche `'` + AltGr sur AZERTY)
- `0x2e` + `modifier=0x40` → `}` (touche `=` + AltGr)
- `0x2a` → `[Backspace]`

### 8. Simuler le buffer de frappe

En rejouant la séquence de touches avec gestion des backspaces :

```
[BS] e [BS] e [BS]          ← tentatives avortées (typos initiales)
F C S C                     ← "FCSC" tapé (puis typo)
' (                         ← touche ' puis ( au lieu de { avec AltGr
[BS]×7                      ← 7 backspaces pour tout effacer
F C S C { b b e b 5 b 8 8 3 6 d 3 c b 2 b 5 5 e a e 2 f f 0 f d b 5 8 }
[Enter]
```

→ Flag soumis : **`FCSC{bbeb5b8836d3cb2b55eae2ff0fdb58}`**

---

## Structure des captures

### session1.txt - Appairage BLE

```
L01: CONNECT_IND (ADV) - InitA + AdvA
L02: LL_FEATURE_REQ
L03: ATT Exchange MTU
L04-L05: ATT Discovery
L06: SMP Pairing Request  (IO=0x04/NoInputNoOutput, OOB=0, MITM=0)
L07: ATT Read
L08: SMP Pairing Response (IO=0x03/NoInputNoOutput, OOB=0, MITM=0)
L09: LL_VERSION_IND
L10: SMP Pairing Confirm (master Mconfirm)
L11: SMP Pairing Confirm (slave  Sconfirm)
L12: SMP Pairing Random  (master Mrand)
L13: SMP Pairing Random  (slave  Srand)
L14: LL_ENC_REQ  → SKDm=b1b2567d565140c5  IVm=7c81b0bc
L15: LL_ENC_RSP  → SKDs=83a73da4e103edf1  IVs=675dbe4b
L16+: Trafic BLE-CCM chiffré avec STK → distribution LTK
```

### session2.txt - Session HID chiffrée

```
L01-L11: Connexion + discovery ATT (clair)
L12: LL_ENC_REQ  → SKDm=003e2e1bfdfa6ad6  IVm=e20f256f
L13: LL_ENC_RSP  → SKDs=6ec2e6d4b0ab5dc8  IVs=c44f1b9b
L14+: Trafic BLE-CCM chiffré avec LTK → rapports HID clavier
```

---

## Formules BLE

### Fonction `e` (AES little-endian)
```python
def aes_e(key, data):
    cipher = AES.new(bytes(reversed(key)), MODE_ECB)
    return bytes(reversed(cipher.encrypt(bytes(reversed(data)))))
```

### Dérivation STK
```
STK = e(TK, Mrand[0:8] || Srand[0:8])
```

### Dérivation SK de session
```
SK  = reversed(e(root_key, SKDm || SKDs))
IV  = IVm || IVs
```

### Nonce BLE-CCM
```
nonce[0:5] = counter (39 bits LE) | direction<<39
nonce[5:13] = IV
```

---

## Exploit

```bash
cd remoble-control
python3 solve.py
```

Étapes du script :
1. Parse `session1.txt` → extrait Mrand, Srand, SKDm/SKDs, IV
2. Calcule STK avec TK=0 → dérive SK1
3. Déchiffre session1 BLE-CCM → trouve SMP Encryption Information → extrait LTK
4. Parse `session2.txt` → extrait SKDm2/SKDs2/IV2
5. Dérive SK2 depuis LTK
6. Brute-force les compteurs par direction → déchiffre tous les paquets valides (MIC OK)
7. Comble les trous avec déchiffrement sans MIC (pour suivre l'état prev_keys)
8. Décode les keycodes AZERTY + simule le buffer → reconstruit le texte tapé

---

## Flag

```
FCSC{bbeb5b8836d3cb2b55eae2ff0fdb58}
```


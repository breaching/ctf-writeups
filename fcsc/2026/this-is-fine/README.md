# This is fine - Walkthrough

## Énoncé

Le challenge fournit `this-is-fine.py` contenant 38 polynômes de degré 16 à coefficients entiers.  
Pour chaque polynôme `y`, on doit fournir deux entiers `this` et `fine` (identiques) tels que :

```python
assert this == fine          # this == fine
this = y(this)
fine = y(fine)
assert this is fine          # même OBJET Python!
flag.append(this | fine)
```

## Analyse

### L'opérateur `is` en Python

`a is b` vérifie l'**identité d'objet** (même adresse mémoire), pas l'égalité de valeur.  
En CPython, les entiers dans `[-5, 256]` sont des **singletons** (objets mis en cache).  
Pour tout entier `x ∈ [-5, 256]`, deux calculs indépendants donnant `x` renvoient le **même objet**.

Donc la condition `this is fine` n'est satisfaite que si `y(x) ∈ [-5, 256]`.  
Le flag est constitué des octets `y_i(x_i)` pour les bons `x_i`.

### Structure des polynômes

Chaque polynôme a la forme :
```
y(x) = c16·x^16 + c15·x^15 + ... + c0
```

Avec `c16 ≈ ±10^18` et les autres coefficients `≈ ±10^36`.

Pour `c16 > 0` : le polynôme tend vers `+∞` aux deux extrémités et possède un **minimum global**. Si ce minimum est négatif, le polynôme croise zéro en deux points.

**Observation clé** : les polynômes ont été construits avec un minimum extrêmement négatif (≈ -10^300), et les traversées de zéro se produisent aux alentours de `x ≈ 10^18`. Au niveau de la traversée, le polynôme est **exactement en train de monter de -∞ à +∞** - le premier entier après la traversée donne `y(x) = flag_byte ∈ [0, 255]`.

## Vulnérabilité / Trick mathématique

Les polynômes ont été construits de telle sorte qu'il existe un entier `x0 ≈ 10^18` (positif ou négatif) pour lequel `y(x0) = flag_byte`. Cet entier est exactement la première valeur positive (ou dernière positive selon le signe de `c16`) au niveau d'une traversée de zéro.

## Exploit

### Bisection pour trouver x0

**Étape 1** : Estimer l'emplacement du point critique.  
`r ≈ -c15 / (16·c16)` donne une approximation de la localisation du minimum/maximum.

**Étape 2** : Trouver un bracket `[lo, hi]` tel que `y(lo)` et `y(hi)` ont des signes opposés.

**Étape 3** : Bisection entière jusqu'à ce que `hi - lo = 1`.  
Pour `c16 > 0` (minimum négatif) : `y(hi)` est le flag_byte après la traversée `- → +`.  
Pour `c16 < 0` (maximum positif) : `y(lo)` est le flag_byte avant la traversée `+ → -`.

### Exemple pour L[0]

```python
y = L[0]  # c16 = 7356954580121243762 > 0
# y(10^18) < 0, y(10^19) > 0 → bracket [10^18, 10^19]
# Bisection → [9231776308982368925, 9231776308982368926]
# y(9231776308982368925) = (énorme valeur négative)
# y(9231776308982368926) = 70 = 'F' ✓
```

## Flag

`FCSC{dqcb7dVhWa9hunProTfJMsurwboPkT3V}`

## Valeurs x trouvées

```
L[0]:  x=9231776308982368926   → 70='F'
L[1]:  x=17470062679834211200  → 67='C'
L[2]:  x=3847942568692225585   → 83='S'
L[3]:  x=-3721779632509986439  → 67='C'
L[4]:  x=677697560488643437    → 123='{'
L[5]:  x=17812723399302343795  → 100='d'
L[6]:  x=-12412546571532943981 → 113='q'
L[7]:  x=-15361973521970193910 → 99='c'
L[8]:  x=8406688666999331825   → 98='b'
L[9]:  x=-6780362890780230533  → 55='7'
L[10]: x=11778943146259422890  → 100='d'
L[11]: x=4751901788189944829   → 86='V'
L[12]: x=5642035493410900900   → 104='h'
L[13]: x=-11486773298901092783 → 87='W'
L[14]: x=670012484308798372    → 97='a'
L[15]: x=783580903194883367    → 57='9'
L[16]: x=3194315361828845112   → 104='h'
L[17]: x=6099694414438288062   → 117='u'
L[18]: x=-6649006727152313000  → 110='n'
L[19]: x=-2884650051575278943  → 80='P'
L[20]: x=-587267009626931694   → 114='r'
L[21]: x=1249460788228168534   → 111='o'
L[22]: x=-8371366589354011104  → 84='T'
L[23]: x=16277005021309510506  → 102='f'
L[24]: x=-2996687033423600158  → 74='J'
L[25]: x=10878119015090189663  → 77='M'
L[26]: x=-10543627991007440312 → 115='s'
L[27]: x=9477782974638701787   → 117='u'
L[28]: x=7109741932748303777   → 114='r'
L[29]: x=-15066930010258048645 → 119='w'
L[30]: x=7042197916554764684   → 98='b'
L[31]: x=6489762439346473483   → 111='o'
L[32]: x=-6867960591914824875  → 80='P'
L[33]: x=6081286450582821811   → 107='k'
L[34]: x=-14577314880997671659 → 84='T'
L[35]: x=15960262031282502329  → 51='3'
L[36]: x=-13941603675556465785 → 86='V'
L[37]: x=-353866012970305198   → 125='}'
```

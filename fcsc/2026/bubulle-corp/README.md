# Bubulle Corp (Part 1/2)

## Énoncé
> Bubulle Corp recrute ! Rejoignez notre équipe d'experts marins et aidez-nous à surveiller les opérations en haute mer depuis notre tout nouveau tableau de bord.
> En tant que nouvelle recrue, vous aurez accès au suivi de la flotte, aux rapports de pêche et aux analyses de profondeur. Mais la rumeur dit que le capitaine cache sa recette secrète de Paella quelque part sur la plateforme...

## Architecture

3 services Docker sur 2 réseaux :

```
[Internet] → public-frontend (DMZ) ←→ internal-proxy (DMZ + internal) ←→ internal-backend (internal)
```

- **public-frontend** (port 8000) : Flask, register/login/settings/icon, réseau `dmz`
- **internal-proxy** : Apache reverse proxy, réseaux `dmz` + `internal`, sert `flag.txt` via AliasMatch
- **internal-backend** : Flask avec `/flag` (env var FLAG), réseau `internal` uniquement

## Analyse

### Flux normal
1. L'utilisateur s'inscrit, se connecte
2. Dans `/settings`, il soumet du XML pour configurer son icône (URL + méthode)
3. `/icon` déclenche `fetch_icon()` qui utilise **pycurl** pour récupérer l'image à l'URL configurée

### Validation des settings (routes.py)
- Parse le XML, vérifie que `<icon_url>` commence par `https://`
- Vérifie que `<method>` est `GET` ou `POST`
- Supprime les tags inconnus (seuls `icon_url`, `method`, `body` sont autorisés)
- **Itère uniquement sur les enfants directs** : `for elem in list(root)`

### Utilisation dans fetch_icon (icon.py)
- Parse le XML stocké
- **Cherche le premier descendant** `icon_url` : `root.find(".//icon_url")` (depth-first)
- Fait la requête avec pycurl

### Apache (internal-proxy)
- `HttpProtocolOptions Unsafe`
- `AliasMatch "^/.+" "/flag.txt"` → tout path ≠ `/` sert flag.txt
- Seul `/` est proxied vers le backend

## Vulnérabilité : XPath depth mismatch → SSRF

La validation itère `list(root)` (enfants directs), mais `fetch_icon` utilise `find(".//icon_url")` (tous les descendants, depth-first).

En nichant un `<icon_url>` malveillant dans `<body>` (tag autorisé, non inspecté récursivement), le `find()` le trouve en premier.

## Exploit

### Payload XML
```xml
<settings>
    <body><icon_url>http://bubulle-corp-internal-proxy/flag</icon_url></body>
    <icon_url>https://example.com</icon_url>
    <method>GET</method>
</settings>
```

### Chaîne d'attaque
1. Register + Login
2. POST /settings avec le XML ci-dessus (passe la validation)
3. GET /icon → pycurl fetch `http://bubulle-corp-internal-proxy/flag` → Apache sert `flag.txt`

### Script
Voir `exploit.py`

## Flag
```
FCSC{c22f014ba1aac9b3c487989156c470b0}
```

# CTF Writeups

![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)
![Writeups](https://img.shields.io/badge/writeups-13-success)
![FCSC 2026](https://img.shields.io/badge/FCSC%202026-10-blue)
![TryHackMe](https://img.shields.io/badge/TryHackMe-3-2dccff)

Forensics Zeek et auditd, reverse Xtensa sur ESP32, exploitation du cache CPython, hardware BLE, web SSRF. Mes writeups CTF, classés par plateforme.

## Sommaire

| Plateforme | Année | Writeups | Index |
|---|---|---|---|
| FCSC (ANSSI) | 2026 | 10 | [fcsc/2026](fcsc/2026/) |
| TryHackMe | 2023 | 3 | [tryhackme](tryhackme/) |

## Par catégorie

| Catégorie | Writeups |
|---|---:|
| Forensics | 5 |
| Hardware | 2 |
| Crypto | 1 |
| Reverse | 1 |
| Web | 1 |
| TryHackMe (boot2root) | 3 |

## À lire en premier

- **[Grhelp - Connect back](fcsc/2026/grhelp-connectback/)** : un binaire Chisel renommé `update` se cache dans des logs auditd. SOCKS reverse, SFTP upload, lateral movement via `sshpass`.
- **[This is fine](fcsc/2026/this-is-fine/)** : 38 polynômes à coefficients géants. Le piège est le cache CPython des petits entiers et l'opérateur `is`. Bisection pour faire tomber un polynôme dans `[-5, 256]`.
- **[Web Logs](fcsc/2026/web-logs/)** : 25k lignes de log Apache. L'attaque qui réussit se repère au byte près sur la taille de réponse.

## Langues

Les writeups FCSC sont en français, ceux de TryHackMe en anglais. Chaque plateforme dans sa langue native.

## Reproduire les challenges

Les fichiers de challenge ne sont pas dans ce repo.

- **FCSC** : archive officielle sur [hackropole.fr](https://hackropole.fr), sources sur [github.com/FCSC-FR](https://github.com/FCSC-FR)
- **TryHackMe** : chaque writeup linke sa room

## Disclaimer

Tout ce qui suit a été fait dans le cadre de CTF autorisés. Ne pas appliquer sur des systèmes que tu ne possèdes pas ou pour lesquels tu n'as pas de permission écrite.

## Licence

MIT. La prose, le code et les screenshots sont les miens. Les artefacts originaux (binaires, sources, captures) restent la propriété de leurs auteurs et ne sont pas redistribués ici.

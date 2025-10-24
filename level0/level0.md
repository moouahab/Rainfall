# Level0 — Extraction de données

**Résumé court :**
Le binaire vérifie si `atoi(argv[1]) == 423` (0x1a7). Si la condition est remplie, il suit le chemin qui appelle `execv` sur un fichier contenant le mot de passe ; sinon il termine normalement.

---

## Analyse (extrait assembleur)

```
0x08048ec9 <+9>:     mov    0xc(%ebp),%eax      # adresse argv
0x08048ecc <+12>:    add    $0x4,%eax          # argv + 1
0x08048ecf <+15>:    mov    (%eax),%eax       # argv[1]
0x08048ed1 <+17>:    mov    %eax,(%esp)
0x08048ed4 <+20>:    call   0x8049710 <atoi>  # atoi(argv[1])
0x08048ed9 <+25>:    cmp    $0x1a7,%eax       # compare avec 0x1a7 (423)
0x08048ede <+30>:    jne    0x8048f58         # si !=, branche vers sortie
0x08048ee0 <+32>:    movl   $0x80c5348,(%esp) # sinon prépare execv...
...
0x08048f51 <+145>:   call   0x8054640 <execv>  # ...et exécute
```

> **Remarque :** `0x1a7` hex = **423** décimal.

---

## Procédure pour reproduire

1. Lancer le binaire en fournissant `423` comme premier argument :

```bash
./level0 423
```

2. Lire le fichier contenant le mot de passe (exemple observé) :

```bash
cat /home/user/level1/.pass
```

---

## Résultat observé

```
1fe8a524fa4bec01ca4ea2a869af2a02260d4a7d5fe7e7c24d8617e6dca12d3a
```

---

## Interprétation rapide

* La comparaison `cmp $0x1a7, %eax` provient de l'appel `atoi(argv[1])`. Fournir la chaîne `"423"` satisfait la condition.
* Le binaire suit ensuite un chemin privilégié qui exécute `execv` et permet l'accès au fichier `.pass` contenant le flag/secret.

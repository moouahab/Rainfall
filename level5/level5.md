# Exploitation du niveau 5

## ✨ Objectif

Obtenir un shell en exploitant une vulnérabilité dans un binaire 32 bits. Le but est de rediriger l'exécution du programme vers la fonction `o()`, qui appelle `system("/bin/sh")`.

---

## 🔍 Analyse du binaire

### Fonctions importantes :

* `main()` : Appelle simplement la fonction `n()`
* `n()` :

  * Alloue un buffer sur la pile (0x218 octets)
  * Lit une chaîne de l'utilisateur avec `fgets`
  * Imprime cette chaîne avec `printf`
  * Appelle `exit()`
* `o()` : Appelle `system("/bin/sh")`

### Vulnérabilité :

La fonction `printf` est appelée **directement sur la chaîne contrôlée par l'utilisateur**, sans format string fixé.
Cela crée une vulnérabilité **de format string**.

---

## 🚫 Protection du binaire

* NX activé : oui
* Stack Canary : non
* PIE : non (important !)

---

## 🚀 Type d'exploitation utilisé

### Format String Exploit

Type : **GOT Overwrite** (redirection via `%n`)

Objectif : écraser l'entrée GOT de `exit()` (adresse dynamique appelée par `exit@plt`) pour la rediriger vers `o()`.

---

## ⚖️ Adresses utiles

| Cible         | Adresse      |
| ------------- | ------------ |
| o()           | `0x080484a4` |
| GOT[exit]     | `0x08049838` |
| GOT[exit] + 2 | `0x0804983A` |

On utilise `%hn` pour écrire 2 octets à la fois :

* `0x0804` dans `0x0804983A`
* `0x84a4` dans `0x08049838`

---

## 🔄 Construction du payload

### Ordre d'écriture

1. On écrit le **high** d'abord (`0x0804`) dans `GOT+2`
2. Puis le **low** (`0x84a4`) dans `GOT`

### Offsets

* Offset du premier argument contrôlé dans `printf`: `4`

### Payload Python final (en bytes):

```bash
python2 -c 'print("\x3a\x98\x04\x08" + "\x38\x98\x04\x08" + "%2044c%4$hn" + "%31904c%5$hn")'
```

### Explication des paddings

* 2 adresses = 8 octets imprimés
* 2052 (0x0804) - 8 = 2044
* 33956 (0x84a4) - 2052 = 31904

---

## ⚖️ Compilation

```bash
gcc -m32 -fno-stack-protector -no-pie -o level5 level5.c
```

---

## ✅ Résultat attendu

Une fois l'entrée injectée, `exit()` est appelé via la GOT, mais la GOT a été redirigée vers `o()`. Cela lance `system("/bin/sh")` :

✅ **Un shell est ouvert !**

---

## 🚀 Remarque finale

Cette exploitation ne repose sur **aucun dépassement de tampon**, seulement sur la mauvaise utilisation de `printf`. C'est un exemple classique d'erreur de format string dans un CTF de type pwn.

---

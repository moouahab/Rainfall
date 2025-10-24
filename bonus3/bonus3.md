# Exploitation du binaire `bonus3` – Dernier niveau RainFall

## 🎯 Objectif

Contourner la vérification `strcmp(buffer, argv[1])` pour lancer un `execl()` et obtenir un shell.

---

## 🔎 Analyse du comportement

1. Le programme lit jusqu'à **66 octets** depuis un fichier (via `fread`) dans un buffer `buf`.
2. Il ajoute un `\0` à l'offset donné par `atoi(argv[1])` (sert à tronquer `buf`).
3. Il fait ensuite :

```c
strcmp(buf, argv[1])
```

4. Si la comparaison réussit (== 0), il exécute :

```c
execl("/bin/sh", "sh", NULL);
```

---

## 🧠 Découverte clé avec Ghidra

En examinant le binaire avec Ghidra, on remarque que :

* Le contenu attendu dans `buf` est une **chaîne vide** : `""`
* La comparaison devient donc :

```c
strcmp(buf, "") == 0
```

---

## ✅ Exploit

On appelle simplement le binaire avec une chaîne vide :

```bash
./bonus3 ''
```

➡️ `argv[1] = ""` → `atoi(argv[1]) = 0`
➡️ Le buffer est tronqué à 0, donc `buf[0] = '\0'`
➡️ `strcmp("", "") == 0` ✅
➡️ `execl("/bin/sh")` est exécuté 🎉

---

## 🏁 Flag final obtenu

```
3321b6f81659f9a71c76616f606e4b50189cecfea611393d5d649f75e157353c
```

---

## 🙌 Fin du challenge

Tu as réussi à :

* Lire et analyser l’ASM ligne par ligne
* Comprendre des effets subtils comme `atoi()` et `strcmp()`
* Contourner la vérification finale
* Obtenir un shell et le **flag final du CTF RainFall** 🏆

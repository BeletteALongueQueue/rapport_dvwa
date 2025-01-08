# 1. DOM Based XSS

Le DOM-Based XSS (Cross-Site Scripting) est une variante de vulnérabilité XSS où l'injection malveillante se produit dans le Document Object Model (DOM) côté client, sans jamais transiter par le serveur. Autrement dit, le code malveillant est interprété directement par le navigateur en manipulant des données dynamiques gérées dans le DOM, souvent à partir de l'URL ou d'autres entrées utilisateur.

---

## 1.1 Premier niveau - low

Nous arrivons sur une page où l'on peut changer la langue.

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\1.png?msec=1736350428910)

On remarque un paramètre `GET` dans l'URL qui est `default` :

```url
http://34.163.97.167/DVWA/vulnerabilities/xss_d/?default=French
```

Nous modifions la valeur `French` par une injection JavaScript simple pour tester :

```url
http://34.163.97.167/DVWA/vulnerabilities/xss_d/?default=%3Cscript%3Ealert(%27info%27);%3C/script%3E
```

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\2.png?msec=1736350428912)

La page est vulnérable aux attaques XSS.

### Exploitation : récupération de cookies

Nous essayons ensuite d'exploiter la faille pour récupérer les cookies de session via `document.cookie` :

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\3.png?msec=1736350428911)

Les cookies de session sont bien récupérés. On constate qu'aucun filtre ou protection n'est mis en place.

---

## 1.2 Deuxième niveau - medium

Pour ce niveau, les balises simples comme `<script>` semblent filtrées. Nous devons trouver une autre méthode.

### Analyse et modification

Après plusieurs essais infructueux dans l'URL, nous passons en mode développeur pour modifier le script directement dans le navigateur.

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\4.png?msec=1736350428914)

Nous ajoutons un attribut `onclick` au bouton `Select` pour afficher les cookies :

```html
<input type="submit" value="Select" onclick="alert(document.cookie)">
```

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\xssDom\5.png?msec=1736350428913)

### Vérification dans le code

Le code PHP montre une vérification basique :

```php
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

Le script bloque les balises `<script>`, mais d'autres méthodes d'injection restent possibles.

---

## 1.3 Troisième niveau - hard

### Analyse du code source

Le code suivant montre une amélioration avec une liste blanche :

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

L'administrateur a ajouté une liste blanche pour vérifier si le paramètre `default` correspond bien à une langue autorisée. Cependant, il reste possible d'injecter après le paramètre validé grâce au caractère `&`.

### Exploitation

Nous injectons le XSS après le paramètre `default` :

```url
http://34.163.97.167/DVWA/vulnerabilities/xss_d/?default=German&<script>alert(document.cookie);</script>
```

Le payload est fonctionnel, et nous récupérons les cookies de session.

---
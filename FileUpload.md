# 1. File Upload

## 1.1 Premier Niveau - low

Pour ce premier niveau, nous avons la possibilité d'uploader un fichier. On part du principe qu'il n'y a pas de contrôle sur les fichiers que l'on dépose.

Nous allons déposer un webshell en `PHP` afin d'exécuter des commandes sur le serveur.

Nous utiliserons ce webshell disponible au lien suivant :

```url
https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985#file-easy-simple-php-webshell-php
```

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\1.png?msec=1736349715368)

Une fois que le fichier est uploadé, nous accédons à l'URL donnée par le site :

```url
http://34.163.97.167/DVWA/hackable/uploads/test.php
```

Le shell est bien présent et nous permet de saisir des commandes :  
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\2.png?msec=1736349715388)

---

## 1.2 Deuxième niveau - medium

Pour le deuxième challenge, en essayant d'uploader un fichier PHP sur le site, nous sommes bloqués car le site n'accepte que des fichiers `JPEG` et `PNG` :

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\3.png?msec=1736349715372)

Nous utilisons Burp Suite. Tout d'abord, nous uploadons un fichier PHP contenant notre shell, puis nous interceptons la requête et modifions la ligne :

```http
Content-Type: application/octet-stream
```

En la remplaçant par :

```http
Content-Type: image/jpeg
```

La requête doit ressembler à ceci :

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\4.png?msec=1736349715403)

En procédant ainsi, nous contournons la vérification du serveur et accédons à notre webshell :  
![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\5.png?msec=1736349715390)

Voici le code source PHP pour mieux comprendre :

```php
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );

    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_type = $_FILES[ 'uploaded' ][ 'type' ];
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];

    // Is it an image?
    if( ( $uploaded_type == "image/jpeg" || $uploaded_type == "image/png" ) &&
        ( $uploaded_size < 100000 ) ) {

        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} successfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}

?>
```

Le code source contrôle le type de fichier, mais il est possible de modifier le `Content-Type` tout en gardant l'extension `.php` pour contourner cette sécurité.

---

## 1.3 Troisième niveau - high

Pour le troisième niveau, après quelques tests, nous constatons que le champ `Content-Type` n'est pas vérifié, mais que l'extension l'est. Nous utilisons la même méthode, en interceptant notre requête avec Burp Suite et en modifiant les informations. Cette fois, nous changeons le `Content-Type` pour celui du PHP : `Content-Type: application/x-php`.

### Requête initiale :

![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\6.png?msec=1736349715417)

### Requête modifiée :

(à compléter avec la capture d'écran ou les détails finaux).
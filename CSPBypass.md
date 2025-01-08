# 1. CSP Bypass

Une **faille CSP** (Content Security Policy) désigne une faiblesse ou une mauvaise configuration de la **Politique de Sécurité du Contenu** (Content Security Policy) d'un site web. Cette politique, utilisée pour protéger contre des attaques comme le **Cross-Site Scripting (XSS)** ou l'injection de contenu malveillant, devient inefficace si elle est mal configurée ou contournée.

---

## 1.1 Premier niveau - low

Voici ce que nous avons dans la catégorie CSP :  
![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\1.png?msec=1736349682104)

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

On voit que le script récupère le contenu du formulaire et le met entre des balises `<script>`. Il permet d'effectuer des requêtes vers certains sites externes avec `script-src 'self' https://code.jquery.com;` qui autorise uniquement les scripts locaux et certains sites "sûrs".

Par exemple, nous utilisons :

```url
https://digi.ninja/dvwa/alert.js
```

Cette page contient ce code :

```js
alert("CSP Bypassed");
```

Ainsi, si la page est vulnérable, le script s'exécutera sur notre site DVWA.  

![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\2.png?msec=1736349682105)

On soumet simplement le lien et on clique sur "Include" pour voir le résultat. Nous obtenons bien l'alerte :  
![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\3.png?msec=1736349682106)

Il y a donc une faille CSP.

---

## 1.2 Niveau intermédiaire - medium

Analysons encore une fois le code source :

```php
$headerCSP = "Content-Security-Policy: script-src 'self' 'unsafe-inline' 'nonce-TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=';";
```

Pour `script-src`, nous avons :

- `'self'` : permet l'exécution de ressources fournies par la même origine.
- `'unsafe-inline'` : autorise les scripts inclus dans les balises `<script>`.
- `'nonce-TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA='` : exige qu'un script possède ce nonce pour être exécuté.

La faille réside dans le fait que le nonce est censé être dynamique (unique pour chaque requête). Or, ici, en effectuant plusieurs requêtes `GET`, nous remarquons que le nonce reste le même et devient donc inutile.

Nous forgeons une requête :

```html
<script nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=">alert('test');</script>
```

Résultats :  
![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\4.png?msec=1736349682107)  
![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\5.png?msec=1736349682107)

---

## 1.3 Troisième niveau - high

Cette fois, la page nous propose de réaliser un calcul en passant par une autre page :  
![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\6.png?msec=1736349682108)

Nous interceptons la requête à cette page et injectons notre propre code pour tester la vulnérabilité.

### Requête initiale :

![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\7.png?msec=1736349682109)

### Requête modifiée :

![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\8.png?msec=1736349682109)

Nous obtenons bien le résultat souhaité : une boîte de dialogue (alert).  
![images](file://C:\Users\sacha\Desktop\pentest_dvwa\rapport_dvwa\images\csp\9.png?msec=1736349682110)

---

## 1.4 Dernier niveau - impossible

À compléter pour ce niveau.

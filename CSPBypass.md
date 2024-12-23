# 1. CSP Bypass

Une **faille CSP** (Content Security Policy) désigne une faiblesse ou une mauvaise configuration de la **Politique de Sécurité du Contenu** (Content Security Policy) d'un site web. Cette politique, utilisée pour protéger contre des attaques comme le **Cross-Site Scripting (XSS)** ou l'injection de contenu malveillant, devient inefficace si elle est mal configurée ou contournée

## 1.1 Niveau simple - low

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

On voit que le script nous permet d'effectuer des requetes a des sites exterieures par exemple nous utiliserons ici :

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


## Low

Notre but ici est de nous rediriger depuis DVWA vers le site de notre choix.

![[Pasted image 20250109111948.png]]

On observe en cliquant sur un des deux liens que nous sommes redirigés vers une autre page web qui est `dvwa.lan/vulnerabilities/open_redirect/source/info.php`, pour en savoir plus sur le fonctionnement de cette redirection nous nous rendons sur Burp Suite.

Dans l'onglet "proxy", nous allons pouvoir décomposer toutes les requêtes qui permettent d'arriver jusqu'à la page des citations.
![[Pasted image 20250109112417.png]]
Lorsque l'on clique sur un des liens on observe la première requête vers une nouvelle page que nous n'avions pas vu et qui sert justement à rediriger l'utilisateur vers la page avec les citations.

L'outil proxy nous permet aussi de modifier les requêtes HTTP interceptés avant de les envoyer, ici on remplace la page info.php par l'URL www.google.com :
![[Pasted image 20250109112903.png]]

En laissant passer la requête, on voit en effet que l'on est redirigé non pas vers la page demandée, mais vers google.com
![[Pasted image 20250109113218.png]]

Cette faille ne parait pas très dangereuse présentée ainsi, pourtant elle peut tromper la vigilance d'utilisateurs ou de systèmes de vérification. Par exemple avec une faille de type Open Redirect je peux utiliser l'URL d'un site légitime et pourtant rediriger vers un site malveillant.


## Medium

Si l'on réessaie la même méthode que précédemment on tombe désormais sur cette page :
![[Pasted image 20250109115425.png]]

Ici on ne peut peut plus utiliser HTTP ou HTTPS dans le champ "redirect", on va donc utiliser les "Protocol Relative URL", cette méthode permet à l'origine de mettre un lien sans spécifier si le protocole a utiliser est HTTP ou HTTPS, ce qui permet d'utiliser automatiquement HTTPS seulement si il est disponible. L'URL de Google s'écrirait donc comme cela : `//www.google.com`.

On modifie donc la requête dans l'outil Proxy de Burp Suite :
![[Pasted image 20250109120204.png]]

On arrive finalement sur le site de Google !
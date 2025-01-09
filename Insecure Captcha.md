# Low

Notre but sur cette page est de contourner la vérification CAPTCHA.

![[Pasted image 20241217091321.png]]

En remplissant le formulaire normalement, on capture les requêtes avec burp suite on observe 2 requêtes POST qui correspondent à l'envoi du formulaire.

![[Pasted image 20241217091403.png]]

La première est difficilement exploitable il s'agit de la validation du captcha, via la vérification proposé par google.
![[Pasted image 20241217091429.png]]

La deuxième requête en revanche est très intéressante, elle permet de modifier le mot de passe mais ne prend pas en compte la vérification captcha, on peut l'envoyer vers l'onglet repeater de burp suite.
![[Pasted image 20241217091448.png]]

Dans cet onglet repeater, on va pouvoir envoyer uniquement la requête post sans le vérification captcha, on essaie donc de modifier le mot de passe par le nouveau mot de passe "test".
![[Pasted image 20241217091605.png]]

On observe la réponse du serveur en mode rendu et on arrive effectivement sur la page de validation.
![[Pasted image 20241217091632.png]]

## Medium

La méthode est exactement la même que pour le niveau précédent, la première requête est indépendante de la requête de confirmation, cependant une variable supplémentaire est nécessaire :
![[Pasted image 20250109105512.png]]

On voit qu'un nouveau champ **passed_captcha** est présent, il suffit de mettre sa valeur sur *true* pour passer la sécurité.
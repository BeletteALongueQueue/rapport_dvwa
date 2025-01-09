## Low

Notre but est d'arriver à cette page sans être connecté avec l'utilisateur admin :
![![[Pasted image 20250109125023.png]]](C:\Users\Sacha\Desktop\SAE%20Pentest\SAE%20Pentest\Authentification%20Bypass\Pasted%20image%2020250109125023.png)

On se connecte donc avec l'utilisateur gordonb et on observe que le bouton pour accéder a la page a disparu.

En entrant directement l'URL de la page `dvwa.lan/vulnerabilities/authbypass/` on arrive sur la page d'administration sans compte administrateur.

## Medium

Cette fois ci, lorsque l'on essaie de d'accéder à la page, cela ne marche pas, on arrive sur une page "unauthorized".

En revenant avec un compte admin on peut observer avec Burp Suite comment la page est chargée, une première requête vers la page `authbypass` est envoyé puis une deuxième vers `authbypass/get_user_data.php`.
On revient à sur notre compte gordonb et l'on essaie d'accéder directement à la page `get_user_data.php` :
![![[Pasted image 20250109130943.png]]](C:\Users\Sacha\Desktop\SAE%20Pentest\SAE%20Pentest\Authentification%20Bypass\Pasted%20image%2020250109130943.png)
On peut voir tout les utilisateurs enregistrés dans la base de donnée mais pour l'instant nous ne pouvons pas modifier leurs informations.

Analysons avec le compte administrateur ce qu'il se passe lorsque l'on modifie un utilisateur :
![![[Pasted image 20250109131344.png]]](C:\Users\Sacha\Desktop\SAE%20Pentest\SAE%20Pentest\Authentification%20Bypass\Pasted%20image%2020250109131344.png)
Une requête POST est envoyée à une nouvelle page `change_user_details.php`, on peut imaginer que comme la page `get_user_data.php`, elle aussi est accessible sans être administrateur.

La requête POST en elle même est construite comme ci :
![![[Pasted image 20250109131618.png]]](C:\Users\Sacha\Desktop\SAE%20Pentest\SAE%20Pentest\Authentification%20Bypass\Pasted%20image%2020250109131618.png)
Les nouvelles données de l'utilisateur sont simplement envoyés au format JSON.

On peut construire notre requête POST avec cURL, voici la commande :

```bash
curl -H "Cookie: PHPSESSID=b5d54af5529246f0ccad0ff747e84dee; security=medium" \
     -H "Content-Type: application/json" \
     -d '{"id":4,"first_name":"Pablo","surname":"Escobar"}' \
     -X POST http://localhost:4280/vulnerabilities/authbypass/change_user_details.php
```

> Explication de la commande :
>     - `-H "Cookie: ...` -> Permet de modifier le header HTTP pour se connecter avec le cookie de la session de l'utilisateur gordonb et de mettre le niveau de sécurité en medium
>     - `-H "Content-Type: ...`-> Permet d'indiquer ce le format des données envoyées dans le header HTTP
>     - `-d '{"id":4, ....` -> Contenu de la requête POST
>     - `-X POST http://...` -> Méthode de la requête et page à cibler

![![[Pasted image 20250109133550.png]]](C:\Users\Sacha\Desktop\SAE%20Pentest\SAE%20Pentest\Authentification%20Bypass\Pasted%20image%2020250109133550.png)

Notre requête a bien été envoyée et accepté, en allant sur la page `get_user_data.php`, on observe que l'utilisateur a bien été modifié.
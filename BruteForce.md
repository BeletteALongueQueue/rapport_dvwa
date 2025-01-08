# 1. Brute Force

Inutile d'expliquer le bruteforce, mais nous allons quand même le faire !  
Le bruteforce est une méthode d'attaque consistant à deviner un mot de passe, une clé de chiffrement ou d'autres informations sensibles en testant systématiquement toutes les combinaisons possibles jusqu'à trouver la bonne. C'est une méthode simple mais parfois efficace, notamment lorsque les mots de passe ou clés sont faibles ou mal sécurisés.

Voici une liste d'outils couramment utilisés pour le bruteforce :

| **Outil**           | **Description**                                                              | **Cible principale**                    |
| ------------------- | ---------------------------------------------------------------------------- | --------------------------------------- |
| **Hydra**           | Outil rapide pour tester des mots de passe sur divers services réseau.       | SSH, FTP, HTTP, SMTP, etc.              |
| **Medusa**          | Outil de bruteforce modulaire et rapide pour les services réseau.            | SSH, FTP, Telnet, MySQL, etc.           |
| **John the Ripper** | Craqueur de mots de passe hachés puissant et configurable.                   | Hachages de mots de passe               |
| **Hashcat**         | Outil avancé pour le déchiffrement des hachages, supportant les GPU.         | Hachages de mots de passe               |
| **Burp Suite**      | Suite d'outils pour les tests de sécurité, incluant un module de bruteforce. | Formulaires web, sessions HTTP          |
| **Wfuzz**           | Outil flexible pour le bruteforce des paramètres web.                        | URL, cookies, paramètres web            |
| **Patator**         | Outil modulaire pour tester les mots de passe et d'autres services.          | SSH, HTTP, DNS, MySQL, etc.             |
| **Ncrack**          | Outil rapide pour tester les authentifications réseau.                       | SSH, RDP, FTP, Telnet, etc.             |
| **Aircrack-ng**     | Outil pour le craquage des clés de réseaux Wi-Fi.                            | Clés WEP/WPA sur Wi-Fi                  |
| **CeWL**            | Génère des wordlists basées sur le contenu d'un site.                        | Création de dictionnaires personnalisés |

---

## 1.1 Premier niveau - low

Nous sommes face à une simple page de login.  
![image](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\1.png)

En testant des identifiants, on remarque que les données sont envoyées via la méthode `GET`, comme en témoigne l'URL suivante :

```url
http://dvwa.lan/DVWA/vulnerabilities/brute/?username=admin&password=admin&Login=Login#
```

Nous allons utiliser **Hydra** pour effectuer un bruteforce.  
Voici la commande utilisée :

```bash
hydra -l admin -P /opt/wordlist/wordlists/wordlists/passwords/most_used_passwords.txt dvwa.lan http-get-form "/DVWA/
vulnerabilities/brute/index.php:username=^USER^&password=^PASS^&Login=Login:H=Cookie:PHPSESSID=bnk7pa0fohd54o1l6orc5624ko; security=low:F=Username and/or password incorrect."
```

Résultat :  
![image](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\2.png)  
Identifiants trouvés : `admin` / `password`.

---

## 1.2 Deuxième niveau - medium

Pour ce niveau, nous utilisons une wordlist à la fois pour les identifiants et les mots de passe :

```bash
hydra -L /opt/wordlists/wordlists/languages/english.txt -P /opt/wordlists/wordlists/passwords/password.txt dvwa.lan http-get-form "/DVWA/vulnerabilities/brute/index.php:username=^USER^&password=^PASS^&Login=Login:H=Cookie:PHPSESSID=dvkdo50vcgtak32lhhadgk8rvs; security=medium:F=Username and/or password incorrect."
```

Cette méthode fonctionne toujours. Cependant, le code source introduit un délai de 2 secondes après chaque tentative de connexion incorrecte, ce qui ralentit le bruteforce :

```php
else {
    sleep(2);
    echo "<pre><br />Username and/or password incorrect.</pre>";
}
```

Le bruteforce reste cependant efficace, bien que plus lent.

---

## 1.3 Troisième niveau - hard

Pour ce niveau, un **token CSRF** est ajouté au formulaire. Voici un extrait du code source :

```php
checkToken($_REQUEST['user_token'], $_SESSION['session_token'], 'index.php');
```

Ce token, unique et généré dynamiquement, est validé à chaque requête. Si le token est absent ou incorrect, la requête est rejetée. Voici une capture Burp Suite illustrant cela :  
![image](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\3.png)

Nous utilisons **Burp Suite** pour gérer ce token automatiquement.  
Voici la configuration dans **Intruder** :  

1. **Mode d'attaque :** Pitchfork.  
2. **Paramètres sélectionnés :** `password` et `user_token`.  
3. Extraction du token via la fonction **Grep - Extract**.

Exemple de configuration pour le `Grep - Extract` :  
![image](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\6.png)

Résultat :  
![image](file://C:\Users\Sacha\Desktop\pentest_dvwa\rapport_dvwa\images\bruteforce\8.png)

Mot de passe trouvé !

---

## 1.4 Niveau Impossible

Le niveau `Impossible` ajoute plusieurs mécanismes anti-bruteforce :  

1. **Token CSRF**.  
2. **Verrouillage après un certain nombre d'échecs**.  
3. **Délai aléatoire (2 à 4 secondes)** pour ralentir les tentatives.

Voici un extrait du code :

```php
if ($row['failed_login'] >= $total_failed_login) {
    $timeout = $last_login + ($lockout_time * 60);
    if ($timenow < $timeout) {
        $account_locked = true;
    }
}
```

Ces protections rendent le bruteforce pratiquement impossible sans exploiter d'autres vulnérabilités.

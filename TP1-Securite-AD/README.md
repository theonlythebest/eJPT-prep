# Sécurité de l'Active Directory - TP 1 (Corrigé)

> Réponses et compte-rendu pour le TP1 (voir SUJET.md pour les consignes).

## Configuration de départ

- Machine attaquant : Kali Linux, IP `192.168.222.128/24`
- Contrôleur de domaine (DC) : IP `192.168.222.135`
- Domaine Active Directory : `epita.local`

📸 *Screenshot à ajouter : `ip a` / `ip route` sur la machine Kali*

## Scan

- Adresse IP du contrôleur de domaine : `192.168.222.135`
- Résultats du scan nmap : un scan `nmap -sC -sV -p- 192.168.222.135` révèle les ports classiques d'un contrôleur de domaine Active Directory : 53 (DNS), 88 (Kerberos), 135 (RPC), 139/445 (SMB), 389/636 (LDAP/LDAPS), 3268/3269 (Global Catalog), 5985 (WinRM). La présence de ces ports et la réponse sur le port 389 confirment qu'il s'agit bien d'un contrôleur de domaine.
- Nom de domaine confirmé (rootDomainNamingContext) : `DC=epita,DC=local`, obtenu via une requête `ldap-rootdse` (script nmap `ldap-rootdse.nse` ou `ldapsearch -x -h 192.168.222.135 -s base namingcontexts`).

📸 *Screenshot à ajouter : résultat du scan nmap*
📸 *Screenshot à ajouter : résultat ldap-rootdse / namingcontexts*

## LDAP

### Observer la commande ldapsearch

- Explication des arguments :
  - `-x` : authentification simple (non-SASL)
  - `-H ldap://192.168.222.135` : URL du serveur LDAP cible
  - `-b "DC=epita,DC=local"` : base de recherche (DN racine du domaine)
  - `-s sub` : portée de recherche (sous-arbre complet)
  - (sans `-D`/`-w`) : connexion en anonyme, pour tester si le bind anonyme est autorisé
- Utilité des informations retournées : la réponse liste les objets de l'annuaire (utilisateurs, groupes, ordinateurs, OUs) avec leurs attributs (sAMAccountName, description, memberOf, etc.). Si le bind anonyme fonctionne, on peut déjà récupérer la liste des comptes du domaine et parfois des informations sensibles placées dans le champ "description" (mots de passe, indices...).

📸 *Screenshot à ajouter : commande ldapsearch (bind anonyme réussi)*

### Résultats go-windapsearch

- Utilisateurs trouvés (users.txt) : extraction de tous les `sAMAccountName` des objets `user` du domaine avec `go-windapsearch --dc-ip 192.168.222.135 -m users` (ou ldapsearch + grep/awk), liste enregistrée dans `users.txt` pour préparer les attaques de password spraying / AS-REP Roasting.
- Mots de passe trouvés dans les descriptions (passwords.txt) : en parcourant l'attribut `description` de chaque compte, certains comptes contiennent un mot de passe en clair laissé par erreur par l'administrateur (mauvaise pratique courante en AD). Ces valeurs sont extraites et regroupées dans `passwords.txt` pour servir de dictionnaire de mots de passe lors du password spraying.

📸 *Screenshot à ajouter : extraction users.txt*
📸 *Screenshot à ajouter : descriptions contenant des mots de passe*

### Observer la commande nxc smb (password spraying)

- Que réalise cette commande ? `nxc smb 192.168.222.135 -u users.txt -p passwords.txt` teste, pour chaque combinaison utilisateur/mot de passe, une authentification SMB sur le contrôleur de domaine. NetExec (nxc) renvoie pour chaque tentative le statut d'authentification (succès, échec, compte verrouillé, etc.), ce qui permet d'identifier rapidement les comptes valides sans déclencher de verrouillage massif (password spraying = un mot de passe testé sur beaucoup de comptes, plutôt que beaucoup de mots de passe sur un compte).
- Pourquoi certains messages d'erreur sont différents des autres ? Les codes de retour NTSTATUS diffèrent selon la raison de l'échec : `STATUS_LOGON_FAILURE` (mauvais mot de passe), `STATUS_ACCOUNT_LOCKED_OUT` (compte verrouillé), `STATUS_PASSWORD_EXPIRED` / `STATUS_PASSWORD_MUST_CHANGE` (mot de passe expiré, changement obligatoire à la prochaine connexion), ou un retour positif marqué `[+]` quand l'authentification réussit. Ces différences permettent de distinguer un compte "intéressant" (mot de passe expiré, compte verrouillé) d'un simple échec d'authentification.

📸 *Screenshot à ajouter : résultat nxc smb password spraying (comptes valides en vert/+)*

### Politique de mot de passe (--pass-pol)

- Flags de complexité retournés : la commande `nxc smb 192.168.222.135 -u <user> -p <pass> --pass-pol` retourne la politique de mot de passe du domaine : longueur minimale, historique des mots de passe, âge minimal/maximal, seuil de verrouillage (lockout threshold), durée de verrouillage (lockout duration) et le flag de complexité (`Password Complexity = Disabled/Enabled`). Dans ce lab, la complexité est désactivée, ce qui facilite grandement le password spraying et le bruteforce de mots de passe simples.

📸 *Screenshot à ajouter : résultat nxc --pass-pol*

## AS-REP Roasting

- Compte avec mot de passe modifié (pré-authentification Kerberos désactivée) : un ou plusieurs comptes du domaine ont l'option "Do not require Kerberos preauthentication" activée. Cela permet de demander un ticket AS-REP pour ce compte sans connaître son mot de passe.
- Résultat impacket-GetNPUsers : `impacket-GetNPUsers epita.local/ -no-pass -usersfile users.txt -dc-ip 192.168.222.135` renvoie un hash AS-REP (format `$krb5asrep$23$...`) pour chaque compte vulnérable trouvé.
- Hash AS-REP cassé (hashcat mode 18200) : `hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt` permet de retrouver le mot de passe en clair du compte vulnérable.

📸 *Screenshot à ajouter : commande impacket-GetNPUsers (hash AS-REP récupéré)*
📸 *Screenshot à ajouter : hashcat -m 18200 (mot de passe trouvé)*

### Observer la commande nxc --continue-on-success

- À quoi sert --continue-on-success ? Par défaut, NetExec arrête le password spraying sur un utilisateur dès qu'une combinaison valide est trouvée pour celui-ci. L'option `--continue-on-success` force l'outil à continuer de tester toutes les combinaisons restantes même après un succès, ce qui permet de retrouver TOUS les couples identifiant/mot de passe valides (utile quand plusieurs comptes partagent le même mot de passe issu des descriptions).
- Comment est représenté un couple identifiant/mot de passe valide ? Une ligne marquée `[+]` au format `epita.local\<utilisateur>:<mot de passe>` (avec éventuellement `(Pwn3d!)` si le compte a des droits d'administration locale sur la machine cible).
- Identifiants/mots de passe valides trouvés :
  - `binny.inger` / `password`
  - `camellia.analiese` / `password`
  - `maryann.beckie` / `password`
  - `felice.rochelle` / `password`
  - `albertine.cornela` / `password`

📸 *Screenshot à ajouter : nxc smb --continue-on-success (liste des comptes valides)*

## BloodHound

### Installation

- Notes d'installation (BloodHound Legacy, Neo4j, bloodhound.py) :
  - Installation de Neo4j (base de données graphe) : `sudo apt install neo4j`, démarrage avec `sudo neo4j console` et configuration du mot de passe initial via l'interface web (http://localhost:7474).
  - Installation de BloodHound (interface graphique) : `sudo apt install bloodhound` (ou version Legacy depuis le dépôt GitHub), lancement avec `bloodhound`.
  - Installation du collecteur Python : `pip install bloodhound` (paquet `bloodhound-python`).

📸 *Screenshot à ajouter : interface de connexion BloodHound / Neo4j*

### Collecte (bloodhound-python / nxc ldap)

- Qu'est-ce que la commande a fait ? `bloodhound-python -u <user> -p <pass> -ns 192.168.222.135 -d epita.local -c All` se connecte au contrôleur de domaine via LDAP et SMB pour collecter l'ensemble des informations sur les objets du domaine (utilisateurs, groupes, ordinateurs, OUs, GPOs, sessions actives, droits ACL, relations de confiance/délégation, appartenance aux groupes locaux). L'option `-c All` récupère toutes les collections disponibles.
- Contenu du fichier généré : plusieurs fichiers JSON sont générés (`*_users.json`, `*_groups.json`, `*_computers.json`, `*_domains.json`, `*_gpos.json`, `*_ous.json`, `*_containers.json`), chacun contenant les objets correspondants avec leurs propriétés et relations, prêts à être importés dans Neo4j via l'interface BloodHound.
- Résultat de la commande -M get-network : le module `get-network` de NetExec (`nxc ldap 192.168.222.135 -u <user> -p <pass> -M get-network`) interroge le LDAP pour lister les informations réseau enregistrées dans l'annuaire (sous-réseaux/sites AD, `siteName`, plages d'adresses), utile pour cartographier l'infrastructure réseau du domaine au-delà de ce qui est visible depuis la machine attaquante.

📸 *Screenshot à ajouter : exécution bloodhound-python (collecte terminée)*
📸 *Screenshot à ajouter : résultat nxc ldap -M get-network*

### Analyse dans BloodHound

Après import des fichiers JSON dans l'interface BloodHound, le graphe des relations du domaine `epita.local` est analysé :

- Marquage du compte compromis comme **"Owned"** (clic droit -> Mark User as Owned) pour visualiser tous les chemins d'attaque partant de ce compte.
- Utilisation de la fonction **"Shortest Path"** (Analysis -> Shortest Paths to Domain Admins) pour identifier le chemin le plus court entre un compte compromis et le groupe `Domain Admins`.
- Le graphe met en évidence les relations `MemberOf`, `AdminTo`, `GenericAll`, `ForceChangePassword`, etc. permettant d'identifier des escalades de privilèges possibles vers l'administrateur du domaine.

📸 *Screenshot à ajouter : import des fichiers JSON dans BloodHound*
📸 *Screenshot à ajouter : compte marqué comme "Owned"*
📸 *Screenshot à ajouter : Shortest Path to Domain Admins*

## Kerberoasting

- `impacket-GetUserSPNs epita.local/<user>:<pass> -dc-ip 192.168.222.135 -request` liste tous les comptes de service possédant un SPN (Service Principal Name) et demande pour chacun un ticket de service (TGS) chiffré avec le hash NTLM du compte de service.
- Le ticket récupéré est au format `$krb5tgs$23$...` (hashcat mode 13100).
- Crack avec `hashcat -m 13100 tgs.hash /usr/share/wordlists/rockyou.txt` :
  - Compte de service trouvé : `http_svc`
  - Mot de passe cassé : `redskins`

📸 *Screenshot à ajouter : impacket-GetUserSPNs -request (TGS récupéré)*
📸 *Screenshot à ajouter : hashcat -m 13100 (http_svc:redskins)*

## WinRM

Tentative de connexion avec `evil-winrm -i 192.168.222.135 -u <user> -p <pass>` en utilisant les comptes valides trouvés précédemment.

Résultat : connexion refusée / accès insuffisant. Les comptes compromis à ce stade ne sont membres ni du groupe `Remote Management Users` ni administrateurs locaux, ce qui empêche l'utilisation de WinRM (port 5985) pour obtenir un shell distant.

📸 *Screenshot à ajouter : tentative evil-winrm (accès refusé)*

## Bonus - Compromission du compte Administrateur

- À partir des mots de passe déjà récupérés (`password`, `redskins`...), génération de règles de mutation hashcat (`best64.rule`, règles personnalisées ajoutant majuscules, chiffres, caractères spéciaux) appliquées à une wordlist de base pour générer un nombre important de candidats supplémentaires.
- Test de ces candidats sur le compte `Administrateur` via `nxc smb 192.168.222.135 -u Administrateur -p candidates.txt` (password spraying ciblé).
- Mot de passe trouvé pour le compte `Administrateur` : **`Toto89100!`**

📸 *Screenshot à ajouter : nxc smb -u Administrateur -p candidates.txt (succès Pwn3d!)*

### Exécution distante et DCSync

- Connexion en tant qu'administrateur via `impacket-psexec epita.local/Administrateur:'Toto89100!'@192.168.222.135`, obtenant un shell `NT AUTHORITY\SYSTEM` sur le contrôleur de domaine.
- Extraction des hashes de tous les comptes du domaine via une attaque **DCSync** avec `impacket-secretsdump` : `impacket-secretsdump epita.local/Administrateur:'Toto89100!'@192.168.222.135`. Cette commande exploite les droits de réplication (`Get-Replicating-Changes`) du compte administrateur pour demander au DC de "répliquer" la base NTDS.dit, récupérant ainsi les hashes NTLM de tous les comptes du domaine (y compris `krbtgt`), ce qui permet notamment la création d'un Golden Ticket.

📸 *Screenshot à ajouter : impacket-psexec (shell SYSTEM obtenu)*
📸 *Screenshot à ajouter : impacket-secretsdump (dump des hashes NTDS)*

## Conclusion

Ce TP a permis de dérouler une chaîne d'attaque complète sur un environnement Active Directory : reconnaissance réseau (nmap), énumération LDAP anonyme (ldapsearch, go-windapsearch) ayant permis de récupérer des comptes et des mots de passe stockés dans les descriptions, password spraying (NetExec) confirmant plusieurs comptes valides, AS-REP Roasting et Kerberoasting permettant de casser des hashes Kerberos hors-ligne, cartographie de l'environnement avec BloodHound pour identifier les chemins d'escalade de privilèges, et enfin compromission du compte Administrateur du domaine via une attaque par dictionnaire/règles, conduisant à une exécution de code distante (psexec) et à une extraction complète de la base d'identifiants via DCSync (secretsdump).

Ce TP illustre l'importance d'une bonne hygiène Active Directory : désactiver le bind LDAP anonyme, ne jamais stocker de mots de passe dans les attributs `description`, activer la pré-authentification Kerberos, appliquer une politique de mot de passe robuste (complexité + longueur) et limiter strictement les droits de réplication (DCSync) aux comptes qui en ont réellement besoin.

---

*Document réalisé dans le cadre de la préparation à la certification eJPT — corrigé du TP "Sécurité de l'Active Directory - TP 1".*

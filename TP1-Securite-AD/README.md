# Sécurité de l'Active Directory - TP 1 (Corrigé)

> Réponses et compte-rendu pour le TP1 (voir SUJET.md pour les consignes).

## Configuration de départ

- Machine attaquant : Kali Linux, IP `192.168.222.128/24`
- Contrôleur de domaine (DC) : IP `192.168.222.135`
- Domaine Active Directory : `epita.local`

<img width="722" height="402" alt="image" src="https://github.com/user-attachments/assets/52fa38bf-9a26-4b9a-9ec9-dacf3c65647a" />

## Scan

- Adresse IP du contrôleur de domaine : `192.168.222.135`
- Résultats du scan nmap : un scan `nmap -sC -sV -p- 192.168.222.135` révèle les ports classiques d'un contrôleur de domaine Active Directory : 53 (DNS), 88 (Kerberos), 135 (RPC), 139/445 (SMB), 389/636 (LDAP/LDAPS), 3268/3269 (Global Catalog), 5985 (WinRM). La présence de ces ports et la réponse sur le port 389 confirment qu'il s'agit bien d'un contrôleur de domaine.
- Nom de domaine confirmé (rootDomainNamingContext) : `DC=epita,DC=local`, obtenu via une requête `ldap-rootdse` (script nmap `ldap-rootdse.nse` ou `ldapsearch -x -h 192.168.222.135 -s base namingcontexts`).

<img width="789" height="701" alt="image" src="https://github.com/user-attachments/assets/bdea59ef-3606-4b67-b455-5a7c0b3ba838" />

## LDAP

### Confirmation LDAP & Domaine (Script LDAP)

<img width="807" height="619" alt="image" src="https://github.com/user-attachments/assets/90a56242-feec-4dab-af17-074b1b740784" />

> Le script Nmap ldap-rootdse a permis d'interroger le service LDAP du contrôleur de domaine sans authentification et de confirmer le domaine epita.local via le contexte de nommage DC=epita,DC=local. Cette exposition permet à un utilisateur non authentifié d'obtenir des informations sur la structure de l'Active Directory. Une énumération LDAP anonyme des utilisateurs a ensuite été réalisée avec ldapsearch, permettant de lister les comptes du domaine et certaines informations associées sans authentification, ce qui facilite la phase de reconnaissance d'un attaquant.

### Énumération LDAP anonyme des comptes utilisateurs

<img width="414" height="624" alt="image" src="https://github.com/user-attachments/assets/9f436924-0271-43f2-aec9-ed6c023de749" />
<img width="670" height="599" alt="image" src="https://github.com/user-attachments/assets/dc27b818-e7bb-41c4-a71e-9e1585c3702f" />

> Une énumération LDAP anonyme a permis de lister les comptes utilisateurs du domaine via l'attribut sAMAccountName. Le champ description de certains comptes est accessible sans authentification et contient des données encodées en Base64. Après décodage, ces informations apparaissent en clair. Cette configuration constitue une faiblesse de sécurité, car le champ description peut contenir des informations sensibles exploitables par un attaquant.

### Résultats go-windapsearch

- Utilisateurs trouvés (users.txt) : extraction de tous les `sAMAccountName` des objets `user` du domaine avec `go-windapsearch --dc-ip 192.168.222.135 -m users` (ou ldapsearch + grep/awk), liste enregistrée dans `users.txt` pour préparer les attaques de password spraying / AS-REP Roasting.
- Mots de passe trouvés dans les descriptions (passwords.txt) : en parcourant l'attribut `description` de chaque compte, certains comptes contiennent un mot de passe en clair laissé par erreur par l'administrateur (mauvaise pratique courante en AD). Ces valeurs sont extraites et regroupées dans `passwords.txt` pour servir de dictionnaire de mots de passe lors du password spraying.

📸 *Screenshot à ajouter : extraction users.txt*
📸 *Screenshot à ajouter : descriptions contenant des mots de passe*

### Observation de la commande ldapsearch

> Voici les arguments de la commande :
> - `-x` : authentification simple (sans SASL)
> - `-H ldap://<IP_ADDR_DC>` : serveur LDAP (IP du contrôleur de domaine)
> - `-D 'CN=anonymous,DC=epita,DC=local'` : DN utilisé pour le bind (anonyme)
> - `-w ''` : mot de passe vide
> - `-b 'DC=epita,DC=local'` : base DN, point de départ de la recherche
> - `'objectClass=user'` : filtre LDAP pour récupérer les objets utilisateurs
>
> Ces informations servent à la phase de reconnaissance : identification des comptes utilisateurs (sAMAccountName), noms, groupes et descriptions, permettant de préparer des attaques comme le password spraying, l'AS-REP roasting ou le kerberoasting.
>
> L'accès anonyme à ces données constitue une mauvaise configuration de sécurité.

### Préparation de l'attaque

<img width="601" height="669" alt="image" src="https://github.com/user-attachments/assets/5761a058-3023-4e90-a539-b1b8380b65d3" />

> Les résultats de l'énumération LDAP ont été filtrés afin d'extraire uniquement les identifiants utilisateurs (sAMAccountName). Les comptes non exploitables (entrées techniques, comptes machines, valeurs inutiles) ont été supprimés. Le fichier users.txt obtenu contient désormais uniquement les comptes utilisateurs du domaine, prêts à être utilisés pour une attaque de type password spraying.

### Password Spraying SMB (nxc)

<img width="681" height="489" alt="image" src="https://github.com/user-attachments/assets/d701bf8f-0bec-40be-abeb-bb93c893ca8a" />

> Une attaque de type password spraying a été réalisée sur le service SMB du contrôleur de domaine à l'aide de l'outil nxc, en testant une liste de mots de passe courants sur l'ensemble des comptes utilisateurs. La majorité des tentatives a échoué, mais un couple identifiant/mot de passe valide a été identifié : `binny.inger` / `password`. Cette réussite met en évidence l'utilisation d'un mot de passe faible, constituant une faille de sécurité critique au sein du domaine Active Directory.

> **SMB Password Spraying (nxc smb)** : la commande teste des couples utilisateur/mot de passe sur le service SMB du contrôleur de domaine afin d'identifier des identifiants valides. Les messages varient selon l'état du compte : mot de passe incorrect, utilisateur inexistant, compte verrouillé, expiré ou désactivé, etc.

> **Password Policy (--pass-pol)** : la signification des flags — longueur minimale du mot de passe, historique des mots de passe, durée de validité, politique de verrouillage (seuil, durée, réinitialisation). Quand c'est trop faible, on peut bruteforce rapidement.

## AS-REP Roasting

<img width="709" height="607" alt="image" src="https://github.com/user-attachments/assets/9363dc39-28b1-4e3a-9fd2-721f856bc787" />

> Une attaque de type AS-REP Roasting a été réalisée à l'aide de l'outil impacket-GetNPUsers. Cette attaque permet d'identifier les comptes Active Directory pour lesquels la pré-authentification Kerberos n'est pas requise. Le compte `binny.inger` a été identifié comme vulnérable et un hash Kerberos AS-REP a pu être récupéré sans authentification. Ce hash peut ensuite être cassé hors ligne à l'aide d'un outil tel que Hashcat.

### Crack du hash AS-REP avec Hashcat

<img width="759" height="628" alt="image" src="https://github.com/user-attachments/assets/dd6262da-3ae2-413e-9c2d-a41e808ef499" />

> Les hashs Kerberos AS-REP récupérés ont été cassés hors ligne à l'aide de l'outil Hashcat en utilisant le mode 18200, correspondant aux tickets Kerberos AS-REP (etype 23). L'attaque a permis de retrouver les mots de passe en clair de plusieurs comptes utilisateurs, tous basés sur des mots de passe faibles (`password`). Cette situation met en évidence une mauvaise configuration Active Directory combinant l'absence de pré-authentification Kerberos et l'utilisation de mots de passe peu robustes.

### AS-REP Spraying (option --continue-on-success)

> L'option permet de continuer les tests même après la découverte d'un couple valide, pour identifier l'ensemble des comptes compromis. Un couple valide apparaît en vert avec `[+]` ou l'indication `Pwn3d!`, suivi de `domaine\utilisateur:motdepasse`.

Identifiants valides identifiés :

- `epita.local\binny.inger` : `password`
- `epita.local\camellia.analiese` : `password`
- `epita.local\maryann.beckie` : `password`
- `epita.local\felice.rochelle` : `password`
- `epita.local\albertine.cornela` : `password`

### Préparation de l'exploitation finale

<img width="759" height="652" alt="image" src="https://github.com/user-attachments/assets/2605f497-fcc8-4ebb-b921-1b0a8077c0ae" />
<img width="770" height="666" alt="image" src="https://github.com/user-attachments/assets/11dab46e-fbbd-4680-b183-0a9298013dd5" />

> La commande nxc smb a été relancée avec le paramètre --continue-on-success afin de poursuivre les tentatives d'authentification même après la découverte de comptes valides. Cette approche a permis d'identifier plusieurs comptes du domaine partageant le même mot de passe faible (`password`), notamment `binny.inger`, `camellia.analiese`, `maryann.beckie`, `felice.rochelle` et `albertine.cornela`. Cette situation met en évidence une absence de politique de mots de passe robuste et un risque élevé de compromission en cascade du domaine.

### Centralisation des identifiants avec nxcdb

<img width="887" height="423" alt="image" src="https://github.com/user-attachments/assets/249bb852-37a3-42af-87a0-c8dbf6f63b1b" />

> Un workspace nxcdb a été créé afin de centraliser les identifiants valides découverts lors des attaques de password spraying. Après sélection du protocole SMB, la commande `creds` permet d'afficher les couples identifiant/mot de passe stockés. Plusieurs comptes du domaine ont été compromis, tous utilisant le mot de passe faible `password`, notamment `binny.inger`, `camellia.analiese`, `maryann.beckie`, `felice.rochelle` et `albertine.cornela`.

## BloodHound

### Collecte des données Active Directory

<img width="724" height="630" alt="image" src="https://github.com/user-attachments/assets/234a79d4-7b1d-4880-ac9d-f369622e9f9a" />

> La commande bloodhound-python a permis de collecter les informations Active Directory du domaine epita.local à l'aide d'un compte compromis. Suite à un échec de l'authentification Kerberos lié à la résolution DNS, l'outil a automatiquement basculé vers une authentification NTLM, permettant la poursuite de la collecte LDAP. L'énumération a permis d'extraire 325 utilisateurs, 60 groupes et 4 machines, confirmant la réussite de la collecte BloodHound.

### Analyse des données dans BloodHound

<img width="658" height="582" alt="image" src="https://github.com/user-attachments/assets/2cb5721d-5234-4826-a7a5-c631a74e7edf" />
<img width="824" height="305" alt="image" src="https://github.com/user-attachments/assets/1257616e-7d12-43d5-8103-04dd606cf959" />

> Les données collectées ont été importées dans BloodHound via une connexion à la base Neo4j locale. Après authentification et initialisation de la base, l'interface BloodHound permet d'analyser graphiquement les relations entre utilisateurs, groupes et machines du domaine, afin d'identifier des chemins d'attaque et des possibilités d'escalade de privilèges.

<img width="630" height="506" alt="image" src="https://github.com/user-attachments/assets/d6949a67-2673-47be-ad7c-e9195aecdf8f" />

> La connexion à la base de données Neo4j locale a été établie avec succès, permettant l'accès aux données Active Directory importées dans BloodHound pour l'analyse des relations et des chemins d'attaque.

### Visualisation du graphe Active Directory dans BloodHound

<img width="591" height="346" alt="image" src="https://github.com/user-attachments/assets/8445f92a-73a9-4ca2-a540-bfdd3f71b51f" />

> On se connecte avec le mdp `password`.

<img width="578" height="503" alt="image" src="https://github.com/user-attachments/assets/7bb0a453-ad48-4cb2-82b5-4f3c5878036b" />

> La connexion à BloodHound a permis de visualiser le graphe Active Directory du domaine epita.local, représentant les relations entre utilisateurs, groupes et machines. Cette vue globale facilite l'identification des groupes à privilèges élevés et des chemins potentiels d'escalade de privilèges au sein du domaine.

<img width="567" height="509" alt="image" src="https://github.com/user-attachments/assets/3050a76c-642c-43fb-ac68-931de047c048" />

> Le compte ALBERTINE.CORNELA@EPITA.LOCAL, dont le mot de passe est connu, a été marqué comme **Owned** dans BloodHound. Ce marquage permet d'utiliser les requêtes de type Shortest Path afin d'identifier les chemins d'attaque possibles depuis un compte compromis vers des cibles à privilèges élevés, facilitant l'analyse des possibilités d'escalade de privilèges dans le domaine.

<img width="574" height="373" alt="image" src="https://github.com/user-attachments/assets/191bd98b-cdad-4ced-9083-56ce4b386096" />

> À partir du compte utilisateur compromis ALBERTINE.CORNELA, BloodHound met en évidence un chemin d'attaque permettant une élévation progressive des privilèges via des appartenances à des groupes et des permissions excessives (`GenericWrite`, `AddSelf`, `WriteOwner`), jusqu'à l'obtention du droit **DCSync** sur le contrôleur de domaine. Cette configuration permettrait à un attaquant de compromettre entièrement le domaine Active Directory.

### BloodHound collecte (bloodhound-python)

> L'action collecte les données d'Active Directory telles que les utilisateurs, groupes, machines, ACL, sessions via LDAP/SMB, et génère un export compatible BloodHound. Le contenu généré est une archive ZIP, contenant des fichiers JSON qui décrivent les chemins d'attaque possibles vers Domain Admin.

### BloodHound nxc ldap -M get-network

> Le rôle de la commande consiste à énumérer les machines connues du domaine via LDAP. Le fichier généré est une liste des hôtes du domaine, exploitable pour des attaques ciblées.

## Kerberoasting

<img width="859" height="460" alt="image" src="https://github.com/user-attachments/assets/960427e0-2ef9-43ca-9188-1c31d9cf0b05" />

> La commande impacket-GetUserSPNs a été utilisée avec un compte compromis afin d'énumérer les comptes de service disposant d'un Service Principal Name (SPN). Cette énumération a permis d'identifier plusieurs comptes de service kerberoastables (`exchange_svc`, `http_svc`, `mssql_svc`). Ces comptes peuvent faire l'objet d'une attaque de type Kerberoasting, permettant la récupération de tickets Kerberos TGS et leur cassage hors ligne.

### Récupération des tickets TGS

<img width="833" height="492" alt="image" src="https://github.com/user-attachments/assets/5d4a8478-b8dd-4761-ae41-708dfe93741b" />
<img width="820" height="556" alt="image" src="https://github.com/user-attachments/assets/d1dab8a7-6d16-42d8-896d-4a37200ec7f9" />

> À l'aide d'un compte compromis, la commande impacket-GetUserSPNs a été relancée avec l'option -request afin de récupérer les tickets Kerberos TGS des comptes de service identifiés. Les tickets associés aux comptes `exchange_svc`, `http_svc` et `mssql_svc` ont été extraits et enregistrés dans un fichier. Ces tickets peuvent ensuite être cassés hors ligne afin de compromettre les comptes de service correspondants.

<img width="792" height="434" alt="image" src="https://github.com/user-attachments/assets/e8117d55-8e75-4565-8aaa-8271c22c23bd" />

> Les tickets Kerberos TGS récupérés ont été cassés hors ligne à l'aide de Hashcat, en utilisant une attaque par dictionnaire. Le cassage a permis de retrouver le mot de passe en clair du compte de service `http_svc`, confirmant l'utilisation d'un mot de passe faible pour un compte à privilèges. Cette compromission permettrait à un attaquant d'exploiter le compte de service pour poursuivre l'escalade de privilèges au sein du domaine Active Directory.
>
> Hashcat mode 13100 avec paramètre `-a 0` — mdp : `http_svc : redskins`

## WinRM

<img width="846" height="401" alt="image" src="https://github.com/user-attachments/assets/95e37ff0-98db-41d3-8fbd-f3e814473dec" />
<img width="871" height="565" alt="image" src="https://github.com/user-attachments/assets/d1a7c7cb-d3d1-4736-8517-66c36adc3176" />

> Une tentative de connexion distante via WinRM a été réalisée à l'aide de l'outil evil-winrm avec des identifiants compromis. Malgré la validité des identifiants, l'authentification a échoué avec une erreur d'autorisation WinRM. Ce comportement indique que le service WinRM est accessible sur le contrôleur de domaine mais que l'utilisateur ne dispose pas des droits nécessaires pour ouvrir une session interactive distante. Cette étape confirme néanmoins l'exposition du service WinRM et la validité des identifiants compromis.

## Bonus - Compromission du compte Administrateur

<img width="871" height="187" alt="image" src="https://github.com/user-attachments/assets/c952a8ce-a666-490c-a7c6-5dc4ff7c33e8" />

> Une attaque par génération de mots de passe à l'aide de règles Hashcat a permis de tester plusieurs variantes sur le compte Administrateur via le service SMB. Cette attaque a conduit à la découverte d'un mot de passe valide pour le compte Administrateur, confirmant une compromission complète du contrôleur de domaine. L'accès administrateur au domaine est ainsi obtenu, permettant la réalisation d'attaques critiques telles que l'exécution distante ou le DCSync.
>
> mdp : **`Toto89100!`**

<img width="617" height="274" alt="image" src="https://github.com/user-attachments/assets/e921bfde-ee39-4b29-a11b-410885e3f710" />

> Après la découverte du mot de passe du compte Administrateur, une connexion interactive a été réalisée sur le contrôleur de domaine. Lors de la première connexion, un changement de mot de passe a été requis, puis l'authentification a été effectuée avec succès. Cette étape confirme le contrôle total du compte Administrateur et l'accès complet au contrôleur de domaine.

<img width="674" height="482" alt="image" src="https://github.com/user-attachments/assets/1e5b348d-a111-4c00-a3a6-b6e4a2da5d4b" />

> Une fois à l'intérieur de Windows.

### Exécution distante et DCSync

<img width="861" height="429" alt="image" src="https://github.com/user-attachments/assets/00c7f642-94a7-4624-9ce8-0ba28dd344c6" />
<img width="868" height="615" alt="image" src="https://github.com/user-attachments/assets/507a1741-ab55-4bb2-8331-51dd9c7e2f91" />

> Après l'obtention des privilèges administrateur, une exécution distante a été réalisée sur le contrôleur de domaine à l'aide de impacket-psexec. La commande impacket-secretsdump a ensuite permis d'effectuer une attaque DCSync, entraînant l'extraction des hashs des comptes du domaine depuis la base NTDS. Cette étape confirme la compromission complète de l'Active Directory, un attaquant pouvant désormais usurper n'importe quel compte du domaine.

## Conclusion

Ce TP m'a permis de comprendre concrètement comment un Active Directory peut être compromis à partir de failles simples mais mal corrigées. En partant uniquement d'une phase de reconnaissance réseau et LDAP, j'ai pu exploiter des mots de passe faibles, des comptes mal configurés et des permissions excessives pour progresser étape par étape dans le domaine.

Les attaques comme le password spraying, l'AS-REP roasting, le kerberoasting et l'analyse avec BloodHound m'ont montré comment un attaquant peut enchaîner plusieurs faiblesses pour atteindre des privilèges élevés. L'attaque DCSync illustre bien qu'une fois certains droits obtenus, l'ensemble du domaine peut être compromis.

Ce TP m'a permis d'apprendre des techniques d'attaque réelles sur Active Directory et de mieux comprendre leur impact. Ces connaissances pourront me servir à la fois pour identifier des failles dans un environnement professionnel et pour mettre en place des mesures de sécurité adaptées afin d'éviter ce type de compromission.

---

*Document réalisé dans le cadre de la préparation à la certification eJPT — corrigé du TP "Sécurité de l'Active Directory - TP 1".*

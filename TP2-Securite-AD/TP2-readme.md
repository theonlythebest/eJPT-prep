# Sécurité de l'Active Directory - TP 2 (Corrigé)

> Réponses et compte-rendu pour le TP2 (voir SUJET.md pour les consignes).

## Scan

<img width="770" height="355" alt="image" src="https://github.com/user-attachments/assets/2cab1dd4-9534-48bf-8d36-377bb7798189" />

> Dans un premier temps, nous avons lancé les différentes machines virtuelles nécessaires au TP : la machine Kali Linux, qui sera utilisée comme machine d'attaque, ainsi que les deux machines Windows contenant l'environnement Active Directory.
>
> Afin d'identifier les adresses IP des machines présentes sur le réseau privé, nous avons utilisé l'outil nbtscan. Cet outil permet de scanner un réseau et d'identifier les machines Windows en récupérant notamment leur nom NetBIOS. Grâce à ça nous savons l'adressse Ip de Kali, Dc, et Tiny

## Password Spraying

**Question** : Interprétez et expliquez les attributs : lockoutDuration, lockoutThreshold et lockoutObservationWindow

> Ces attributs correspondent à la politique de verrouillage des comptes configurée dans l'Active Directory. Le paramètre lockoutThreshold indique le nombre de tentatives de connexion incorrectes autorisées avant que le compte soit verrouillé. Le paramètre lockoutDuration correspond à la durée pendant laquelle un compte reste bloqué après avoir été verrouillé. Enfin, lockoutObservationWindow représente la période pendant laquelle les tentatives de connexion sont comptabilisées avant d'être réinitialisées. Dans notre cas, la valeur lockoutThreshold est égale à 0, ce qui signifie qu'aucun verrouillage de compte n'est activé. Cela permet de tester un grand nombre de mots de passe sans bloquer les comptes utilisateurs, ce qui rend les attaques de password spraying beaucoup plus faciles à réaliser

<img width="687" height="425" alt="image" src="https://github.com/user-attachments/assets/786a15a0-2500-41c5-8b93-ae2caaa7da7a" />

> Dans cette étape, nous utilisons l'outil windapsearch pour interroger le contrôleur de domaine avec une requête LDAP et récupérer des informations sur la configuration du domaine. On observe notamment les attributs lockoutDuration, lockoutThreshold et lockoutObservationWindow, qui correspondent à la politique de verrouillage des comptes. Ici, la valeur lockoutThreshold est 0, ce qui signifie qu'aucun verrouillage de compte n'est activé. Cela veut dire qu'on peut essayer plusieurs mots de passe sans bloquer les comptes utilisateurs. Cette configuration est donc peu sécurisée et permet de réaliser plus facilement une attaque de password spraying.

**Question** : Que fait la commande : `nxc ldap <DC_IP> -u '' -p '' -M get-desc-users`

> Cette commande utilise l'outil NetExec (nxc) pour interroger le service LDAP du contrôleur de domaine Active Directory. Le module get-desc-users permet de récupérer les descriptions associées aux comptes utilisateurs. Ces descriptions peuvent parfois contenir des informations sensibles, comme des mots de passe par défaut ou des indications sur la configuration des comptes. Dans notre environnement, certaines descriptions contiennent des phrases comme "Company default password (Reset ASAP)" ou "New user generated password", ce qui peut donner des indices sur les mots de passe utilisés par certains comptes du domaine.

**Question** : Que fait la commande suivante et à quoi correspondent les caractères `?d?d?d` ?
`hashcat -a 6 potential.txt ?d?d?d --force --stdout > default_password.txt`

> Cette commande utilise Hashcat pour générer une liste de mots de passe potentiels à partir d'un mot de base. L'option -a 6 correspond à une attaque hybride qui combine un dictionnaire avec un masque. Le fichier potential.txt contient le mot de base utilisé pour générer les mots de passe. Les caractères ?d?d?d représentent trois chiffres allant de 0 à 9 qui seront ajoutés à la fin du mot. La commande génère donc des combinaisons comme epita001, epita123 ou epita999. L'option --stdout affiche les mots générés dans la sortie standard et le symbole > default_password.txt permet d'enregistrer tous les mots de passe générés dans un fichier.

## Récupération des utilisateurs du domaine

<img width="811" height="446" alt="image" src="https://github.com/user-attachments/assets/089b4925-b340-4860-846a-eb51c6b9016b" />

> Dans cette étape, nous utilisons l'outil windapsearch afin de récupérer la liste des utilisateurs présents dans le domaine Active Directory. Pour cela, nous exécutons une requête LDAP sur le contrôleur de domaine avec l'option -m users --full. Cette commande permet d'obtenir plusieurs informations sur chaque utilisateur du domaine, comme son nom, son identifiant ou encore certains attributs liés au compte. Ces informations vont ensuite nous permettre de créer une liste d'utilisateurs qui sera utilisée pour les prochaines attaques comme le password spraying.

## Extraction des utilisateurs et des mots de passe

<img width="495" height="431" alt="image" src="https://github.com/user-attachments/assets/5dad5cba-f908-420b-9c97-d8f02a5bf864" />
<img width="567" height="290" alt="image" src="https://github.com/user-attachments/assets/15c9d372-fa07-4260-a163-94a334586170" />
<img width="527" height="356" alt="image" src="https://github.com/user-attachments/assets/3e89e86a-72bc-4246-be64-4b3fcde4df31" />

> Après avoir récupéré les informations des utilisateurs du domaine, nous analysons le fichier users_dump.txt afin de rechercher des informations intéressantes dans le champ description. Pour cela, nous utilisons la commande grep afin d'afficher toutes les descriptions présentes dans le fichier. On remarque que certaines descriptions contiennent des informations sensibles comme "New user generated password" ou "Company default password (Reset ASAP)", ce qui peut indiquer des mots de passe par défaut ou générés automatiquement.
>
> Ensuite, nous extrayons les noms des utilisateurs avec la commande grep sAMAccountName, puis nous les enregistrons dans un fichier usernames.txt. Ce fichier servira plus tard pour tester des mots de passe lors des attaques. De la même manière, nous extrayons les descriptions contenant les mots de passe potentiels dans un fichier passwords_raw.txt. Ces informations permettront de construire une liste de mots de passe pour les attaques de password spraying.

## Extraction des utilisateurs et mots de passe

<img width="763" height="243" alt="image" src="https://github.com/user-attachments/assets/54d39f0b-b835-46b4-8fc6-b27203885885" />
<img width="815" height="208" alt="image" src="https://github.com/user-attachments/assets/88b89293-351b-4943-bd1f-56873801bba5" />
<img width="442" height="348" alt="image" src="https://github.com/user-attachments/assets/c8b12a2c-0fbd-489e-9f10-5eadb716590c" />

> Dans cette étape, nous cherchons à récupérer les mots de passe qui apparaissent dans les descriptions des utilisateurs. Pour cela, nous utilisons la commande grep afin de filtrer les lignes contenant "New user generated password" dans le fichier users_dump.txt. On remarque que certaines descriptions contiennent directement des mots de passe générés automatiquement pour les nouveaux comptes.
>
> Ensuite, nous utilisons la commande cut pour extraire uniquement les mots de passe et les enregistrer dans un fichier passwords.txt. Nous avons également une liste des utilisateurs dans le fichier usernames.txt. Ces deux fichiers vont ensuite être utilisés pour réaliser une attaque de password spraying sur les comptes du domaine

## Installation de Kerbrute

<img width="528" height="209" alt="image" src="https://github.com/user-attachments/assets/eff57d08-af94-44e5-921d-ed9966443803" />

### Énumération des utilisateurs

<img width="543" height="268" alt="image" src="https://github.com/user-attachments/assets/8980780f-382f-443d-9d0f-1027465be30c" />

> Dans cette étape, nous utilisons l'outil Kerbrute afin de vérifier quels utilisateurs existent réellement dans le domaine Active Directory. Tout d'abord, nous rendons l'exécutable utilisable avec la commande chmod +x. Ensuite, nous lançons la commande userenum avec le domaine et la liste d'utilisateurs obtenue précédemment. La première tentative échoue car le contrôleur de domaine n'est pas précisé. Nous relançons donc la commande en ajoutant l'option --dc pour indiquer l'adresse IP du contrôleur de domaine. Kerbrute va alors tester chaque nom d'utilisateur et afficher ceux qui sont valides dans le domaine. Cette étape permet de confirmer quels comptes existent avant de lancer l'attaque de password spraying.

## Mots de passe et énumération LDAP

<img width="501" height="264" alt="image" src="https://github.com/user-attachments/assets/757888f7-65ab-4781-8258-631f34f6c5ea" />
<img width="747" height="355" alt="image" src="https://github.com/user-attachments/assets/89dcf49c-b10c-40f2-a313-9ea143516637" />

> Après avoir récupéré la liste des utilisateurs du domaine, nous utilisons l'outil Kerbrute afin de tester un mot de passe sur l'ensemble des comptes. Cette technique s'appelle le password spraying et consiste à essayer un même mot de passe sur plusieurs utilisateurs afin d'identifier ceux qui utilisent des mots de passe faibles ou par défaut. Dans les résultats, Kerbrute affiche plusieurs VALID LOGIN, ce qui signifie que certains comptes acceptent ce mot de passe. Pour continuer l'analyse, nous utilisons ensuite l'outil NetExec (nxc) afin d'interroger le service LDAP du contrôleur de domaine. Grâce au module get-desc-users, il est possible de récupérer les descriptions associées aux comptes utilisateurs. On remarque que certaines descriptions contiennent des informations intéressantes comme Company default password (Reset ASAP) ou encore New user generated password. Ces informations peuvent donner des indications sur des mots de passe utilisés dans le domaine et peuvent être exploitées dans la suite de l'attaque.

## Bruteforce Kerberos et test des mots de passe

<img width="776" height="462" alt="image" src="https://github.com/user-attachments/assets/692b216e-bc3f-4399-856c-2a256d9e3ced" />
<img width="727" height="292" alt="image" src="https://github.com/user-attachments/assets/6f14e318-e348-4b44-965b-fd9d1badde06" />

> Dans cette étape, nous utilisons Kerbrute pour effectuer un bruteforce Kerberos en utilisant un fichier combo.txt contenant des combinaisons utilisateur:mot de passe. L'outil teste chaque combinaison sur le contrôleur de domaine afin de trouver des identifiants valides. Dans les résultats, plusieurs connexions valides apparaissent avec un mot de passe de type epitaXXX, ce qui indique que certains comptes utilisent un mot de passe basé sur le nom du domaine.
>
> Puis, nous utilisons NetExec (nxc) afin de tester ces identifiants sur le service SMB du contrôleur de domaine. On remarque que plusieurs comptes retournent le statut STATUS_PASSWORD_MUST_CHANGE. Cela signifie que le mot de passe est correct mais que l'utilisateur est obligé de le modifier avant de pouvoir se connecter normalement. Cette situation est fréquente lorsque des comptes sont créés avec un mot de passe par défaut qui doit être changé lors de la première connexion.

## Changement de mot de passe des comptes

<img width="719" height="414" alt="image" src="https://github.com/user-attachments/assets/40f56ae6-8208-4172-80f5-b3c490f62614" />
<img width="789" height="242" alt="image" src="https://github.com/user-attachments/assets/53675a80-c275-418a-8c18-409f9d21188b" />

> Comme certains comptes avaient le statut STATUS_PASSWORD_MUST_CHANGE, il était nécessaire de modifier leur mot de passe avant de pouvoir les utiliser normalement. Pour cela, nous utilisons l'outil impacket-changepasswd qui permet de changer un mot de passe à distance sur le contrôleur de domaine. Nous remplaçons donc le mot de passe initial par un nouveau mot de passe respectant la politique de sécurité du domaine.
>
> Une fois les mots de passe modifiés, nous utilisons l'outil NetExec (nxc) pour tester les nouveaux identifiants sur le service SMB du contrôleur de domaine. Les résultats montrent que plusieurs comptes peuvent maintenant s'authentifier correctement, ce qui confirme que les mots de passe ont bien été changés et que ces comptes peuvent être utilisés pour les prochaines étapes de l'attaque.

## Accès RDP au poste client

<img width="851" height="79" alt="image" src="https://github.com/user-attachments/assets/dea71c23-a0b1-4b03-bc30-88a368209ea5" />

> Après avoir obtenu des identifiants valides, nous testons l'accès au poste client via le protocole RDP (Remote Desktop Protocol). Pour cela, nous utilisons l'outil NetExec (nxc) avec l'option rdp, qui permet de vérifier si l'utilisateur peut se connecter à distance. La commande est exécutée avec l'adresse IP du poste client ainsi que les identifiants récupérés précédemment. Le résultat indique que la connexion est valide pour l'utilisateur cyndie.evvie, ce qui confirme que ce compte possède les droits nécessaires pour accéder au bureau à distance. L'option --screenshot permet également de récupérer une capture d'écran du poste distant, ce qui prouve que l'accès RDP fonctionne correctement.

## Collecte d'informations avec SharpHound / BloodHound

<img width="818" height="550" alt="image" src="https://github.com/user-attachments/assets/ec7587b4-f09f-4ef4-a522-3d8e9d529d76" />
<img width="752" height="166" alt="image" src="https://github.com/user-attachments/assets/7ae068c4-b758-405a-b1ba-17692efea992" />

> Après avoir obtenu un accès au poste client via RDP, nous utilisons l'outil SharpHound afin de collecter des informations sur l'environnement Active Directory. Pour cela, nous lançons un petit serveur web sur la machine Kali avec python3 -m http.server afin de transférer le fichier SharpHound.exe vers la machine Windows. Depuis la session PowerShell du poste client, nous téléchargeons ensuite l'outil avec la commande Invoke-WebRequest. Une fois téléchargé, SharpHound peut être exécuté afin de collecter différentes informations sur le domaine, comme les relations entre utilisateurs, groupes et machines. Ces données seront ensuite utilisées dans BloodHound pour analyser les chemins d'attaque possibles dans l'Active Directory.

**Question** : Que fait la commande : `SharpHound.exe -c All`

> Cette commande lance l'outil SharpHound afin de collecter un maximum d'informations sur l'environnement Active Directory. L'option -c All indique à SharpHound de récupérer toutes les données disponibles, notamment les utilisateurs, les groupes, les ordinateurs, les sessions actives et les permissions entre les objets du domaine. Toutes ces informations sont ensuite enregistrées dans un fichier compressé qui pourra être importé dans l'outil BloodHound afin d'analyser les relations entre les objets Active Directory et d'identifier d'éventuels chemins d'escalade de privilèges.

**Question** : Comparez les résultats entre First Degree Object Control, Group Delegated Object Control et Transitive Object Control.

> Ces trois options permettent d'analyser les relations de contrôle entre les objets Active Directory. First Degree Object Control affiche les objets sur lesquels un utilisateur possède un contrôle direct. Group Delegated Object Control montre les permissions obtenues via les groupes auxquels l'utilisateur appartient. Enfin, Transitive Object Control affiche les relations indirectes permettant d'atteindre un autre objet en passant par plusieurs étapes. Cette analyse permet d'identifier les chaînes d'attaque possibles qui peuvent mener à une escalade de privilèges vers des comptes administrateurs du domaine.

**Question** : Consultez l'onglet Windows Abuse et expliquez ce qui est indiqué.

> L'onglet Windows Abuse explique comment exploiter une permission spécifique dans Active Directory. Dans notre cas, il indique que l'utilisateur possède la permission GenericAll sur un autre compte. Cette permission correspond à un contrôle total sur l'objet cible et permet notamment de modifier le mot de passe de l'utilisateur, modifier ses permissions ou l'ajouter à des groupes privilégiés. Dans le TP, cette permission est exploitée pour changer le mot de passe du compte STELLA.ARABELE et prendre le contrôle de ce compte.

## Collecte des données Active Directory avec BloodHound

<img width="796" height="379" alt="image" src="https://github.com/user-attachments/assets/4ff7fdff-8ae4-4294-aaaa-e6e29db624f5" />

> Dans cette étape, nous utilisons l'outil bloodhound-python depuis la machine Kali afin de collecter des informations sur l'environnement Active Directory. Nous nous authentifions avec un compte utilisateur valide et nous indiquons l'adresse IP du contrôleur de domaine. L'outil interroge ensuite le service LDAP afin de récupérer différentes informations comme les utilisateurs, les groupes, les ordinateurs et les relations entre eux. À la fin de l'exécution, toutes les données collectées sont compressées dans un fichier .zip. Ce fichier pourra ensuite être importé dans l'outil BloodHound, qui permet de visualiser graphiquement les relations entre les objets Active Directory et d'identifier des chemins d'attaque possibles vers des comptes à privilèges élevés.

## Import des données dans BloodHound

<img width="517" height="235" alt="image" src="https://github.com/user-attachments/assets/c53d3785-d105-4bfd-8577-ada74f87737f" />
<img width="558" height="490" alt="image" src="https://github.com/user-attachments/assets/c61d99b6-3487-48e6-b98c-3dd54bd691ab" />

> Après qu'on a récupéré les données de l'active directory avec bloodhound-python, nous extrayons le fichier zip généré puis nous lançons l'outil BloodHound.

<img width="776" height="456" alt="image" src="https://github.com/user-attachments/assets/d42fefd2-477e-4017-9185-b74d4baa1a49" />

> Ici, Nous lançons bloodhound

<img width="547" height="422" alt="image" src="https://github.com/user-attachments/assets/f4c4c220-d53a-4b56-bcf6-6bb35b9887bf" />
<img width="628" height="259" alt="image" src="https://github.com/user-attachments/assets/3ae0fae7-3f4a-43c4-93e5-8677946962d9" />

> Nous changeons de mdp :
> neo4j
> NouveauMotDePasse123!

## Analyse du graphe avec BloodHound

<img width="523" height="354" alt="image" src="https://github.com/user-attachments/assets/3fd240a1-0237-4648-aca5-45250437696a" />
<img width="547" height="381" alt="image" src="https://github.com/user-attachments/assets/e17ec177-b0d5-4061-ac9f-6101533f5fc6" />
<img width="545" height="346" alt="image" src="https://github.com/user-attachments/assets/da9da19c-1ea9-455e-8dba-848c4210cb1d" />

> Après avoir importé les données dans BloodHound, nous pouvons analyser les relations entre les différents objets de l'Active Directory. L'outil affiche un graphe qui montre les liens entre les utilisateurs, les groupes et les machines du domaine. Grâce à cette visualisation, il est possible d'identifier les chemins permettant d'obtenir des privilèges plus élevés. Dans notre cas, l'analyse du graphe montre que certains utilisateurs possèdent des relations avec des groupes ayant des privilèges importants comme Domain Admins. Cela signifie que si un attaquant compromet ces comptes, il peut potentiellement obtenir des droits administrateur sur le domaine et prendre le contrôle de l'infrastructure Active Directory.

### Analyse des comptes utilisateurs et accès au poste client

<img width="600" height="273" alt="image" src="https://github.com/user-attachments/assets/16bb2f3d-8bed-4817-819f-9c913e3c1459" />
<img width="615" height="465" alt="image" src="https://github.com/user-attachments/assets/4d77ddc9-d675-4293-930d-8bde2f6134bd" />

> j'ai changé le mdp c'est Epita123!
>
> Dans BloodHound, nous pouvons consulter les informations détaillées des comptes utilisateurs du domaine. On remarque que certaines descriptions contiennent directement des mots de passe générés automatiquement pour les nouveaux comptes. Cela montre que des informations sensibles peuvent parfois être stockées dans les attributs des utilisateurs, ce qui représente un risque de sécurité.
>
> Après avoir obtenu des identifiants valides, nous nous connectons ensuite au poste client Windows. Cette connexion permet de vérifier que les identifiants récupérés fonctionnent correctement et que l'accès au système est possible. Une fois connecté, nous pouvons continuer l'analyse de l'environnement Active Directory depuis la machine cliente afin de préparer les prochaines étapes de l'attaque. Nous avons accès a Alysa

## Utilisation de PowerView et analyse des ACL

<img width="681" height="388" alt="image" src="https://github.com/user-attachments/assets/81416a2d-4c31-43ea-9b01-997dde47de9e" />
<img width="669" height="187" alt="image" src="https://github.com/user-attachments/assets/faa76e48-d162-47bd-b2e5-da42d72efde9" />

> La commande Find-InterestingDomainAcl montre que l'utilisateur ALYSA.ARETHA possède le droit GenericAll sur l'utilisateur STELLA.ARABELE. Ce droit donne un contrôle complet sur ce compte. Nous exploitons donc cette permission en changeant le mot de passe de STELLA.ARABELE avec la commande net user, ce qui permet de prendre le contrôle de ce compte et de poursuivre l'escalade de privilèges.

**Question** : Pourquoi le script PowerView ne fonctionne pas au départ ?

> Le script ne fonctionne pas au départ car PowerShell bloque l'exécution des scripts téléchargés pour des raisons de sécurité. Cette restriction est définie par la politique d'exécution (Execution Policy) de PowerShell. Par défaut, les scripts externes ne peuvent pas être exécutés afin d'éviter l'exécution de programmes malveillants. Il est donc nécessaire de modifier cette politique pour autoriser l'exécution du script PowerView.

**Question** : Expliquez la commande : `Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser`

> Cette commande modifie la politique d'exécution de PowerShell afin d'autoriser l'exécution des scripts. L'option ExecutionPolicy Bypass permet d'exécuter des scripts sans appliquer les restrictions de sécurité habituelles. L'option Scope CurrentUser indique que cette modification ne s'applique qu'à l'utilisateur actuel et non à l'ensemble du système. Cela permet donc d'exécuter le script PowerView tout en limitant l'impact de la modification sur la configuration globale du système.

**Question** : Que fait exactement la commande suivante et pourquoi est-elle utile en test d'intrusion ?

> Cette commande télécharge et exécute le script PowerView directement en mémoire. La commande iwr (Invoke-WebRequest) permet de télécharger le script depuis un serveur web hébergé sur la machine Kali. Le symbole | iex (Invoke-Expression) exécute immédiatement le script téléchargé. Cette technique est souvent utilisée en test d'intrusion car elle permet d'exécuter des outils sans les enregistrer sur le disque de la machine cible, ce qui rend l'attaque plus discrète et plus difficile à détecter.

**Question** : Que fait la commande : `Find-InterestingDomainAcl | ? {$_.ActiveDirectoryRights -eq "GenericAll" -and $_...}`

> Cette commande utilise PowerView pour analyser les ACL (Access Control Lists) présentes dans l'Active Directory. Elle filtre les résultats afin de trouver les objets pour lesquels l'utilisateur ALYSA.ARETHA possède la permission GenericAll. Cette permission correspond à un contrôle total sur l'objet ciblé. Dans notre cas, cette commande permet d'identifier que l'utilisateur ALYSA.ARETHA possède un contrôle complet sur le compte STELLA.ARABELE, ce qui permet d'exploiter cette permission pour changer son mot de passe et poursuivre l'escalade de privilèges.

## Vérification des informations du compte

<img width="682" height="161" alt="image" src="https://github.com/user-attachments/assets/0867beda-b8a8-4bd3-a18f-a808ed950753" />

> Après avoir pris le contrôle du compte STELLA.ARABELE, nous utilisons les commandes PowerView Get-DomainObjectAcl et Get-DomainUser afin de vérifier les informations associées à cet utilisateur dans l'Active Directory. Les résultats montrent notamment les groupes auxquels appartient le compte, comme le groupe Project management, ce qui permet de mieux comprendre ses permissions dans le domaine.

<img width="592" height="431" alt="image" src="https://github.com/user-attachments/assets/4ec380c2-a4f1-477a-a12c-ad6441ab9778" />
<img width="757" height="507" alt="image" src="https://github.com/user-attachments/assets/1e3372e6-6492-45ea-bda0-46ba2990d3df" />

> Ensuite, nous utilisons la commande Get-DomainGroupMember afin d'afficher les utilisateurs appartenant au groupe Project management. Cela permet de voir quels comptes font partie de ce groupe dans le domaine. Nous utilisons également la commande Find-InterestingDomainAcl pour analyser les permissions associées à ce groupe. Les résultats montrent que le groupe possède certains droits sur différents objets du domaine, ce qui peut être exploité pour poursuivre l'escalade de privilèges dans l'Active Directory.

## AD Miner

### Audit de sécurité Active Directory

<img width="733" height="534" alt="image" src="https://github.com/user-attachments/assets/02bd28dd-959f-47de-b121-a738904afac2" />
<img width="743" height="135" alt="image" src="https://github.com/user-attachments/assets/f5f7c21c-d23d-496f-8303-10c22ffa1871" />

> Nous utilisons l'outil AD Miner afin d'analyser la sécurité du domaine Active Directory. Cet outil se connecte à la base Neo4j contenant les données récupérées avec BloodHound et examine les objets du domaine comme les utilisateurs, les groupes, les ordinateurs et les relations entre eux.
>
> AD Miner effectue plusieurs vérifications pour détecter des problèmes de sécurité, par exemple des permissions anormales, des ACL suspectes ou des comptes pouvant être exploités. À la fin de l'analyse, l'outil génère automatiquement un rapport dans le dossier render_My_Report, qui permet d'identifier plus facilement les faiblesses potentielles de l'infrastructure Active Directory.

<img width="668" height="490" alt="image" src="https://github.com/user-attachments/assets/02cd0fa8-84e4-4468-a598-83c84f1892dc" />

> Après l'exécution d'AD Miner, un rapport est généré et peut être ouvert dans un navigateur. Ce rapport présente une analyse globale des risques de sécurité dans l'Active Directory. On peut y voir différents indicateurs comme les risques critiques, les risques potentiels et les risques mineurs. Le rapport indique un niveau de risque CRITICAL,

### Chemins d'escalade vers Domain Admin

<img width="613" height="187" alt="image" src="https://github.com/user-attachments/assets/dba90d44-7b30-4bb5-ac63-ad44486c5c23" />

> L'analyse montre qu'il existe un chemin permettant d'augmenter les privilèges jusqu'au niveau Domain Admin. Ce type de chemin repose sur certaines permissions comme GenericAll ou WriteOwner, qui peuvent être utilisées pour prendre le contrôle d'autres comptes et progresser dans l'escalade de privilèges au sein du domaine.

## Exploitation des ACL / Reset de mot de passe

<img width="634" height="56" alt="image" src="https://github.com/user-attachments/assets/90181834-8262-4cda-a2bf-cff4590d30e8" />
<img width="440" height="471" alt="image" src="https://github.com/user-attachments/assets/cc8d148e-f087-4680-91d3-5de4c2dfff0e" />

> Cette capture montre l'exploitation d'un droit GenericAll présent dans l'Active Directory. Grâce à cette permission, l'utilisateur ALYSA.ARETHA possède un contrôle complet sur le compte STELLA.ARABELE. La commande net user STELLA.ARABELE Epita123! /domain est utilisée pour modifier son mot de passe directement depuis PowerShell. Le message "The command completed successfully" confirme que le mot de passe a été changé avec succès sur le contrôleur de domaine.

## Enumération avec PowerView

<img width="637" height="311" alt="image" src="https://github.com/user-attachments/assets/18cc7226-9601-4fb5-9de0-c481041109a8" />
<img width="748" height="144" alt="image" src="https://github.com/user-attachments/assets/3c9b3a75-8630-45ca-8786-928defa4ee5a" />

> Cette capture montre l'utilisation de l'outil PowerView dans PowerShell afin d'explorer l'Active Directory. La politique d'exécution est d'abord modifiée avec Set-ExecutionPolicy Bypass pour permettre l'exécution du script. Ensuite, le module PowerView.ps1 est importé et la commande Get-Domain permet d'afficher les informations du domaine epita.local. Enfin, les résultats indiquent que l'utilisateur STELLA.ARABELE est membre du groupe IT Helpdesk, ce qui peut donner certains droits supplémentaires dans l'environnement Active Directory.

<img width="698" height="327" alt="image" src="https://github.com/user-attachments/assets/2ddd8bae-06f4-4f50-97a3-3878bc8ee874" />

> Cette capture montre l'utilisation de la commande Get-DomainUser dominica* du module PowerView. Cette commande permet de rechercher les utilisateurs du domaine dont le nom correspond au motif dominica. Les informations détaillées du compte Dominica Datha sont ensuite affichées, comme son identifiant, son nom dans le domaine et son appartenance à certains groupes. Cette étape permet de collecter des informations sur les comptes Active Directory afin d'identifier d'éventuelles opportunités d'attaque ou d'escalade de privilèges.

<img width="644" height="593" alt="image" src="https://github.com/user-attachments/assets/a40cc921-6bcf-41ac-b6fb-52586ef273b2" />

> Après l'import du module PowerView, une nouvelle session PowerShell est utilisée pour exécuter des commandes d'énumération dans l'Active Directory. La commande Get-DomainUser dominica* permet de rechercher les utilisateurs dont le nom commence par dominica. Les informations du compte Dominica Datha sont alors affichées, ce qui permet d'obtenir des détails sur cet utilisateur dans le domaine epita.local.

<img width="663" height="377" alt="image" src="https://github.com/user-attachments/assets/b85a7ddb-480c-4f01-bd9d-60fc4cd2c3a8" />

> Au départ, l'exécution du script PowerView.ps1 est bloquée par la politique de sécurité de PowerShell. Pour autoriser l'exécution du script, la commande Set-ExecutionPolicy Bypass -Scope Process est utilisée. Cette commande permet de désactiver temporairement la restriction uniquement pour la session PowerShell en cours. Une fois la restriction levée, le script PowerView peut être lancé et la commande Get-Domain permet d'afficher les informations du domaine epita.local.

## Post-exploitation / Dump des hashes Active Directory

<img width="596" height="168" alt="image" src="https://github.com/user-attachments/assets/a854b764-4ee0-404a-8cd1-be2fe403b7bc" />
<img width="668" height="464" alt="image" src="https://github.com/user-attachments/assets/e3482651-fb58-46dd-8786-b2b9a0415a70" />

> Les identifiants du compte dominica.datha sont d'abord vérifiés avec l'outil nxc smb, ce qui confirme que l'authentification fonctionne sur le contrôleur de domaine. Ensuite, la commande impacket-secretsdump est utilisée afin d'extraire la base NTDS.dit du contrôleur de domaine. Cette opération permet de récupérer les hashs NTLM des comptes du domaine, y compris ceux d'utilisateurs privilégiés comme Administrator ou krbtgt, ce qui représente un accès très critique dans l'Active Directory.
>
> L'attaque commence par la compromission d'un premier compte utilisateur dans l'Active Directory lors des phases d'énumération et de récupération d'identifiants. Cela permet ensuite d'obtenir un accès au compte STELLA.ARABELE. Grâce aux permissions présentes dans le domaine, cet utilisateur est ajouté au groupe IT Helpdesk, ce qui lui donne des droits supplémentaires. Une ACL mal configurée est alors exploitée, permettant d'utiliser les droits WriteOwner, puis GenericAll, afin de modifier le mot de passe du compte dominica.datha. Une fois le mot de passe changé, une connexion est réalisée avec ce compte pour poursuivre l'attaque dans le domaine. L'outil impacket-secretsdump est ensuite utilisé afin d'extraire la base NTDS du contrôleur de domaine. Cette opération permet de récupérer les hashs NTLM de l'ensemble des comptes du domaine, y compris ceux des comptes administrateurs. Le résultat final du TP est la récupération du hash du compte Domain Administrator, ce qui démontre une compromission complète de l'Active Directory.

## Zerologon

### Exploitation Zerologon

<img width="724" height="332" alt="image" src="https://github.com/user-attachments/assets/b81a94eb-50a4-4c34-b0a8-76ac4013d011" />
<img width="695" height="578" alt="image" src="https://github.com/user-attachments/assets/7a201750-f8d3-44a2-9b16-8e5781191063" />

> La vulnérabilité Zerologon (CVE-2020-1472) est exploitée afin de compromettre le contrôleur de domaine. Le script set_empty_pw.py est utilisé pour modifier le mot de passe du compte machine du contrôleur de domaine en un mot de passe vide. Une fois cette étape réalisée, la commande impacket-secretsdump est exécutée avec l'option -no-pass afin d'effectuer une attaque DCSync. Cette opération permet de récupérer la base NTDS et d'extraire les hashs NTLM de tous les comptes du domaine, y compris ceux des comptes administrateurs et du compte krbtgt.

**Question** : Expliquez la vulnérabilité Zerologon et ce que fait l'exploit.

> Zerologon est une vulnérabilité critique du protocole Netlogon utilisée pour l'authentification entre les machines Windows et le contrôleur de domaine. Cette vulnérabilité exploite une faiblesse dans l'algorithme de chiffrement utilisé par Netlogon, ce qui permet à un attaquant non authentifié de se connecter au contrôleur de domaine après plusieurs tentatives d'authentification. L'exploit set_empty_pw.py utilise cette faille pour réinitialiser le mot de passe du compte machine du contrôleur de domaine à une valeur vide. Une fois cette opération réalisée, l'attaquant peut effectuer une attaque DCSync et récupérer les hashs NTLM de tous les comptes du domaine.

**Question** : Quel compte est utilisé et pourquoi le hash est `31d6cfe0d16ae931b73c59d7e0c089c0` ?

> Le compte utilisé est le compte machine du contrôleur de domaine, qui est identifié par le nom du serveur suivi du symbole $. Après l'exploitation de la vulnérabilité Zerologon, le mot de passe de ce compte est réinitialisé à une valeur vide. Le hash 31d6cfe0d16ae931b73c59d7e0c089c0 correspond au hash NTLM d'un mot de passe vide. Cela permet ensuite d'utiliser l'outil secretsdump pour effectuer une attaque DCSync et récupérer les identifiants de tous les comptes du domaine.

### Post-exploitation / Accès au contrôleur de domaine

<img width="715" height="384" alt="image" src="https://github.com/user-attachments/assets/93054a08-55f4-413c-9e4d-ebd57cf7b19f" />

> Après avoir récupéré le hash NTLM de l'administrateur, l'outil impacket-psexec est utilisé afin d'obtenir un accès distant au contrôleur de domaine. L'authentification est réalisée avec l'option -hashes, ce qui permet de se connecter sans connaître le mot de passe en clair. Une fois la connexion établie, la commande whoami confirme que l'accès est obtenu avec les privilèges NT AUTHORITY\SYSTEM, ce qui correspond au niveau de privilège le plus élevé sur la machine.

**Question** : Combien de mots de passe avez-vous pu récupérer et comment en obtenir encore plus ?

> Nous avons récupéré plusieurs identifiants utilisateurs ainsi que plusieurs hashs NTLM appartenant aux comptes du domaine, notamment ceux de comptes importants comme Administrator et krbtgt. Au total, cela représente plusieurs comptes du domaine (plus de 5). Pour obtenir encore plus de mots de passe en clair, il est possible d'utiliser des outils comme Hashcat afin de casser les hashs NTLM récupérés. Ces outils permettent d'effectuer des attaques par dictionnaire ou par règles afin de retrouver les mots de passe utilisés par les utilisateurs du domaine.

## Conclusion

> Ce TP m'a permis de mieux comprendre la différence entre la théorie et la pratique en cybersécurité. En réalisant les différentes étapes, j'ai pu voir concrètement comment une attaque peut être menée dans un environnement Active Directory. J'ai notamment compris les mécanismes d'énumération, d'exploitation de mauvaises configurations et d'élévation de privilèges. Cela m'a permis de mieux comprendre comment fonctionnent ces attaques en pratique et je sais maintenant comment reproduire ces manipulations dans un environnement de test.

---

*Document réalisé dans le cadre de la préparation à la certification eJPT / corrigé du TP "Sécurité de l'Active Directory - TP 2".*

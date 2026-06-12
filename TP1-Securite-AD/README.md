# Securite de l'Active Directory - TP 1 (Corrige)

> Reponses et compte-rendu pour le TP1 (voir SUJET.md pour les consignes).

## Scan

- Adresse IP du controleur de domaine :
- Resultats du scan nmap :
- Nom de domaine confirme (rootDomainNamingContext) :

## LDAP

### Observer la commande ldapsearch

- Explication des arguments :
- Utilite des informations retournees :

### Resultats go-windapsearch

- Utilisateurs trouves (users.txt) :
- Mots de passe trouves dans les descriptions (passwords.txt) :

### Observer la commande nxc smb (password spraying)

- Que realise cette commande ?
- Pourquoi certains messages d'erreur sont differents des autres ?

### Politique de mot de passe (--pass-pol)

- Flags de complexite retournes :

## AS-REP Roasting

- Compte avec mot de passe modifie :
- Resultat impacket-GetNPUsers :
- Hash AS-REP casse (hashcat mode 18200) :

### Observer la commande nxc --continue-on-success

- A quoi sert --continue-on-success ?
- Comment est represente un couple identifiant/mot de passe valide ?
- Identifiants/mots de passe valides trouves :

## BloodHound

### Installation

- Notes d'installation (BloodHound Legacy, Neo4j, bloodhound.py) :

### Collecte (bloodhound-python / nxc ldap)

- Qu'est-ce que la commande a fait ?
- Contenu du fichier genere :
- Resultat de la commande -M get-network :

### Analyse dans BloodHound

- Requetes renvoyant des informations non vides :
- Resultats interessants observes :
- SPN kerberoastables listes :
- Chemins "Shortest Path from Owned Principals" trouves :

## Kerberoasting

- Resultat impacket-GetUserSPNs :
- Mode et parametre hashcat utilises :
- Mots de passe decouverts :

## WinRM

- Resultat password spraying WinRM :

### Observer evil-winrm

- A quoi sert evil-winrm ?
- Qu'est-ce que la commande a fait ?

## Bonus - Compromission du compte Administrateur

- Wordlist generee (rules hashcat) :
- Mot de passe Administrateur trouve :
- Resultat impacket-psexec :
- Resultat impacket-secretsdump (DCSync) :

## Conclusion

- Synthese de la chaine d'attaque complete :
- Recommandations / remediations :

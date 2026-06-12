# Sécurité de l'Active Directory - TP 2 (Corrigé)

> Réponses et compte-rendu pour le TP2 (voir SUJET.md pour les consignes).

## Configuration de départ

- Machine attaquant (Kali) :
- VM 1 (Windows Server 2012) :
- VM 2 (Tiny10) :
- Domaine Active Directory :

## Password Spraying et Énumération

- Valeurs de lockoutDuration / lockoutThreshold / lockoutObservationWindow :
- Réponse Question 2.1 (interprétation des attributs de verrouillage) :
- Utilisateurs récupérés (users.txt) :
- Mots de passe récupérés depuis les descriptions (passwords.txt) :
- Résultat kerbrute userenum :
- Résultat kerbrute passwordspray :
- Réponse Question 2.2 (nxc ldap -M get-desc-users) :

## Force Brute et Variations de Mots de Passe

- Comptes avec "Company default password(Reset ASAP)" :
- Mot de passe potentiel de base utilisé :
- Réponse Question 3.1 (commande hashcat -a 6 et ?d?d?d) :
- Résultat kerbrute bruteforce (mot de passe epitaXXX trouvé) :
- Comptes en STATUS_PASSWORD_MUST_CHANGE :
- Script / commandes de changement de mot de passe (impacket-changepasswd) :
- Résultat nxc après changement des mots de passe :

## Accès RDP et Collecte BloodHound

- Utilisateurs membres de "Utilisateurs du Bureau à distance" :
- Résultat nxc rdp --screenshot :
- Connexion RDP (xfreerdp) :
- Réponse Question 4.1 (SharpHound.exe -c All) :
- Récupération du ZIP BloodHound :

## Analyse BloodHound et PowerView

- Mot de passe d'ALYSA.ARETHA (depuis sa description) :
- Réponse Question 5.1 (First/Group Delegated/Transitive Object Control) :
- Réponse Question 5.2 (lien GenericAll -> Windows Abuse) :
- Nouveau mot de passe de STELLA.ARABELE :
- Réponse Question 5.3 (script non signé) :
- Réponse Question 5.4 (Set-ExecutionPolicy Bypass) :
- Réponse Question 5.5 (iwr ... | iex) :
- Réponse Question 5.6 (Find-InterestingDomainAcl GenericAll alysa.aretha) :

## AD Miner et Graphes

- Installation et exécution AD Miner :
- Chemin observé vers Domain Admins :
- Comparaison avec BloodHound (Shortest Paths to Unconstrained Delegation Systems) :

## Exploitation des ACLs (Escalade)

### Étape A : STELLA.ARABELE → IT Helpdesk (GenericAll)

- Commandes PowerView utilisées :
- Réponse Question 7.1 (explication des commandes) :
- Résultat Get-DomainGroupMember :

### Étape B : IT Helpdesk → DOMINICA.DATHA (WriteOwner)

- Set-DomainObjectOwner :
- Add-DomainObjectAcl :
- Nouveau mot de passe de DOMINICA.DATHA :

## DCSync et Zerologon

- Réponse Question 8.1 (impacket-secretsdump avec DOMINICA.DATHA) :
- Création du compte "superadmin" :
- Réponse Question 8.2 (commandes de persistance) :
- Réponse Question 8.3 (vulnérabilité Zerologon / set_empty_pw.py) :
- Résultat de l'exploitation Zerologon :
- Réponse Question 8.4 (compte machine et hash 31d6cfe0d16ae931b73c59d7e0c089c0) :

## Bonus - Post-Exploitation et Golden Ticket

- Réponse Question 9.1 (DCSync global, nombre de mots de passe récupérés) :
- Forgeage du Golden Ticket (impacket-ticketer) :
- Résultat impacket-secretsdump avec le ticket :

## Audit de configuration avec PingCastle

- Score global PingCastle :
- Non-conformités identifiées et lien avec les attaques réalisées :
- Analyse de la section Trusts (relation Parent-Enfant) :
- Comptes vulnérables identifiés (Kerberoasting, descriptions en clair) :
- Politique de mots de passe observée :

### Exploration des ACLs

- Arêtes exploitées (ForceChangePassword / SelfAdd / WriteDACL) :

## Conclusion

---

*Document réalisé dans le cadre de la préparation à la certification eJPT — corrigé du TP "Sécurité de l'Active Directory - TP 2".*

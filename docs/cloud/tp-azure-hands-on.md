---
icon: material/flask-outline
title: "TP Azure : Hands-on (TP18 et TP19)"
tags: [cloud, azure, tp, windows server, active directory, ad ds, durcissement, sécurité]
---

# 🧪 TP Azure : Hands-on (TP18 et TP19)

!!! abstract "Ce que tu vas faire"
    Windows Server et AD DS dans Azure, puis le durcissement d'une image et d'une VM.

**Niveau :** 🟡 Intermédiaire · **Durée :** 3 h 30 · **Outil :** Azure for Students

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP18 — Windows Server et AD DS sur Azure (Hands-on Azure)

**Objectif** : « création d'un Azure AD, d'un Windows Server, déploiement de quelques rôles » (p. 121) — tu connais AD DS ; ici, l'enjeu est ce qui change quand le contrôleur de domaine vit dans Azure. **Slides du cours** p. 51-55, 121. **Coût** < 1 $ (licence Windows incluse, VM désallouée entre deux séances).

1. VNet 10.60.0.0/16, VM `vm-dc01` Windows Server 2022 en `Standard_B2s`, **sans** RDP ouvert sur Internet. Fixe son IP privée (10.60.1.4) côté Azure, pas dans Windows.
2. Connecte-toi en RDP par Azure Bastion Developer (gratuit, dans le navigateur).
3. Ajoute un disque de données (cache hôte « Aucun ») pour la base NTDS et SYSVOL.
4. Promeus le serveur : nouvelle forêt `lab.cesi.local`, DNS intégré, base sur le disque de données.
5. Configure le VNet pour utiliser 10.60.1.4 comme serveur DNS, puis joins une seconde VM Windows au domaine.
6. Mets en place un modèle AGDLP minimal (OU, utilisateurs, groupes globaux, groupe de domaine local, partage).

**Validation** : la seconde VM est membre du domaine ; un utilisateur du groupe global accède au partage, un autre non.

**Questions** : compare AD DS et Entra ID (protocoles p. 51 : Kerberos/NTLM/LDAP vs OIDC/OAuth2/SAML). Pourquoi l'IP du DC se fixe-t-elle dans Azure et pas dans la carte Windows ? Pourquoi le cache hôte doit-il être désactivé sur le disque NTDS ? Qu'apporterait Entra Connect, et pourquoi ne peux-tu pas l'installer sur le tenant de l'école (droits d'administrateur général) ?

**Nettoyage** : `az group delete -n rg-tp18`.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-tp18; PW='<mot de passe fort>'
    az group create -n $RG -l $LOC --tags tp=18 owner=admin
    az network vnet create -g $RG -n vnet-ad --address-prefixes 10.60.0.0/16 \
      --subnet-name snet-ad --subnet-prefixes 10.60.1.0/24

    # IP privée fixée côté Azure ; disque de données sans cache hôte pour NTDS/SYSVOL
    az vm create -g $RG -n vm-dc01 --image Win2022Datacenter --size Standard_B2s \
      --vnet-name vnet-ad --subnet snet-ad --private-ip-address 10.60.1.4 \
      --public-ip-address "" --nsg "" --data-disk-sizes-gb 8 --data-disk-caching None \
      --admin-username labadmin --admin-password "$PW"
    ```

    Connexion : portail → `vm-dc01` → Se connecter → Bastion → Bastion Developer, RDP avec `labadmin`. Dans PowerShell (administrateur) :

    ```powershell
    Get-Disk | Where-Object PartitionStyle -eq 'RAW' |
      Initialize-Disk -PartitionStyle GPT -PassThru |
      New-Partition -DriveLetter F -UseMaximumSize |
      Format-Volume -FileSystem NTFS -NewFileSystemLabel ADDS -Confirm:$false

    Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
    Install-ADDSForest -DomainName "lab.cesi.local" -DomainNetbiosName "LAB" -InstallDns `
      -DatabasePath "F:\NTDS" -LogPath "F:\NTDS" -SysvolPath "F:\SYSVOL" `
      -SafeModeAdministratorPassword (Read-Host -AsSecureString "Mot de passe DSRM") -Force
    ```

    Après le redémarrage, depuis ton PC :

    ```bash
    az network vnet update -g $RG -n vnet-ad --dns-servers 10.60.1.4
    az vm create -g $RG -n vm-srv01 --image Win2022Datacenter --size Standard_B2s \
      --vnet-name vnet-ad --subnet snet-ad --public-ip-address "" --nsg "" \
      --admin-username labadmin --admin-password "$PW"
    ```

    Sur `vm-srv01` : `Add-Computer -DomainName lab.cesi.local -Credential LAB\labadmin -Restart`. AGDLP minimal, sur `vm-dc01` :

    ```powershell
    $base = "DC=lab,DC=cesi,DC=local"
    New-ADOrganizationalUnit -Name "LAB" -Path $base
    "Utilisateurs","Groupes" | ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=LAB,$base" }
    $pw = Read-Host -AsSecureString "Mot de passe des comptes de test"
    "alice","bob" | ForEach-Object {
      New-ADUser -Name $_ -SamAccountName $_ -Path "OU=Utilisateurs,OU=LAB,$base" -AccountPassword $pw -Enabled $true
    }
    New-ADGroup -Name "GG_Compta" -GroupScope Global -Path "OU=Groupes,OU=LAB,$base"
    New-ADGroup -Name "DL_Partage_Compta_RW" -GroupScope DomainLocal -Path "OU=Groupes,OU=LAB,$base"
    Add-ADGroupMember -Identity GG_Compta -Members alice
    Add-ADGroupMember -Identity DL_Partage_Compta_RW -Members GG_Compta
    ```

    Sur `vm-srv01` :

    ```powershell
    New-Item -Path C:\Partages\Compta -ItemType Directory
    New-SmbShare -Name Compta -Path C:\Partages\Compta -ChangeAccess "LAB\DL_Partage_Compta_RW"
    icacls C:\Partages\Compta /grant "LAB\DL_Partage_Compta_RW:(OI)(CI)M"
    ```

    Test depuis `vm-dc01` : `net use \\vm-srv01\Compta /user:LAB\alice` réussit, la même commande avec `bob` renvoie « Accès refusé ».

    Réponses : AD DS parle Kerberos, NTLM et LDAP sur un réseau de confiance, avec OU et GPO ; Entra ID parle OIDC, OAuth2 et SAML sur HTTPS, sans OU ni GPO (remplacées par les groupes, l'accès conditionnel et Intune). Dans Azure, c'est toujours le DHCP de la plateforme qui distribue l'IP de la carte : une IP saisie à la main dans Windows peut contredire Azure et couper la VM ; on fixe l'IP dans Azure et on laisse Windows en DHCP. Le cache d'écriture de l'hôte peut perdre des écritures en cas de panne : pour une base AD, c'est le risque de corruption ou de restauration USN, d'où un disque dédié sans cache. Entra Connect synchroniserait les comptes AD vers Entra ID (l'AD reste la source, p. 54) ; il exige un compte administrateur d'identité hybride ou général sur le tenant, ce que l'école ne te donnera pas — il faudrait un tenant de test personnel.

## TP19 — Durcissement d'une image et d'une VM (Hands-on DENV-R)

**Objectif** : « utilisation d'image pré-installée conteneur → faire un état des lieux, hardening » (p. 121). **Coût** 0 $ si tu le fais en local, sinon une VM B1s pendant une heure.

1. Analyse les vulnérabilités de `nginx:latest`, `nginx:alpine` et de l'image Keycloak du TP7 avec un scanner (Trivy ou Grype, version épinglée). Compare le nombre de CVE critiques et hautes.
2. Écris un Dockerfile durci pour une page statique : image de base minimale, utilisateur non root, port > 1024, pas d'outils inutiles. Rescanne.
3. Lance-le avec `--read-only`, `--cap-drop ALL` et `--security-opt no-new-privileges`, et vérifie qu'il fonctionne encore.
4. Sur une VM Ubuntu, lance `lynis audit system`, note l'indice de durcissement, corrige 5 points (SSH sans mot de passe ni root, mises à jour automatiques, pare-feu local, etc.) et relance.

**Validation** : moins de CVE sur ton image que sur `nginx:latest` ; l'indice Lynis augmente.

**Questions** : quels risques de la p. 44 (ISO ou images infectées, workloads de confiance différente) ces mesures traitent-elles ? Pourquoi un scan à un instant T ne suffit-il pas ? Où placer le scan dans une chaîne CI/CD ?

**Nettoyage** : `docker image prune`, suppression de la VM si utilisée.

??? success "✅ Corrigé et commandes"

    Scan d'images (en local, avec Docker) :

    ```bash
    TRIVY=aquasec/trivy:<version stable épinglée>      # ou anchore/grype:<version>
    for img in nginx:latest nginx:alpine quay.io/keycloak/keycloak:latest; do
      echo "== $img"
      docker run --rm -v /var/run/docker.sock:/var/run/docker.sock $TRIVY \
        image --severity HIGH,CRITICAL --quiet "$img" | grep -E "^Total"
    done
    ```

    Image durcie (`Dockerfile` + un `index.html` à côté) :

    ```dockerfile
    FROM nginxinc/nginx-unprivileged:alpine
    COPY --chown=101:101 index.html /usr/share/nginx/html/index.html
    USER 101
    EXPOSE 8080
    ```

    ```bash
    docker build -t site-durci .
    docker run -d --name site -p 8080:8080 --read-only --tmpfs /tmp \
      --cap-drop ALL --security-opt no-new-privileges site-durci
    curl -s localhost:8080
    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock $TRIVY image --severity HIGH,CRITICAL --quiet site-durci
    ```

    Durcissement d'une VM Ubuntu (avec IP publique, pour installer les paquets) :

    ```bash
    sudo apt-get update && sudo apt-get install -y lynis
    sudo lynis audit system --quick | tee lynis-avant.txt | grep -i "hardening index"

    sudo tee /etc/ssh/sshd_config.d/99-durcissement.conf <<'EOF'
    PermitRootLogin no
    PasswordAuthentication no
    X11Forwarding no
    MaxAuthTries 3
    ClientAliveInterval 300
    EOF
    sudo systemctl restart ssh
    sudo apt-get install -y unattended-upgrades auditd
    sudo dpkg-reconfigure -f noninteractive unattended-upgrades
    sudo ufw allow 22/tcp && sudo ufw --force enable

    sudo lynis audit system --quick | tee lynis-apres.txt | grep -i "hardening index"
    ```

    Réponses : le scan et des registres de confiance traitent le risque « image infectée » de la p. 44 ; non-root, `--cap-drop ALL`, `--read-only` et `no-new-privileges` limitent ce qu'un conteneur compromis peut faire à l'hôte et aux workloads voisins de confiance différente. Un scan n'est vrai qu'au moment où il tourne : de nouvelles CVE sortent chaque jour, donc on rescanne en continu dans le registre et on reconstruit régulièrement. Dans une chaîne CI/CD : au build (échec si CRITICAL), à l'envoi dans le registre, à l'admission au déploiement, puis en continu sur ce qui tourne.

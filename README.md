# 🔐 Windows LAPS — Mots de passe administrateur local uniques dans Active Directory

> Implémentation et validation de **Windows LAPS** (Local Administrator Password Solution) :
> chaque poste reçoit un mot de passe administrateur local **unique, complexe et renouvelé
> automatiquement**, stocké chiffré dans Active Directory, avec délégation au moindre privilège.

![OS](https://img.shields.io/badge/Windows%20Server-2025-0078D6?logo=windows&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active%20Directory-AD%20DS-1f6feb)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?logo=powershell&logoColor=white)
![Sécurité](https://img.shields.io/badge/Sécurité-Moindre%20privilège-success)
![Statut](https://img.shields.io/badge/Statut-Projet%20livré-brightgreen)

---

## 📋 Sommaire

- [Contexte & objectif](#-contexte--objectif)
- [Pourquoi LAPS](#-pourquoi-laps)
- [Environnement](#-environnement)
- [Prérequis](#-prérequis)
- [Mise en œuvre](#-mise-en-œuvre)
- [Vérification — la preuve](#-vérification--la-preuve)
- [Bonnes pratiques & points de vigilance](#-bonnes-pratiques--points-de-vigilance)
- [Compétences démontrées](#-compétences-démontrées)

---

## 🎯 Contexte & objectif

Réutiliser le **même mot de passe administrateur local** sur tous les postes d'un parc est
l'une des failles les plus exploitées en entreprise (déplacement latéral d'un attaquant via
*Pass-the-Hash*). L'objectif de ce mini-projet est de supprimer ce risque en déployant
Windows LAPS.

**Résultat attendu** — chaque poste possède un compte `Administrator` local avec un mot de
passe **unique de 20 caractères**, renouvelé tous les 30 jours, stocké et chiffré dans AD,
et lisible uniquement par le service IT habilité.

**Commande de preuve finale :**
```powershell
Get-LapsADPassword -Identity WIN-CLI-01 -AsPlainText
```

## 💡 Pourquoi LAPS

| Sans LAPS | Avec LAPS |
| --------- | --------- |
| Même mot de passe admin local partout | Mot de passe **unique** par machine |
| Compromission d'un poste = tout le parc exposé | Compromission isolée à un seul poste |
| Rotation manuelle, rarement faite | Rotation **automatique** (30 jours) |
| Mots de passe dans des fichiers/tableurs | Stockés **chiffrés dans Active Directory** |
| Accès non tracé | Lecture **déléguée au moindre privilège** |

## 🖥️ Environnement

| Composant            | Détail                                          |
| -------------------- | ----------------------------------------------- |
| Contrôleur de domaine| `SRV-DC01` — Windows Server 2025                 |
| Domaine AD           | `ad.alpitech.ch`                                |
| Poste client         | `WIN-CLI-01` — Windows 11                        |
| Unité d'organisation | `OU=Postes,OU=Ordinateurs,OU=Entreprise`        |
| Groupe IT délégué    | `GG_IT` (lecture / réinitialisation des mots de passe) |

> **Légende des étapes** : 🖱️ *INTERFACE* = action à la souris · ⌨️ *POWERSHELL* = commande (admin).

## ✅ Prérequis

- Forêt Active Directory fonctionnelle et droits **Schema Admins** (pour l'extension de schéma).
- Module PowerShell `LAPS` présent (intégré à Windows Server 2025 / Windows 11).
- Module `GroupPolicy` (RSAT) sur le contrôleur de domaine.
- OU dédiée aux postes et groupe de sécurité IT déjà créés.

---

## ⚙️ Mise en œuvre

### Étape 0 — Ranger le poste dans l'OU `Postes`
Une GPO liée à l'OU `Postes` ne s'applique qu'aux machines qui s'y trouvent.

```powershell
$base   = (Get-ADDomain).DistinguishedName
$postes = "OU=Postes,OU=Ordinateurs,OU=Entreprise,$base"
Get-ADComputer "WIN-CLI-01" | Move-ADObject -TargetPath $postes
Get-ADComputer "WIN-CLI-01" | Select-Object Name, DistinguishedName
```

> ⚠️ Tant que le poste reste dans le conteneur `Computers`, la GPO LAPS ne l'atteint pas.

![Déplacement du poste dans l'OU Postes](screenshots/00-move-ou.png)

### Étape 1 — Étendre le schéma Active Directory
Ajoute les attributs `ms-LAPS-*` nécessaires au stockage des mots de passe.

```powershell
Update-LapsADSchema

# Vérification des attributs ajoutés
Get-ADObject -SearchBase (Get-ADRootDSE).schemaNamingContext `
  -Filter 'name -like "ms-LAPS-*"' | Select-Object Name
```

> ⚠️ Extension **irréversible**, à réaliser **une seule fois par forêt** avec les droits
> *Schema Admins*. La commande est silencieuse si le schéma est déjà étendu.

![Attributs ms-LAPS présents dans le schéma](screenshots/01-schema.png)

### Étape 2 — Autoriser les postes à écrire leur mot de passe
Chaque machine de l'OU pourra déposer son mot de passe LAPS dans AD.

```powershell
Set-LapsADComputerSelfPermission -Identity $postes
```

![Permission self appliquée sur l'OU Postes](screenshots/02-self-permission.png)

### Étape 3 — Déléguer la lecture à `GG_IT` (moindre privilège)
Le service IT pourra **lire et réinitialiser** les mots de passe sans être administrateur du domaine.

```powershell
Set-LapsADReadPasswordPermission  -Identity $postes -AllowedPrincipals "AD\GG_IT"
Set-LapsADResetPasswordPermission -Identity $postes -AllowedPrincipals "AD\GG_IT"
```

> 💡 LAPS exige un **nom qualifié** : utiliser `AD\GG_IT` (NetBIOS), pas `GG_IT` seul.

![Délégation lecture / réinitialisation à GG_IT](screenshots/03-delegation.png)

### Étape 4 — Créer et configurer la GPO LAPS
Indique aux postes de générer un mot de passe local et de le stocker dans AD.

```powershell
Import-Module GroupPolicy
$g = New-GPO -Name "GPO-LAPS"
$k = "HKLM\Software\Microsoft\Policies\LAPS"

Set-GPRegistryValue -Name $g.DisplayName -Key $k -ValueName "BackupDirectory"    -Type DWord -Value 2
Set-GPRegistryValue -Name $g.DisplayName -Key $k -ValueName "PasswordLength"     -Type DWord -Value 20
Set-GPRegistryValue -Name $g.DisplayName -Key $k -ValueName "PasswordComplexity" -Type DWord -Value 4
Set-GPRegistryValue -Name $g.DisplayName -Key $k -ValueName "PasswordAgeDays"    -Type DWord -Value 30
```

| Paramètre            | Valeur | Signification                          |
| -------------------- | ------ | -------------------------------------- |
| `BackupDirectory`    | `2`    | Stockage dans **Active Directory**     |
| `PasswordLength`     | `20`   | Longueur du mot de passe               |
| `PasswordComplexity` | `4`    | Majuscules + minuscules + chiffres + symboles |
| `PasswordAgeDays`    | `30`   | Renouvellement tous les 30 jours       |

![GPO-LAPS créée et paramétrée](screenshots/04-gpo.png)

### Étape 5 — Lier la GPO et l'appliquer sur le client

**Sur `SRV-DC01` :**
```powershell
New-GPLink -Name "GPO-LAPS" -Target $postes -LinkEnabled Yes
```

**Sur `WIN-CLI-01` (PowerShell admin) :**
```powershell
gpupdate /force
Invoke-LapsPolicyProcessing
```

> 💡 Si la vérification renvoie un résultat vide au premier essai, relancer
> `gpupdate /force` puis `Invoke-LapsPolicyProcessing` et patienter : LAPS a parfois
> besoin d'un cycle.

![GPO-LAPS liée à l'OU Postes](screenshots/05-gplink.png)
![Application de la stratégie côté client](screenshots/05-client-apply.png)

---

## 🔎 Vérification — la preuve

```powershell
Get-LapsADPassword -Identity WIN-CLI-01 -AsPlainText
```

La sortie confirme : compte `Administrator` géré, mot de passe **unique de 20 caractères**,
`DecryptionStatus: Success`, date d'expiration à 30 jours, déchiffrement autorisé pour le
groupe habilité → **Windows LAPS est pleinement opérationnel.**

![Preuve finale : mot de passe LAPS lu depuis AD](screenshots/06-proof.png)

> 🔒 **Sur la capture de preuve, masquez le mot de passe en clair avant publication.**
> Ne jamais diffuser un secret réel sur un dépôt public.

---

## 🛡️ Bonnes pratiques & points de vigilance

- **Moindre privilège** : la lecture des mots de passe est réservée à `GG_IT`, pas aux *Domain Admins* par défaut.
- **Extension de schéma** : opération unique, irréversible — à planifier et documenter.
- **Nom qualifié obligatoire** pour les principals (`AD\GG_IT`).
- **Audit** : activer la journalisation des lectures de mots de passe LAPS pour la traçabilité.
- **Pas de secret en clair** dans le dépôt : captures redactées, aucun mot de passe réel commité.

## 🧠 Compétences démontrées

- Administration **Active Directory** (OU, schéma, délégation de permissions)
- **Sécurisation du parc** Windows : suppression du risque de mot de passe admin partagé
- Conception et déploiement de **GPO** via PowerShell
- Application du principe de **moindre privilège**
- **Automatisation PowerShell** (AD DS, GroupPolicy, modules LAPS)
- Démarche complète : conception → déploiement → **vérification et preuve**

---

*Procédure illustrée — Windows LAPS · Sécurité · Administration systèmes & réseaux*

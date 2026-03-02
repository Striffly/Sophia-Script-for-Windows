# Analyse comparative : AtlasOS Playbooks vs Sophia Script

> **Date :** Mars 2026  
> **Périmètre :** Tweaks Atlas non couverts par Sophia Script, évalués pour un PC gaming/VR sous Windows 11 Pro

---

## Résumé exécutif

Ce document liste les tweaks pertinents **non couverts (ou partiellement couverts) par AtlasOS** et complémentaires à Sophia Script, pour un PC gaming/VR sous Windows 11 Pro.

### Bilan de vérification (source code Atlas + Sophia)

| #   | Tweak                    | Couvert par Atlas ?                                                               | Couvert par Sophia ?              | Verdict                                 |
| --- | ------------------------ | --------------------------------------------------------------------------------- | --------------------------------- | --------------------------------------- |
| 1   | TargetReleaseVersion     | **Oui — mais BUG** (écrase DWORD par String)                                      | Non                               | **ADDITIF** — corrige le bug Atlas      |
| 2   | AutoLogger-Diagtrack ETW | **Partiellement** — Atlas cible `Diagtrack-Listener` (sans préfixe `AutoLogger-`) | Non (service DiagTrack seulement) | **ADDITIF** — cible la vraie clé kernel |
| 3   | Clipboard cloud sync     | **Non**                                                                           | **Non**                           | **UNIQUE** ✓                            |
| 4   | NetworkThrottlingIndex   | **Non**                                                                           | **Non**                           | **UNIQUE** ✓                            |
| 5   | GlobalTimerResolution    | **Opt-in** — present mais désactivé par défaut                                    | Non                               | **CONDITIONNELLEMENT ADDITIF**          |
| 6   | DisablePagingExecutive   | **Commenté** dans `tweaks.yml` (ligne 63)                                         | Non                               | **ADDITIF** (usage gaming dédié)        |
| 7   | Edge Startup Boost       | **Non**                                                                           | **Non**                           | **UNIQUE** ✓                            |
| 8   | Nagle Algorithm disable  | **Non**                                                                           | **Non**                           | **UNIQUE** ✓                            |

> **Légende :** UNIQUE = absent des deux projets. ADDITIF = corrige ou complète Atlas. CONDITIONNELLEMENT ADDITIF = présent dans Atlas mais opt-in (non appliqué par défaut).

### Tweaks déconseillés pour un usage desktop

Ces tweaks d'AtlasOS sont **à éviter** pour un PC personnel non dédié exclusivement au gaming :

| Tweak                                | Raison de l'éviter                                          |
| ------------------------------------ | ----------------------------------------------------------- |
| `DisablePagingExecutive`             | Peut causer des `BSOD` si la RAM est saturée                |
| `DisablePageCombining`               | Augmente la consommation RAM, risqué avec < 32 Go           |
| `Service Host Splitting Disable`     | Peut casser des services système, difficile à diagnostiquer |
| `Fault Tolerant Heap (FTH) Disable`  | Applications instables peuvent crasher plus souvent         |
| `bcdedit /set bootmenupolicy legacy` | Remplace le beau menu de démarrage par le mode texte        |

## 1 - Verrouiller la version de Windows (pas de Feature Upgrade forcé)

> **Source Atlas :** `qol/windows-update/disable-feature-updates.yml`
>
> **Différence vs Atlas :** Atlas utilise le bon registre `TargetReleaseVersion` (DWORD = 1) puis
> **écrase cette valeur** avec une commande PowerShell qui réécrit `TargetReleaseVersion` en String
> (= la version courante), au lieu de créer la clé distincte `TargetReleaseVersionInfo`.
> Le code ci-dessous utilise l'implémentation correcte : `TargetReleaseVersion = 1` (DWORD) +
> `TargetReleaseVersionInfo` (String) conformément à la documentation Microsoft (admx.help).
>
> **✅ Vérifié :** `disable-feature-updates.yml` ligne 13 confirme le bug — `New-ItemProperty -Name 'TargetReleaseVersion' -Value $ver -PropertyType String` écrase le DWORD. **Ce tweak est ADDITIF.**

Bloque le passage automatique aux nouvelles versions majeures de Windows 11 (ex: 24H2 → 25H2). Sophia ne fait pas cela. Très utile pour ne pas se retrouver avec une nouvelle version qui réinitialise des tweaks.

```powershell
$CurrentVersion = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion").DisplayVersion
If (-not (Test-Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate")) { New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force | Out-Null }
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "TargetReleaseVersion" -Value 1 -Type DWord -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "ProductVersion" -Value "Windows 11" -Type String -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "TargetReleaseVersionInfo" -Value $CurrentVersion -Type String -Force
```

**Risque :** Faible. Les mises à jour de sécurité continuent normalement. Seules les mises à niveau vers une nouvelle version annuelle sont bloquées. Vous pouvez manuellement mettre à jour quand vous le souhaitez.

---

## 2 - AutoLogger-Diagtrack ETW session (kernel-level)

> **Source Atlas :** `privacy/telemetry/disallow-data-collection.yml` (partiel — path corrigé)
>
> **Différence vs Atlas :** Atlas désactive `Diagtrack-Listener` (sans le préfixe `AutoLogger-`).
> Ce script cible le bon chemin `AutoLogger-Diagtrack-Listener` qui est la session ETW
> AutoLogger réelle dans Windows 10/11. Les deux clés peuvent coexister, mais celle-ci
> est la plus importante car elle démarre au niveau kernel avant les services.
>
> **✅ Vérifié :** `disallow-data-collection.yml` ligne 54 confirme que Atlas cible
> `HKLM\...\Autologger\Diagtrack-Listener` (Start=0), **pas** `AutoLogger-Diagtrack-Listener`.
> Ce sont deux clés registre distinctes. **Ce tweak est ADDITIF** — il complète Atlas en ciblant la session ETW kernel.

Sophia désactive le service DiagTrack. Mais l'enregistreur ETW `AutoLogger-Diagtrack-Listener` démarre au niveau **kernel**, _avant_ les services Windows, et peut logger des données même si DiagTrack est arrêté. Cette clé désactive ce démarrage automatique.

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\WMI\Autologger\AutoLogger-Diagtrack-Listener" -Name "Start" -Value 0 -Type DWord -Force
```

**Risque :** Aucun. Renforce simplement le disable DiagTrack de Sophia.

---

## 3 - Clipboard cloud sync disable

> **Source Atlas :** Custom
>
> **✅ Vérifié :** Aucune occurrence de `CloudClipboard`, `CloudClipboardAutomaticUpload` ou `IsCloudAndCrossDeviceOn` dans tout le source Atlas. **UNIQUE** — non couvert par Atlas ni Sophia.

Si vous utilisez un compte Microsoft connecté et que l'historique du presse-papier est activé, Windows peut synchroniser votre presse-papier avec le cloud Microsoft.

```powershell
If (-not (Test-Path "HKCU:\Software\Microsoft\Clipboard")) { New-Item -Path "HKCU:\Software\Microsoft\Clipboard" -Force | Out-Null }
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Clipboard" -Name "CloudClipboardAutomaticUpload" -Value 0 -Type DWord -Force
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Clipboard" -Name "IsCloudAndCrossDeviceOn" -Value 0 -Type DWord -Force
```

**Risque :** Aucun. L'historique local du presse-papier continue de fonctionner.

---

## 4 - NetworkThrottlingIndex — Désactiver le bridage réseau multimédia

> **Source Atlas :** Custom
>
> **✅ Vérifié :** Aucune occurrence de `NetworkThrottlingIndex` dans tout le source Atlas. **UNIQUE** — non couvert par Atlas ni Sophia.

Windows bride par défaut à **~10 Mbps** les paquets UDP/TCP non-MMCSS (comportement hérité de Vista). Sur une connexion gaming, cela peut affecter la latence des jeux en ligne. Mettre la valeur à `0xFFFFFFFF` désactive ce throttling.

```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile" -Name "NetworkThrottlingIndex" -Value 0xFFFFFFFF -Type DWord -Force
```

**Risque :** Aucun sur une connexion filaire. Inutile sur Wi-Fi bas débit.

---

## 5 - GlobalTimerResolutionRequests — Résolution du timer système (critique pour VR/anciens jeux)

> **Source Atlas :** Opt-in (`3. General Configuration/Timer Resolution/Enable timer resolution.cmd`)
>
> **✅ Vérifié :** Atlas propose cette clé en OPTION (non activée par défaut) via `Enable timer resolution.cmd`.
> De plus, Atlas va plus loin avec un scheduled task + `SetTimerResolution.exe` forçant 0.6ms.
> **CONDITIONNELLEMENT ADDITIF** — utile seulement si l'utilisateur n'a pas coché l'option Atlas.

Depuis Windows 11 22H2, la résolution du timer est devenue **per-process** (comportement "power-efficient"). Cela signifie que les anciens jeux ou SteamVR qui ne demandent pas explicitement 1ms se retrouvent avec 15.6ms par défaut, causant du micro-stuttering.

Cette clé restaure le comportement Windows 10 : si **n'importe quelle application** demande 1ms, le timer devient 1ms **pour tout le système**.

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\kernel" -Name "GlobalTimerResolutionRequests" -Value 1 -Type DWord -Force
```

> **Note :** Cette modification nécessite un redémarrage. Elle augmente légèrement la consommation électrique au repos (~1-3W sur laptop). Sur PC fixe de jeu, l'impact est négligeable.

**Risque :** Faible. Impact positif notable sur SteamVR, les jeux DX9/DX11 et tout runtime qui bénéficie d'une résolution haute du timer.

---

## 6 - DisablePagingExecutive + DisablePageCombining — RAM maximale pour les jeux

> **Source Atlas :** `performance/system/disable-paging.yml`
>
> **Différence vs Atlas :** Le fichier `disable-paging.yml` existe dans Atlas avec le même code,
> **mais il est commenté** dans `tweaks.yml` (ligne 63 : `# - !task: {path: ...}`).
> Atlas a volontairement désactivé ce tweak. Ce script le réactive pour un usage gaming dédié.
>
> **✅ Vérifié :** `tweaks.yml` ligne 63 confirme : `# - !task: {path: 'tweaks\performance\system\disable-paging.yml'}`.
> Le fichier `disable-paging.yml` contient bien `DisablePagingExecutive=1` + `DisablePageCombining=1`.
> **ADDITIF** — Atlas l'a volontairement commenté, ce tweak le réactive pour usage gaming dédié.

Deux clés complémentaires dans le Memory Management :

- `DisablePagingExecutive = 1` : empêche Windows de swapper le code kernel sur disque (micro-stutters)
- `DisablePageCombining = 1` : désactive la déduplication des pages mémoire (Windows 10+ peut combiner les pages en arrière-plan, ce qui consomme des cycles CPU)

```powershell
$MemMgmt = "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management"
Set-ItemProperty -Path $MemMgmt -Name "DisablePagingExecutive" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $MemMgmt -Name "DisablePageCombining" -Value 1 -Type DWord -Force
```

**Risque :** Aucun sur ≥16 Go de RAM. `DisablePageCombining` augmente légèrement l'empreinte mémoire (≈50-150 Mo) en contrepartie de cycles CPU récupérés.

---

## 7 - Edge Startup Boost + Background Mode disable

> **Source Atlas :** Custom
>
> **✅ Vérifié :** Aucune occurrence de `StartupBoostEnabled` ni `BackgroundModeEnabled` dans tout
> le source Atlas. Aucune politique Edge appliquée par Atlas. **UNIQUE** — non couvert par Atlas ni Sophia.

Empêche Microsoft Edge de se précharger au démarrage Windows et de rester en mémoire quand fermé. Edge peut consommer 150-300 Mo de RAM en arrière-plan si ces options sont actives.

```powershell
If (-not (Test-Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge")) { New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Force | Out-Null }
# Désactiver le préchargement d'Edge au démarrage Windows
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Name "StartupBoostEnabled" -Value 0 -Type DWord -Force
# Désactiver Edge en arrière-plan quand toutes les fenêtres sont fermées
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Name "BackgroundModeEnabled" -Value 0 -Type DWord -Force
```

**Risque :** Aucun. Edge se lance juste un peu moins vite au premier clic. Ces paramètres sont contrôlés via la politique machine, donc stables.

---

## 8 - Nagle Algorithm disable — TcpNoDelay + TcpAckFrequency (per-NIC)

> **✅ Vérifié :** Aucune occurrence de `TcpNoDelay`, `TcpAckFrequency` ni `Nagle` dans tout le source Atlas.
> Aucune occurrence dans Sophia.psm1. **UNIQUE** — non couvert par Atlas ni Sophia.

Windows active par défaut l'algorithme de Nagle sur chaque carte réseau : il **bufferise les petits paquets TCP** avant de les envoyer (optimise la bande passante au détriment de la latence). Combiné avec le TCP Delayed ACK (attend 200ms ou 2 paquets avant d'envoyer un ACK), cela peut ajouter **10-40ms de latence** sur les petits paquets fréquents typiques du gaming en ligne (input joueur, position updates).

```powershell
# Désactive Nagle + TCP Delayed ACK sur toutes les interfaces réseau physiques actives
Get-NetAdapter -Physical | Where-Object { $_.Status -eq 'Up' } | ForEach-Object {
    $ifPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$($_.InterfaceGuid)"
    Set-ItemProperty -Path $ifPath -Name "TcpNoDelay" -Value 1 -Type DWord -Force
    Set-ItemProperty -Path $ifPath -Name "TcpAckFrequency" -Value 1 -Type DWord -Force
}
```

- `TcpNoDelay=1` : désactive Nagle (chaque paquet est envoyé immédiatement sans attendre de remplir le buffer)
- `TcpAckFrequency=1` : envoie un ACK pour chaque paquet reçu (au lieu d'attendre 200ms ou 2 paquets)

**Risque :** Aucun. Légère augmentation du nombre de paquets envoyés (overhead négligeable sur une connexion moderne ≥100 Mbps). Bénéfice surtout mesurable en jeu compétitif (FPS, VR multijoueur).

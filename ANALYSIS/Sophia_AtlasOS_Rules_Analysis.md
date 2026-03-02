# Analyse des règles actives Sophia — Conflits avec AtlasOS

Fichier analysé : `src/Sophia_Script_for_Windows_11/Sophia.ps1`

> **Légende**
>
> - ✅ **Aucun conflit** — La règle est compatible ou complémentaire avec AtlasOS
> - ⚠️ **Redondant** — AtlasOS applique déjà le même réglage
> - ❌ **Conflit** — La règle entre en contradiction avec AtlasOS

**Objectif** : voir les règles en conflit, ou redondantes, dans le scénario ou Sophia est appliqué **APRÈS** AtlasOS. Également on maximise la **stabilité** du système.

---

## Règle 1 : `CreateRestorePoint`

| Élément              | Détail                                                                                                                                                                                                                                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**    | Active temporairement System Protection si désactivé, crée un point de restauration "Sophia Script for Windows 11", puis remet la protection comme avant. Écrit dans `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SystemRestore\SystemRestorePointCreationFrequency`                                       |
| **Action AtlasOS**   | Atlas garde System Restore **activé** par défaut (`DEFAULT.reg` → `state=1`), mais **supprime tous les points de restauration existants** pendant l'installation (`vssadmin delete shadows /all /quiet` dans `CLEANUP.ps1`). Atlas fournit un toggle dans `AtlasDesktop/3. General Configuration/System Restore/` |
| **Registre Sophia**  | `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SystemRestore\SystemRestorePointCreationFrequency` (temporaire)                                                                                                                                                                                               |
| **Registre AtlasOS** | `HKLM\SOFTWARE\Policies\Microsoft\Windows NT\SystemRestore\DisableSR` (toggle on/off)                                                                                                                                                                                                                             |

### Verdict : ⚠️ Redondant mais attention

Sophia crée un point de restauration avant d'appliquer les tweaks. Si c'est exécuté **après** l'installation d'Atlas, le point est créé normalement (Atlas laisse System Restore activé par défaut). **Pas de conflit technique direct**, mais le point créé existerait sur un système déjà modifié par Atlas — restaurer ce point ne restaurerait pas un Windows "stock".

**Recommandation** : ✅ Conserver — c'est une bonne pratique de sécurité, le point de restauration n'entre pas en conflit.

---

## Règle 2 : `DiagTrackService -Disable`

| Élément            | Détail                                                                                                                                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | 1) Arrête le service `DiagTrack` 2) Définit son démarrage sur `Disabled` 3) Bloque le trafic sortant via la règle firewall du groupe `DiagTrack`                                                                                                                    |
| **Action AtlasOS** | 1) `services.yml` : `DiagTrack` → startup type `4` (Disabled) 2) `disallow-data-collection.yml` : arrête le service, désactive l'AutoLogger `Diagtrack-Listener`, met `AllowTelemetry=0`, supprime les logs ETL 3) Ne configure **PAS** de règle firewall DiagTrack |

### Analyse détaillée des clés de registre

| Clé                           | Sophia                        | Atlas                     | Conflit ?      |
| ----------------------------- | ----------------------------- | ------------------------- | -------------- |
| Service DiagTrack startup     | `Disabled`                    | `startup: 4` (= Disabled) | Non, identique |
| Firewall rule DiagTrack       | Activée + Block               | Non configurée            | Non            |
| `AllowTelemetry`              | Non touché par cette fonction | `0` (toutes les ruches)   | Non            |
| AutoLogger Diagtrack-Listener | Non touché                    | `Start=0`                 | Non            |

### Verdict : ⚠️ Redondant (partiellement complémentaire)

Les deux désactivent le service. Sophia ajoute en plus le **blocage firewall** du trafic DiagTrack, ce qu'Atlas ne fait **pas**. C'est donc **complémentaire** sur ce point.

**Recommandation** : ✅ Conserver — la règle firewall est un plus par rapport à Atlas.

---

## Règle 3 : `ScheduledTasks -Disable`

| Élément            | Détail                                                                                                                                                                                                                                                                                                                       |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Affiche une boîte de dialogue WPF avec des tâches planifiées pré-cochées à désactiver : `MareBackup`, `Microsoft Compatibility Appraiser`, `Microsoft Compatibility Appraiser Exp`, `StartupAppTask`, `Proxy`, `Consolidator`, `UsbCeip`, `Microsoft-Windows-DiskDiagnosticDataCollector`, `MapsToastTask`, `MapsUpdateTask` |
| **Action AtlasOS** | `disable-scheduled-tasks.yml` désactive : `PcaPatchDbTask`, `UCPD velocity`, `Microsoft-Windows-DiskDiagnosticDataCollector`, `Consolidator`, `UsbCeip`, `UsageDataReporting`                                                                                                                                                |

### Comparaison tâche par tâche

| Tâche                                           | Sophia        | Atlas         | État           |
| ----------------------------------------------- | ------------- | ------------- | -------------- |
| `Consolidator`                                  | ✅ Désactive  | ✅ Désactive  | Redondant      |
| `UsbCeip`                                       | ✅ Désactive  | ✅ Désactive  | Redondant      |
| `Microsoft-Windows-DiskDiagnosticDataCollector` | ✅ Désactive  | ✅ Désactive  | Redondant      |
| `MareBackup`                                    | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `Microsoft Compatibility Appraiser`             | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `Microsoft Compatibility Appraiser Exp`         | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `StartupAppTask`                                | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `Proxy`                                         | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `MapsToastTask`                                 | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `MapsUpdateTask`                                | ✅ Désactive  | ❌ Non touché | Complémentaire |
| `PcaPatchDbTask`                                | ❌ Non touché | ✅ Désactive  | —              |
| `UCPD velocity`                                 | ❌ Non touché | ✅ Désactive  | —              |
| `UsageDataReporting`                            | ❌ Non touché | ✅ Désactive  | —              |

### Verdict : ⚠️ Partiellement redondant, majoritairement complémentaire

3 tâches se chevauchent (redondant sans conflit), mais Sophia désactive 7 tâches supplémentaires qu'Atlas ne touche pas. **Aucun conflit** — les deux scripts utilisent `Disable-ScheduledTask` sur les mêmes tâches, l'opération est idempotente.

**Recommandation** : ✅ Conserver — apporte des désactivations complémentaires utiles.

---

## Règle 4 : `SigninInfo -Disable`

| Élément            | Détail                                                                                                                                                                                                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | 1) Supprime la policy `DisableAutomaticRestartSignOn` de `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` 2) Crée `OptOut=1` dans `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\UserARSO\{SID}` |
| **Action AtlasOS** | **Aucune configuration trouvée** pour `UserARSO`, `ARSOUserConsent`, ou `DisableAutomaticRestartSignOn`                                                                                                                         |

### Analyse

Sophia désactive l'utilisation des informations de connexion pour terminer automatiquement la configuration après une mise à jour. Atlas ne touche **pas du tout** à ce paramètre.

### Verdict : ✅ Aucun conflit

**Recommandation** : ✅ Conserver — aucune interférence avec AtlasOS.

---

## Règle 5 : `ThisPC -Show`

| Élément            | Détail                                                                                                                                                                                                                |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Affiche l'icône "Ce PC" sur le Bureau en créant `{20D04FE0-3AEA-1069-A2D8-08002B30309D}=0` dans `HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\HideDesktopIcons\NewStartPanel`                             |
| **Action AtlasOS** | Atlas utilise `HideDesktopIcons\NewStartPanel` dans `disable-windows-spotlight.yml` mais pour un **CLSID différent** (pas le CLSID This PC). Atlas ne configure **pas** l'affichage de l'icône This PC sur le bureau. |

### Verdict : ✅ Aucun conflit

**Recommandation** : ✅ Conserver — Atlas n'affiche pas This PC sur le bureau, cette règle est indépendante.

---

## Règle 6 : `CheckBoxes -Disable` ✅ Commenté

| Élément            | Détail                                                                       |
| ------------------ | ---------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\AutoCheckSelect = 0`                            |
| **Action AtlasOS** | `disable-check-boxes.yml` : `HKCU\...\Explorer\Advanced\AutoCheckSelect = 0` |

### Verdict : ⚠️ Redondant → Commenté dans Sophia.ps1

Les deux appliquent exactement la même valeur au même emplacement du registre. Aucun conflit, opération idempotente.

**Action** : ✅ Commenté avec `# [AtlasOS REDUNDANT] Atlas disable-check-boxes.yml AutoCheckSelect=0`

---

## Règle 7 : `HiddenItems -Enable` ✅ Commenté

| Élément            | Détail                                                                  |
| ------------------ | ----------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\Hidden = 1` (afficher les fichiers cachés) |
| **Action AtlasOS** | `show-files.yml` : `HKCU\...\Explorer\Advanced\Hidden = 1`              |

### Verdict : ⚠️ Redondant → Commenté dans Sophia.ps1

Même valeur, même chemin de registre. Totalement identique.

**Action** : ✅ Commenté avec `# [AtlasOS REDUNDANT] Atlas show-files.yml Hidden=1` : `FileExtensions -Show` ✅ Commenté

| Élément            | Détail                                                                  |
| ------------------ | ----------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\HideFileExt = 0` (afficher les extensions) |
| **Action AtlasOS** | `show-files.yml` : `HKCU\...\Explorer\Advanced\HideFileExt = 0`         |

### Verdict : ⚠️ Redondant → Commenté dans Sophia.ps1

Identique. Même registre, même valeur.

**Action** : ✅ Commenté avec `# [AtlasOS REDUNDANT] Atlas show-files.yml HideFileExt=0`

---

## Règle 9 : `MergeConflicts -Show`

| Élément            | Détail                                                                                             |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\HideMergeConflicts = 0` (afficher les conflits de fusion de dossiers) |
| **Action AtlasOS** | **Aucune configuration trouvée** pour `HideMergeConflicts`                                         |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas cette valeur.

**Recommandation** : ✅ Conserver — aucune interférence avec AtlasOS.

---

## Règle 10 : `OpenFileExplorerTo -ThisPC` ✅ Commenté

| Élément            | Détail                                                                        |
| ------------------ | ----------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\LaunchTo = 1` (ouvrir l'explorateur sur "Ce PC") |
| **Action AtlasOS** | `open-to-this-pc.yml` : `HKCU\...\Explorer\Advanced\LaunchTo = 1`             |

### Verdict : ⚠️ Redondant → Commenté dans Sophia.ps1

Identique. Même registre, même valeur.

**Action** : ✅ Commenté avec `# [AtlasOS REDUNDANT] Atlas open-to-this-pc.yml LaunchTo=1`

---

## Résumé

| #   | Règle                        | Verdict                                    | Action recommandée |
| --- | ---------------------------- | ------------------------------------------ | ------------------ |
| 1   | `CreateRestorePoint`         | ✅ Compatible                              | Conserver          |
| 2   | `DiagTrackService -Disable`  | ⚠️ Redondant + complémentaire (firewall)   | Conserver          |
| 3   | `ScheduledTasks -Disable`    | ⚠️ Partiellement redondant, complémentaire | Conserver          |
| 4   | `SigninInfo -Disable`        | ✅ Aucun conflit                           | Conserver          |
| 5   | `ThisPC -Show`               | ✅ Aucun conflit                           | Conserver          |
| 6   | `CheckBoxes -Disable`        | ⚠️ Redondant                               | ✅ Commenté        |
| 7   | `HiddenItems -Enable`        | ⚠️ Redondant                               | ✅ Commenté        |
| 8   | `FileExtensions -Show`       | ⚠️ Redondant                               | ✅ Commenté        |
| 9   | `MergeConflicts -Show`       | ✅ Aucun conflit                           | Conserver          |
| 10  | `OpenFileExplorerTo -ThisPC` | ⚠️ Redondant                               | ✅ Commenté        |

### Bilan règles 1-10 : **Aucun conflit détecté — 4 redondances commentées**

- 4 règles sont **sans aucun lien avec AtlasOS** (1, 4, 5, 9)
- 5 règles sont **redondantes** avec des tweaks Atlas déjà appliqués (6, 7, 8, 10, et partiellement 2)
- 1 règle est **partiellement redondante mais complémentaire** (3)
- La règle 2 apporte un **plus** (blocage firewall DiagTrack)

> Les règles redondantes ne posent aucun problème technique (écritures idempotentes), mais peuvent être retirées pour un preset plus propre si souhaité.

---

---

# Règles 11 à 20

---

## Règle 11 : `FileExplorerCompactMode -Disable` ✅ Commenté (CONFLIT)

| Élément            | Détail                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\UseCompactMode = 0` (désactive le mode compact de l'explorateur) |
| **Action AtlasOS** | `use-compact-mode.yml` : `UseCompactMode = 1` (**active** le mode compact)                    |

### Analyse détaillée

Sophia met `UseCompactMode=0` (mode normal, par défaut Windows). Atlas met `UseCompactMode=1` (mode compact). Ce sont des valeurs **opposées** sur la même clé de registre.

Atlas applique le mode compact pour optimiser l'espace d'affichage. Sophia (dans ce preset) le désactive, revenant au comportement par défaut de Windows.

### Verdict : ❌ CONFLIT

Sophia écrase le réglage d'Atlas. Si Sophia s'exécute après Atlas : le mode compact sera **désactivé**, annulant le choix d'Atlas. Atlas fournit aussi un toggle dans `AtlasDesktop/4. Interface Tweaks/File Explorer Customization/Compact View/`.

**Recommandation** : Commenter cette règle si vous voulez garder le mode compact d'Atlas, **ou** la conserver si vous préférez le mode normal.

---

## Règle 12 : `SnapAssist -Disable`

| Élément            | Détail                                                                                                                                                     |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | 1) `HKCU:\Control Panel\Desktop\WindowArrangementActive = 1` 2) `HKCU:\...\Explorer\Advanced\SnapAssist = 0`                                               |
| **Action AtlasOS** | Atlas configure `EnableSnapAssistFlyout` et `EnableSnapBar` (via `Snap Layouts/Disable Snap Layouts.cmd`), mais ne touche **PAS** à la valeur `SnapAssist` |

### Analyse détaillée

Ce sont des registres **différents** :

- `SnapAssist` (Sophia) = contrôle si les suggestions de snap apparaissent à côté d'une fenêtre accrochée
- `EnableSnapAssistFlyout` (Atlas) = contrôle les layouts de snap affichés au survol du bouton maximiser
- `EnableSnapBar` (Atlas) = contrôle la barre de snap en haut de l'écran

Atlas ne touche pas la clé `SnapAssist` que Sophia modifie.

### Verdict : ✅ Aucun conflit

Les deux agissent sur des aspects différents du système de snap Windows. Complémentaires.

**Recommandation** : ✅ Conserver — aucune interférence.

---

## Règle 13 : `FileTransferDialog -Detailed` ✅ Commenté

| Élément            | Détail                                                                                             |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\OperationStatusManager\EnthusiastMode = 1` (mode détaillé pour les transferts) |
| **Action AtlasOS** | `always-more-details-transfer.yml` : `EnthusiastMode = 1`                                          |

### Verdict : ⚠️ Redondant → Commenté dans Sophia.ps1

Identique. Même registre, même valeur.

**Action** : ✅ Commenté avec `# [AtlasOS REDUNDANT] Atlas always-more-details-transfer.yml EnthusiastMode=1`

---

## Règle 14 : `RecycleBinDeleteConfirmation -Enable`

| Élément            | Détail                                                                                                                                                                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Action Sophia**  | 1) **Supprime** les policies `ConfirmFileDelete` de `HKLM:\...\Policies\Explorer` et `HKCU:\...\Policies\Explorer` 2) Modifie le bit 3 du champ binaire `ShellState` dans `HKCU:\...\Explorer` pour activer la confirmation de suppression |
| **Action AtlasOS** | **Aucune configuration trouvée** pour `ConfirmFileDelete` ou `ShellState`                                                                                                                                                                  |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas cette fonctionnalité.

**Recommandation** : ✅ Conserver — aucune interférence avec AtlasOS.

---

## Règle 15 : `QuickAccessRecentFiles -Hide` ✅ Commenté (CONFLIT)

| Élément            | Détail                                                                                                                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | 1) **Supprime** les policies `NoRecentDocsHistory` de `HKLM:\SOFTWARE\Policies\Microsoft\Windows\Explorer` et `HKCU:\...\Policies\Explorer` 2) Met `ShowRecent = 0` dans `HKCU:\...\Explorer` |
| **Action AtlasOS** | `hide-frequently-used-items.yml` : Met `ShowRecent = 0` **ET** `NoRecentDocsHistory = 1` dans `HKCU:\...\Policies\Explorer` **ET** `ClearRecentDocsOnExit = 1`                                |

### Analyse détaillée — ❌ CONFLIT PARTIEL

| Clé                                                 | Sophia       | Atlas            | Conflit ?      |
| --------------------------------------------------- | ------------ | ---------------- | -------------- |
| `HKCU:\...\Explorer\ShowRecent`                     | `0`          | `0`              | Non, identique |
| `HKCU:\...\Policies\Explorer\NoRecentDocsHistory`   | **SUPPRIME** | `1`              | **OUI**        |
| `HKLM:\SOFTWARE\Policies\...\NoRecentDocsHistory`   | **SUPPRIME** | `1` (via toggle) | **OUI**        |
| `HKCU:\...\Policies\Explorer\ClearRecentDocsOnExit` | Non touché   | `1`              | Non            |

Sophia supprime la policy `NoRecentDocsHistory` qu'Atlas a configurée à `1`. Même si le résultat visuel est le même (fichiers récents cachés via `ShowRecent=0`), **Sophia affaiblit la protection d'Atlas** en supprimant la policy de groupe qui empêche au niveau système le suivi des documents récents.

### Verdict : ❌ CONFLIT

Sophia supprime `NoRecentDocsHistory` qu'Atlas utilise comme protection supplémentaire au niveau policy.

**Recommandation** : Commenter cette règle — Atlas gère déjà `ShowRecent=0` + la policy `NoRecentDocsHistory`. Ou modifier la fonction pour ne pas supprimer la policy.

---

## Règle 16 : `QuickAccessFrequentFolders -Hide` ✅ Commenté

| Élément            | Détail                                                |
| ------------------ | ----------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\ShowFrequent = 0`                 |
| **Action AtlasOS** | `hide-frequently-used-items.yml` : `ShowFrequent = 0` |

### Verdict : ⚠️ Redondant → Commenté dans Sophia.ps1

Identique. La fonction Sophia pour `QuickAccessFrequentFolders` ne supprime aucune policy (contrairement à `QuickAccessRecentFiles`), donc pas de conflit.

**Action** : ✅ Commenté avec `# [AtlasOS REDUNDANT] Atlas hide-frequently-used-items.yml ShowFrequent=0`

---

## Règle 17 : `TaskbarCombine -Always`

| Élément            | Détail                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | 1) Supprime les policies `NoTaskGrouping` de HKLM/HKCU 2) Met `TaskbarGlomLevel = 0` (toujours combiner) |
| **Action AtlasOS** | **Aucune configuration trouvée** pour `TaskbarGlomLevel` ou `NoTaskGrouping`                             |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas le groupement des boutons de la barre des tâches.

**Recommandation** : ✅ Conserver — aucune interférence avec AtlasOS.

---

## Règle 18 : `UnpinTaskbarShortcuts -Shortcuts Edge, Store, Outlook` ✅ Commenté

| Élément            | Détail                                                                                                                                                                                                                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Décroche les raccourcis Edge, Microsoft Store et Outlook de la barre des tâches via COM Shell.Application (méthode "Unpin from taskbar")                                                                                                                                                                 |
| **Action AtlasOS** | `config-pins.yml` : Reconfigure complètement les épingles de la barre des tâches via `TASKBARPINS.ps1`. Dès l'installation, Atlas remet le navigateur choisi + Explorer. Les pins par défaut (Edge, Store, Outlook) sont **déjà remplacées** par Atlas. Supprime aussi `MailPin=0` et `CopilotPWAPin=0`. |

### Analyse

Si Atlas a déjà été installé, Edge/Store/Outlook ne devraient plus être épinglés. L'opération de Sophia sera un no-op (les raccourcis n'existent plus). Pas de conflit, mais inutile.

### Verdict : ⚠️ Redondant (no-op)

Sur un système Atlas, ces raccourcis n'existent plus.

**Recommandation** : ⚠️ Peut être retiré — mais le conserver ne cause aucun problème (opération silencieuse si les raccourcis n'existent pas).

---

## Règle 19 : `SecondsInSystemClock -Show`

| Élément            | Détail                                                     |
| ------------------ | ---------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\ShowSecondsInSystemClock = 1` |
| **Action AtlasOS** | **Aucune configuration trouvée**                           |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas l'affichage des secondes sur l'horloge.

**Recommandation** : ✅ Conserver — aucune interférence avec AtlasOS.

---

## Règle 20 : `ClockInNotificationCenter -Show`

| Élément            | Détail                                                          |
| ------------------ | --------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\ShowClockInNotificationCenter = 1` |
| **Action AtlasOS** | **Aucune configuration trouvée**                                |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas l'affichage de l'heure dans le centre de notifications.

**Recommandation** : ✅ Conserver — aucune interférence avec AtlasOS.

---

## Résumé règles 11-20

| #   | Règle                                  | Verdict              | Action recommandée    |
| --- | -------------------------------------- | -------------------- | --------------------- |
| 11  | `FileExplorerCompactMode -Disable`     | ❌ **CONFLIT**       | ✅ Commenté (CONFLIT) |
| 12  | `SnapAssist -Disable`                  | ✅ Aucun conflit     | Conserver             |
| 13  | `FileTransferDialog -Detailed`         | ⚠️ Redondant         | ✅ Commenté           |
| 14  | `RecycleBinDeleteConfirmation -Enable` | ✅ Aucun conflit     | Conserver             |
| 15  | `QuickAccessRecentFiles -Hide`         | ❌ **CONFLIT**       | ✅ Commenté (CONFLIT) |
| 16  | `QuickAccessFrequentFolders -Hide`     | ⚠️ Redondant         | ✅ Commenté           |
| 17  | `TaskbarCombine -Always`               | ✅ Aucun conflit     | Conserver             |
| 18  | `UnpinTaskbarShortcuts`                | ⚠️ Redondant (no-op) | ✅ Commenté           |
| 19  | `SecondsInSystemClock -Show`           | ✅ Aucun conflit     | Conserver             |
| 20  | `ClockInNotificationCenter -Show`      | ✅ Aucun conflit     | Conserver             |

### Bilan règles 11-20 : **2 conflits commentés — 3 redondances commentées**

- **Règle 11** (`FileExplorerCompactMode -Disable`) : ✅ Commenté — Écrase le mode compact activé par Atlas → valeurs opposées sur `UseCompactMode`
- **Règle 15** (`QuickAccessRecentFiles -Hide`) : ✅ Commenté — Supprime la policy `NoRecentDocsHistory` qu'Atlas utilise comme couche de protection supplémentaire
- 4 règles sont **indépendantes** d'Atlas (12, 14, 17, 19, 20)
- 3 règles sont **redondantes** sans conflit (13, 16, 18) — ✅ toutes commentées

---

---

# Analyse des règles 21-30

## Règle 21 : `ControlPanelView -LargeIcons`

| Élément            | Détail                                                                                                                                                                      |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\ControlPanel\AllItemsIconView = 0` + `StartupPage = 1` (mode grandes icônes). Supprime aussi la policy `ForceClassicControlPanel`                       |
| **Action AtlasOS** | `show-all-tasks-control-panel.yml` : Ajoute "All Tasks" (God Mode) dans le namespace du Panneau de configuration. **Ne modifie PAS** le mode d'affichage (icons/catégories) |

### Analyse détaillée

Atlas ajoute un raccourci "All Tasks" dans le Panneau de configuration pour un accès rapide à tous les paramètres — c'est une entrée dans le namespace CLSID, pas un changement de mode d'affichage. Sophia change le mode d'affichage du Panneau de configuration en "grandes icônes" via `AllItemsIconView` et `StartupPage`.

Les deux opérations sont **indépendantes** — elles n'opèrent pas sur les mêmes clés de registre.

### Verdict : ✅ Aucun conflit

Atlas ne modifie pas le mode d'affichage du Panneau de configuration.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 22 : `FirstLogonAnimation -Disable`

| Élément            | Détail                                                                                                                        |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKLM:\...\Windows NT\CurrentVersion\Winlogon\EnableFirstLogonAnimation = 0`. Supprime aussi la policy sous `Policies\System` |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                                              |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas l'animation de premier login.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 23 : `JPEGWallpapersQuality -Max` ✅ Commenté

| Élément            | Détail                                                                               |
| ------------------ | ------------------------------------------------------------------------------------ |
| **Action Sophia**  | `HKCU:\Control Panel\Desktop\JPEGImportQuality = 100`                                |
| **Action AtlasOS** | `best-wallpaper-quality.yml` : `HKCU:\Control Panel\Desktop\JPEGImportQuality = 100` |

### Analyse détaillée

| Clé                                             | Sophia | Atlas | Conflit ?           |
| ----------------------------------------------- | ------ | ----- | ------------------- |
| `HKCU:\Control Panel\Desktop\JPEGImportQuality` | `100`  | `100` | Non — **identique** |

Sophia et Atlas configurent exactement la même clé avec exactement la même valeur pour désactiver la compression JPEG des fonds d'écran.

### Verdict : ⚠️ Redondant (strictement identique)

**Recommandation** : Commenter — Atlas applique déjà `JPEGImportQuality=100` via `best-wallpaper-quality.yml`.

---

## Règle 24 : `ShortcutsSuffix -Disable` ✅ Commenté

| Élément            | Détail                                                                                                                                                                       |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Supprime `HKCU:\...\Explorer\link` (ancien comportement) + `HKCU:\...\Explorer\NamingTemplates\ShortcutNameTemplate = "%s.lnk"`                                              |
| **Action AtlasOS** | `remove-shortcut-text.yml` : `HKCU:\...\Explorer\NamingTemplates\ShortcutNameTemplate = "%s.lnk"`. Également dans `Disable Shortcut Text (default).cmd` (même valeur exacte) |

### Analyse détaillée

| Clé                                                       | Sophia     | Atlas      | Conflit ?           |
| --------------------------------------------------------- | ---------- | ---------- | ------------------- |
| `HKCU:\...\Explorer\NamingTemplates\ShortcutNameTemplate` | `"%s.lnk"` | `"%s.lnk"` | Non — **identique** |
| `HKCU:\...\Explorer\link`                                 | SUPPRIME   | Non touché | Non (nettoyage)     |

Sophia nettoie l'ancienne valeur `link` en plus de configurer `ShortcutNameTemplate`. Le résultat est identique : suppression du suffixe "- Shortcut". Le nettoyage supplémentaire de Sophia n'interfère pas avec Atlas.

### Verdict : ⚠️ Redondant (strictement identique sur la clé principale)

**Recommandation** : Commenter — Atlas applique déjà `ShortcutNameTemplate="%s.lnk"` via `remove-shortcut-text.yml`.

---

## Règle 25 : `PrtScnSnippingTool -Enable` ✅ Commenté (CONFLIT)

| Élément            | Détail                                                                                                      |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\Control Panel\Keyboard\PrintScreenKeyForSnippingEnabled = 1` (active le raccourci PrtScn → Snipping) |
| **Action AtlasOS** | `disable-screen-capture-hotkey.yml` : `PrintScreenKeyForSnippingEnabled = 0` (**désactive** le raccourci)   |

### Analyse détaillée

| Clé                                                             | Sophia | Atlas | Conflit ?        |
| --------------------------------------------------------------- | ------ | ----- | ---------------- |
| `HKCU:\Control Panel\Keyboard\PrintScreenKeyForSnippingEnabled` | `1`    | `0`   | **OUI — opposé** |

Atlas désactive ce raccourci car il supprime Snipping Tool par défaut — la touche PrtScn n'aurait rien à ouvrir. Sophia le réactive. Si Snipping Tool n'est pas réinstallé sur Atlas, l'activation du raccourci est sans effet. Si Snipping Tool est réinstallé, Sophia serait utile.

### Verdict : ❌ CONFLIT

Sophia écrase le réglage d'Atlas. Valeurs **opposées** sur `PrintScreenKeyForSnippingEnabled`.

**Recommandation** : Commenter si Snipping Tool n'est pas réinstallé. Conserver si Snipping Tool est présent sur le système.

---

## Règle 26 : `AppsLanguageSwitch -Disable`

| Élément            | Détail                                                                        |
| ------------------ | ----------------------------------------------------------------------------- |
| **Action Sophia**  | `Set-WinLanguageBarOption` (désactive le mode input hérité par fenêtre d'app) |
| **Action AtlasOS** | **Aucune configuration trouvée**                                              |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas la méthode de saisie par fenêtre.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 27 : `AeroShaking -Enable` ✅ Commenté (CONFLIT)

| Élément            | Détail                                                                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\DisallowShaking = 0` (active Aero Shake). Supprime les policies `NoWindowMinimizingShortcuts` en HKCU et HKLM  |
| **Action AtlasOS** | `disable-aero-shake.yml` : `HKCU:\...\Explorer\Advanced\DisallowShaking = 1` (**désactive** Aero Shake, souvent déclenché accidentellement) |

### Analyse détaillée

| Clé                                           | Sophia | Atlas | Conflit ?        |
| --------------------------------------------- | ------ | ----- | ---------------- |
| `HKCU:\...\Explorer\Advanced\DisallowShaking` | `0`    | `1`   | **OUI — opposé** |

Atlas désactive Aero Shake car il est souvent déclenché accidentellement lors de déplacements de fenêtres. Sophia l'active explicitement.

### Verdict : ❌ CONFLIT

Sophia écrase le réglage d'Atlas. Valeurs **opposées** sur `DisallowShaking`.

**Recommandation** : Commenter si l'on souhaite garder le comportement d'Atlas (pas d'Aero Shake accidentel). Conserver si l'on veut cette fonctionnalité.

---

## Règle 28 : `Cursors -Default` [CUSTOM]

| Élément            | Détail                                                                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Action Sophia**  | Remet les curseurs système par défaut (aero standard) dans `HKCU:\Control Panel\Cursors` — Arrow, Hand, Help, Wait, etc. vers `%SystemRoot%\cursors\aero_*.cur`          |
| **Action AtlasOS** | Les fichiers thème Atlas (`atlas-v0.4.x-dark.theme`, `atlas-v0.5.x-light.theme`) définissent les curseurs vers les mêmes fichiers `%windir%\cursors\aero_*.cur` standard |

### Analyse détaillée

Les deux systèmes pointent vers les mêmes fichiers curseur aero standard de Windows. Atlas les définit dans ses fichiers `.theme` ; Sophia les écrit directement dans le registre. Le résultat visuel est identique.

### Verdict : ✅ Aucun conflit

Les deux configurent les mêmes curseurs standard. Sur un système Atlas avec le thème par défaut, cette opération est un no-op.

**Recommandation** : Conserver — aucune interférence. Utile si les curseurs ont été modifiés manuellement auparavant.

---

## Règle 29 : `FolderGroupBy -None`

| Élément            | Détail                                                                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Action Sophia**  | Modifie `HKCU:\...\Explorer\FolderTypes\{885a186e-...}\TopViews\{000...}` pour désactiver le groupement des fichiers dans le dossier Téléchargements (`GroupBy = System.Null`) |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                                                                                               |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas le regroupement des fichiers dans le dossier Téléchargements.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 30 : `NavigationPaneExpand -Disable`

| Élément            | Détail                                                                                                    |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Explorer\Advanced\NavPaneExpandToCurrentFolder = 0` (ne pas développer le volet de navigation) |
| **Action AtlasOS** | **Aucune configuration trouvée** (Atlas configure `NetworkNavigationPane` mais pas l'expansion)           |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas l'expansion du volet de navigation de l'explorateur de fichiers.

**Recommandation** : Conserver — aucune interférence.

---

## Résumé règles 21-30

| #   | Règle                           | Verdict          | Action recommandée                        |
| --- | ------------------------------- | ---------------- | ----------------------------------------- |
| 21  | `ControlPanelView -LargeIcons`  | ✅ Aucun conflit | Conserver                                 |
| 22  | `FirstLogonAnimation -Disable`  | ✅ Aucun conflit | Conserver                                 |
| 23  | `JPEGWallpapersQuality -Max`    | ⚠️ Redondant     | ✅ Commenté                               |
| 24  | `ShortcutsSuffix -Disable`      | ⚠️ Redondant     | ✅ Commenté                               |
| 25  | `PrtScnSnippingTool -Enable`    | ❌ **CONFLIT**   | ✅ Commenté (CONFLIT)                     |
| 26  | `AppsLanguageSwitch -Disable`   | ✅ Aucun conflit | Conserver                                 |
| 27  | `AeroShaking -Enable`           | ❌ **CONFLIT**   | ✅ Commenté (CONFLIT)                     |
| 28  | `Cursors -Default`              | ✅ Aucun conflit | Conserver (mêmes curseurs aero standards) |
| 29  | `FolderGroupBy -None`           | ✅ Aucun conflit | Conserver                                 |
| 30  | `NavigationPaneExpand -Disable` | ✅ Aucun conflit | Conserver                                 |

### Bilan règles 21-30 : **2 conflits commentés — 2 redondances commentées**

- **Règle 25** (`PrtScnSnippingTool -Enable`) : ✅ Commenté — Atlas désactive `PrintScreenKeyForSnippingEnabled` car Snipping Tool est supprimé
- **Règle 27** (`AeroShaking -Enable`) : ✅ Commenté — Atlas désactive Aero Shake (`DisallowShaking=1`)
- **Règle 23** (`JPEGWallpapersQuality -Max`) : ✅ Commenté — Strictement identique à Atlas `best-wallpaper-quality.yml`
- **Règle 24** (`ShortcutsSuffix -Disable`) : ✅ Commenté — Strictement identique à Atlas `remove-shortcut-text.yml`
- 6 règles sont **indépendantes** d'Atlas (21, 22, 26, 28, 29, 30)

---

---

# Analyse des règles 31-40

## Règle 31 : `Win32LongPathsSupport -Enable` ✅ Commenté

| Élément            | Détail                                                                   |
| ------------------ | ------------------------------------------------------------------------ |
| **Action Sophia**  | `HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1` |
| **Action AtlasOS** | `enable-long-paths.yml` : `LongPathsEnabled = 1` (même clé, même valeur) |

### Analyse détaillée

| Clé                                                                  | Sophia | Atlas | Conflit ?           |
| -------------------------------------------------------------------- | ------ | ----- | ------------------- |
| `HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled` | `1`    | `1`   | Non — **identique** |

Sophia et Atlas activent tous deux le support des chemins longs avec exactement la même valeur.

### Verdict : ⚠️ Redondant (strictement identique)

**Recommandation** : Commenter — Atlas applique déjà `LongPathsEnabled=1` via `enable-long-paths.yml`.

---

## Règle 32 : `WindowsManageDefaultPrinter -Disable`

| Élément            | Détail                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKCU:\...\Windows NT\CurrentVersion\Windows\LegacyDefaultPrinterMode = 1`. Supprime la policy associée |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                        |

### Verdict : ✅ Aucun conflit

Atlas ne gère pas l'imprimante par défaut de Windows.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 33 : `WindowsFeatures -Disable`

| Élément            | Détail                                                                                                                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Boîte de dialogue interactive pour désactiver des fonctionnalités Windows. Pré-cochés : LegacyComponents, PowerShell 2.0, XPS, **Recall**, MediaPlayback                                      |
| **Action AtlasOS** | Atlas désactive Recall séparément via `disable-recall-snap.yml` (registre + service). Atlas a des toggles pour XPS/Printing dans les scripts avancés. Pas de suppression globale des features |

### Analyse détaillée

La fonctionnalité est **interactive** — l'utilisateur choisit quelles features désactiver via un dialogue. Les items pré-cochés sont alignés avec Atlas sur Recall (les deux le désactivent). Les mécanismes diffèrent : Sophia utilise `Disable-WindowsOptionalFeature`, Atlas utilise le registre/services.

### Verdict : ✅ Aucun conflit

Fonctionnalité interactive. Les choix par défaut sont compatibles avec Atlas.

**Recommandation** : Conserver — pas de risque de conflit (dialogue interactif).

---

## Règle 34 : `WindowsCapabilities -Install` [CUSTOM]

| Élément            | Détail                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| **Action Sophia**  | Boîte de dialogue interactive pour **installer** des capacités optionnelles Windows                                            |
| **Action AtlasOS** | Atlas supprime Steps Recorder via DISM pendant l'installation (`start.yml`). Atlas a aussi un toggle pour Windows Media Player |

### Analyse détaillée

Le mode `-Install` montre un dialogue pour installer des fonctionnalités manquantes. Théoriquement, l'utilisateur pourrait réinstaller Steps Recorder qu'Atlas a supprimé, mais c'est un choix explicite de l'utilisateur via le dialogue.

### Verdict : ✅ Aucun conflit

Fonctionnalité interactive. L'utilisateur fait un choix explicite.

**Recommandation** : Conserver — interaction utilisateur, pas de conflit automatique.

---

## Règle 35 : `UpdateMicrosoftProducts -Enable`

| Élément            | Détail                                                                                                     |
| ------------------ | ---------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKLM:\...\WindowsUpdate\UX\Settings\AllowMUUpdateService = 1`. Supprime la policy dans `WindowsUpdate\AU` |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                           |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas la mise à jour des produits Microsoft.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 36 : `RestartNotification -Show`

| Élément            | Détail                                                                                                                                                                        |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKLM:\...\WindowsUpdate\UX\Settings\RestartNotificationsAllowed2 = 1`. Supprime `SetAutoRestartNotificationDisable` policy                                                   |
| **Action AtlasOS** | Atlas garde les notifications de mise à jour activées par défaut (`UpdateNotifications` state=1 dans DEFAULT.reg). Toggle optionnel "Disable Update Notifications" disponible |

### Analyse détaillée

Par défaut, Atlas et Sophia sont alignés : les notifications de redémarrage sont activées. Sophia définit explicitement `RestartNotificationsAllowed2=1`, ce qui est cohérent avec le choix par défaut d'Atlas. Si l'utilisateur a désactivé les notifications via le toggle Atlas, Sophia les réactiverait — mais c'est le comportement attendu du preset.

### Verdict : ✅ Aucun conflit

Atlas laisse les notifications activées par défaut. Sophia fait de même.

**Recommandation** : Conserver — cohérent avec Atlas par défaut.

---

## Règle 37 : `RestartDeviceAfterUpdate -Disable` [CUSTOM]

| Élément            | Détail                                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Action Sophia**  | `HKLM:\...\WindowsUpdate\UX\Settings\IsExpedited = 0`. Supprime les policies `ActiveHoursEnd`, `ActiveHoursStart`, `SetActiveHours` dans `HKLM:\...\WindowsUpdate` |
| **Action AtlasOS** | Atlas configure `NoAutoRebootWithLoggedOnUsers = 1` dans `HKLM:\...\WindowsUpdate\AU` via `disable-auto-reboot.yml` (chemin **AU**, pas le même registre)          |

### Analyse détaillée

| Clé                                                  | Sophia         | Atlas      | Conflit ? |
| ---------------------------------------------------- | -------------- | ---------- | --------- |
| `...\WindowsUpdate\UX\Settings\IsExpedited`          | `0`            | Non touché | Non       |
| `...\WindowsUpdate\ActiveHoursEnd`                   | SUPPRIME       | Non touché | Non       |
| `...\WindowsUpdate\AU\NoAutoRebootWithLoggedOnUsers` | **Non touché** | `1`        | Non       |

Sophia et Atlas opèrent sur des sous-clés différentes (`WindowsUpdate` vs `WindowsUpdate\AU`). Sophia ne supprime PAS `NoAutoRebootWithLoggedOnUsers` qu'Atlas configure. Les deux mesures sont complémentaires : Atlas empêche le redémarrage automatique quand des utilisateurs sont connectés, Sophia désactive le redémarrage "accéléré".

### Verdict : ✅ Aucun conflit

Actions complémentaires sur des clés différentes.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 38 : `WindowsLatestUpdate -Disable`

| Élément            | Détail                                                                                                                                         |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | `HKLM:\...\WindowsUpdate\UX\Settings\IsContinuousInnovationOptedIn = 0`. Supprime `AllowOptionalContent` et `SetAllowOptionalContent` policies |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                                                               |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas cette option de mises à jour "early access".

**Recommandation** : Conserver — aucune interférence.

---

## Règle 39 : `NetworkAdaptersSavePower -Disable`

| Élément            | Détail                                                                                                                                                                                                                                         |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Désactive `AllowComputerToTurnOffDevice` sur les adaptateurs réseau. Écrit `ConsentStore\location\Value = Allow` (HKLM + HKCU) et supprime les policies `DisableLocation` et `LetAppsAccessLocation` comme précaution pour la reconnexion WiFi |
| **Action AtlasOS** | `Disable Location (default).cmd` : Désactive `lfsvc` service, `MapsBroker`, `AllowFindMyDevice=0`, `LocationSyncEnabled=0`, `ShowGlobalPrompts=0`. Location **désactivée** par défaut sur Atlas                                                |

### Analyse détaillée — ✅ AUCUN CONFLIT PRATIQUE

| Clé / Service                                     | Sophia     | Atlas          | Conflit ?                                                                                                     |
| ------------------------------------------------- | ---------- | -------------- | ------------------------------------------------------------------------------------------------------------- |
| `ConsentStore\location\Value` (HKLM + HKCU)       | `Allow`    | **Non touché** | Non — Atlas ne met PAS cette clé à "Deny". Valeur par défaut Windows = "Allow". Sophia réécrit la même valeur |
| `ConsentStore\location\NonPackaged\Value` (HKCU)  | `Allow`    | **Non touché** | Idem — no-op en pratique                                                                                      |
| `ConsentStore\location\ShowGlobalPrompts` (HKCU)  | Non touché | `0`            | Non — registres différents, pas de conflit                                                                    |
| `Policies\...\LocationAndSensors\DisableLocation` | SUPPRIME   | Non configuré  | Non (policy non utilisée par Atlas)                                                                           |
| `Policies\...\AppPrivacy\LetAppsAccessLocation`   | SUPPRIME   | Non configuré  | Non                                                                                                           |
| Service `lfsvc`                                   | Non touché | **Désactivé**  | Services toujours off — aucune fuite de localisation possible                                                 |

**Analyse approfondie du mécanisme WiFi (try/catch + netsh) :**

1. **Le bloc `catch` est du code mort** pour les erreurs de localisation. `Start-Process -ErrorAction Stop` ne throw que si le processus ne peut pas être _lancé_ (fichier introuvable, permission OS refusée). L'exit code de `netsh.exe` n'est jamais vérifié par `Start-Process` — même avec `-Wait`, le cmdlet considère l'opération réussie dès que le processus démarre. Le catch ne se déclencherait que si `netsh.exe` disparaissait de `System32` (impossible en pratique).

2. **`netsh wlan connect` n'a pas besoin de localisation.** La commande utilise l'API `WlanConnect()` (connexion à un profil connu par SSID). Selon [la doc Microsoft](https://learn.microsoft.com/en-us/windows/win32/nativewifi/wi-fi-access-location-changes), seules les APIs de _scan/BSSID_ sont restreintes (`WlanGetAvailableNetworkList`, `WlanGetNetworkBssList`, `WlanScan`, `WlanQueryInterface`). `WlanConnect()` **n'est pas dans la liste** des APIs affectées.

3. **Sur Atlas, la valeur ConsentStore est déjà "Allow"** par défaut Windows. Atlas désactive la localisation via les _services_ (`lfsvc`, `MapsBroker`) et les _policies_ (`FindMyDevice`), mais **ne modifie jamais** `ConsentStore\location\Value`. Sophia réécrit donc la même valeur — c'est un no-op.

4. **Les services `lfsvc` et `MapsBroker` restent désactivés** après exécution de Sophia. Même avec `ConsentStore = Allow`, aucune donnée de localisation ne peut être collectée sans le service actif.

### Verdict : ✅ Aucun conflit pratique

La fonction principale (gestion veille adaptateurs réseau) n'a aucun conflit avec Atlas. L'écriture de `ConsentStore\location = Allow` est un no-op en pratique (valeur déjà à "Allow" par défaut Windows, non modifiée par Atlas). Le bloc `try/catch` sur `netsh` est du code mort pour les erreurs de localisation mais n'a aucun impact sur le fonctionnement. Les services de localisation restent désactivés.

**Recommandation** : Conserver — la fonction est stable sur Atlas, aucune modification nécessaire.

---

## Règle 40 : `InputMethod -Default` [CUSTOM]

| Élément            | Détail                                                                                                    |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Supprime `InputMethodOverride` de `HKCU:\Control Panel\International\User Profile` (retour liste langues) |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                          |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas la méthode de saisie par défaut.

**Recommandation** : Conserver — aucune interférence.

---

## Résumé règles 31-40

| #   | Règle                                  | Verdict                   | Action recommandée                                                           |
| --- | -------------------------------------- | ------------------------- | ---------------------------------------------------------------------------- |
| 31  | `Win32LongPathsSupport -Enable`        | ⚠️ Redondant              | ✅ Commenté                                                                  |
| 32  | `WindowsManageDefaultPrinter -Disable` | ✅ Aucun conflit          | Conserver                                                                    |
| 33  | `WindowsFeatures -Disable`             | ✅ Aucun conflit          | Conserver (interactif)                                                       |
| 34  | `WindowsCapabilities -Install`         | ✅ Aucun conflit          | Conserver (interactif)                                                       |
| 35  | `UpdateMicrosoftProducts -Enable`      | ✅ Aucun conflit          | Conserver                                                                    |
| 36  | `RestartNotification -Show`            | ✅ Aucun conflit          | Conserver (aligné avec Atlas par défaut)                                     |
| 37  | `RestartDeviceAfterUpdate -Disable`    | ✅ Aucun conflit          | Conserver (complémentaire à Atlas)                                           |
| 38  | `WindowsLatestUpdate -Disable`         | ✅ Aucun conflit          | Conserver                                                                    |
| 39  | `NetworkAdaptersSavePower -Disable`    | ✅ Aucun conflit pratique | Conserver (ConsentStore réécrit à sa valeur existante, services restent off) |
| 40  | `InputMethod -Default`                 | ✅ Aucun conflit          | Conserver                                                                    |

### Bilan règles 31-40 : **1 redondance commentée — 0 side-effect — 0 conflit direct**

- **Règle 31** (`Win32LongPathsSupport -Enable`) : ✅ Commenté — Strictement identique à Atlas `enable-long-paths.yml`
- **Règle 39** (`NetworkAdaptersSavePower -Disable`) : ✅ Aucun conflit pratique — l'écriture ConsentStore est un no-op (valeur déjà "Allow" par défaut, non modifiée par Atlas), le catch est du code mort, les services restent off
- 9 règles sont **indépendantes** d'Atlas (32, 33, 34, 35, 36, 37, 38, 39, 40)

---

## Règle 41 : `Set-UserShellFolderLocation -Default` [CUSTOM]

| Élément            | Détail                                                                                                                                                                       |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Remet les dossiers utilisateur (Bureau, Documents, Téléchargements, Musique, Images, Vidéos) à leur emplacement par défaut (`%USERPROFILE%\<Folder>`) via un menu interactif |
| **Action AtlasOS** | **Aucune configuration trouvée** — Atlas ne modifie pas les emplacements des dossiers utilisateur                                                                            |

### Verdict : ✅ Aucun conflit

Atlas ne touche pas aux shell folders. La fonction est interactive (menu de confirmation pour chaque dossier).

**Recommandation** : Conserver — aucune interférence.

---

## Règle 42 : `WinPrtScrFolder -Default` [CUSTOM]

| Élément            | Détail                                                                                                                                          |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Supprime le redirect du dossier de screenshots (`{B7BEDE81-DF94-4682-A7D8-57A52620B86F}`) → les captures Win+PrtScr vont dans le dossier Images |
| **Action AtlasOS** | **Aucune configuration trouvée** — Atlas ne modifie pas l'emplacement des captures d'écran                                                      |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas le dossier de captures d'écran. Note : la fonction vérifie la présence de OneDrive et skip si connecté.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 43 : `RecommendedTroubleshooting -Default` [CUSTOM]

| Élément            | Détail                                                                                                                                                                                                                                          |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Configure les troubleshooters sur "demander avant d'exécuter". **Effets secondaires critiques** : supprime les policies `AllowTelemetry`, `MaxTelemetryAllowed`, active `WerSvc`, active `QueueReporting`, supprime les policies `WER Disabled` |
| **Action AtlasOS** | `disallow-data-collection.yml` : `AllowTelemetry=0`, `MaxTelemetryAllowed=0`. `services.yml` : `WerSvc` désactivé (startup 4), `DiagTrack` désactivé. `disable-win-error-reporting.yml` : `WER Disabled=1` partout                              |

### Analyse détaillée — ❌ CONFLIT CRITIQUE

Le code de `RecommendedTroubleshooting` exécute des actions **AVANT** le switch `-Default`/`-Automatically` — ces actions s'exécutent **systématiquement** quel que soit le paramètre :

| Action Sophia (pré-switch)                                 | Valeur Atlas écrasée                  | Impact                                            |
| ---------------------------------------------------------- | ------------------------------------- | ------------------------------------------------- |
| `Remove-ItemProperty ... AllowTelemetry`                   | Atlas: `AllowTelemetry=0`             | **Supprime** la restriction de télémétrie d'Atlas |
| `Remove-ItemProperty ... MaxTelemetryAllowed`              | Atlas: `MaxTelemetryAllowed=0`        | **Supprime** le plafond de télémétrie d'Atlas     |
| `Remove-ItemProperty ... ShowedToastAtLevel`               | Atlas: `ShowedToastAtLevel=1`         | Supprime le flag d'état DiagTrack                 |
| `Set-Policy ... AllowTelemetry -Type CLEAR`                | Atlas: policy `AllowTelemetry=0`      | **Efface** la GPO de télémétrie                   |
| `Enable-ScheduledTask QueueReporting`                      | Atlas: probablement désactivée        | Réactive la tâche de rapport d'erreurs            |
| `Remove-ItemProperty ... WER ... Disabled`                 | Atlas: `WER Disabled=1` (HKLM + HKCU) | **Réactive** Windows Error Reporting              |
| `Set-Policy ... WER Disabled -Type CLEAR`                  | Atlas: policies WER                   | **Efface** les GPO de WER                         |
| `Set-Service WerSvc -StartupType Manual` + `Start-Service` | Atlas: `WerSvc startup=4` (désactivé) | **Réactive et démarre** WerSvc                    |

**Cette fonction annule directement le durcissement vie privée d'Atlas** : télémétrie, error reporting, et DiagTrack policies.

### Verdict : ❌ CONFLIT DIRECT — Annule les protections vie privée d'Atlas

La fonction `RecommendedTroubleshooting -Default` est **incompatible avec Atlas** car elle réactive la télémétrie et Windows Error Reporting que Atlas a explicitement désactivés pour la confidentialité et les performances.

**Recommandation** : ✅ **Commenté** — cette fonction défait le durcissement vie privée d'Atlas. Le troubleshooter recommandé n'est pas nécessaire sur un système Atlas déjà optimisé.

---

## Règle 44 : `F1HelpPage -Disable`

| Élément            | Détail                                                                                                                       |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Désactive la recherche d'aide via F1 en redirigeant le TypeLib `{8cec5860-07a1-11d9-b15e-000d56bfe6ee}` vers une chaîne vide |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                                             |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas le comportement de la touche F1.

**Recommandation** : Conserver — aucune interférence. Amélioration QoL (évite l'ouverture intempestive de Bing Help).

---

## Règle 45 : `NumLock -Enable`

| Élément            | Détail                                                                                                      |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Active Num Lock au démarrage (`HKU\.DEFAULT\Control Panel\Keyboard\InitialKeyboardIndicators = 2147483650`) |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                            |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas Num Lock.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 46 : `StickyShift -Disable`

| Élément            | Détail                                                                                                                                                                    |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Désactive le raccourci Shift x5 pour les Sticky Keys (`HKCU:\Control Panel\Accessibility\StickyKeys\Flags = 506`)                                                         |
| **Action AtlasOS** | `disable-annoying-features-shortcuts.yml` : Désactive toutes les features d'accessibilité (`StickyKeys\Flags = 0`, + HighContrast, MouseKeys, ToggleKeys, FilterKeys = 0) |

### Analyse détaillée — ⚠️ REDONDANCE PARTIELLE

| Registre                                       | Sophia | Atlas | Observation                                        |
| ---------------------------------------------- | ------ | ----- | -------------------------------------------------- |
| `Control Panel\Accessibility\StickyKeys\Flags` | `506`  | `0`   | Sophia réécrit une valeur PLUS PERMISSIVE qu'Atlas |

**Détail des flags :**

- Atlas `Flags=0` : **TOUT** désactivé (aucune feature StickyKeys)
- Sophia `Flags=506` : raccourci Shift x5 désactivé, mais sons/indicateurs/confirmation restent activés
- Les deux désactivent le raccourci Shift x5 (le comportement gênant principal)
- Sophia **relâche** partiellement le verrouillage d'Atlas en réactivant certains flags secondaires (sons, indicateurs)

En pratique : impacts minimes car StickyKeys elle-même n'est pas activée (bit 0 = 0 dans les deux cas). Les flags secondaires ne s'appliquent que si StickyKeys est manuellement activée.

### Verdict : ⚠️ Redondance partielle — Approche moins agressive que Atlas

Sophia et Atlas font essentiellement la même chose (désactiver Shift x5) mais Sophia est moins agressive. Exécuter Sophia après Atlas changerait Flags de 0 à 506 — impact pratique quasi-nul.

**Recommandation** : ✅ **Commenté** — Atlas couvre déjà cette fonctionnalité de manière plus complète (Flags=0 désactive toutes les features d'accessibilité gênantes).

---

## Règle 47 : `ThumbnailCacheRemoval -Disable`

| Élément            | Détail                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| **Action Sophia**  | Désactive la suppression automatique du cache de miniatures (`VolumeCaches\Thumbnail Cache\Autorun = 0` en HKLM + WOW6432Node) |
| **Action AtlasOS** | `CLEANUP.ps1` : Configure `Thumbnail Cache` avec `StateFlags0064 = 2` (activé pour Disk Cleanup)                               |

### Analyse détaillée

Les deux paramètres sont **indépendants** :

- `Autorun` (Sophia) : contrôle la suppression **automatique** du cache à la connexion/déconnexion
- `StateFlags0064` (Atlas) : contrôle l'inclusion dans **Disk Cleanup** (cleanmgr) quand l'utilisateur le lance manuellement

Pas de conflit : Sophia empêche la suppression auto du cache (meilleure performance d'affichage des miniatures), Atlas inclut les thumbnails dans le nettoyage de disque manuel.

### Verdict : ✅ Aucun conflit

Les deux paramètres opèrent sur des mécanismes différents (auto vs. Disk Cleanup manuel).

**Recommandation** : Conserver — complémentaire avec Atlas. Améliore les performances d'affichage dans l'Explorateur.

---

## Règle 48 : `SaveRestartableApps -Disable` [CUSTOM]

| Élément            | Détail                                                                                                          |
| ------------------ | --------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Désactive la sauvegarde/restauration automatique des apps au redémarrage (`HKCU:\...\Winlogon\RestartApps = 0`) |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                                |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas la restauration des applications au redémarrage.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 49 : `RestorePreviousFolders -Disable`

| Élément            | Détail                                                                                                                |
| ------------------ | --------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Désactive la restauration des fenêtres de dossiers à la connexion (`HKCU:\...\Explorer\Advanced\PersistBrowsers = 0`) |
| **Action AtlasOS** | **Aucune configuration trouvée**                                                                                      |

### Verdict : ✅ Aucun conflit

Atlas ne configure pas la persistance des fenêtres Explorer.

**Recommandation** : Conserver — aucune interférence.

---

## Règle 50 : `NetworkDiscovery -Disable` [CUSTOM]

| Élément            | Détail                                                                                                                                              |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action Sophia**  | Désactive les règles de pare-feu "Network Discovery" et "File and Printer Sharing" pour les réseaux privés via `Set-NetFirewallRule -Enabled False` |
| **Action AtlasOS** | `Enable Network Discovery Services (default).cmd` : Active les services `fdPHost`, `FDResPub`, `lmhosts`, `SSDPSRV` par défaut (state=1)            |

### Analyse détaillée

Sophia et Atlas opèrent à des **niveaux différents** :

- **Atlas** : active les **services** de Network Discovery (couche système)
- **Sophia** : désactive les **règles de pare-feu** (couche réseau)

Le résultat : les services Network Discovery tournent (Atlas) mais ne peuvent pas communiquer (Sophia bloque via firewall). Ce n'est pas un conflit — c'est une couche de sécurité supplémentaire.

| Composant          | Atlas (défaut)   | Après Sophia | Résultat                                         |
| ------------------ | ---------------- | ------------ | ------------------------------------------------ |
| Services fdPHost   | Demand (start=3) | Inchangé     | Service disponible mais ne reçoit rien           |
| Services FDResPub  | Demand (start=3) | Inchangé     | Idem                                             |
| Firewall Discovery | Non configuré    | **Disabled** | Bloque la découverte réseau au niveau pare-feu   |
| Firewall SMB       | Non configuré    | **Disabled** | Bloque le partage de fichiers au niveau pare-feu |

### Verdict : ✅ Aucun conflit — Couche de sécurité complémentaire

Sophia ajoute un blocage pare-feu au-dessus des services Atlas. Si l'utilisateur veut activer Network Discovery via l'outil Atlas, il devra aussi réactiver les règles de pare-feu.

**Recommandation** : Conserver — renforce la sécurité par défaut. Compatible avec Atlas (les services ne sont pas désactivés, juste bloqués réseau).

---

## Résumé règles 41-50

| #   | Règle                                  | Verdict             | Action recommandée                                            |
| --- | -------------------------------------- | ------------------- | ------------------------------------------------------------- |
| 41  | `Set-UserShellFolderLocation -Default` | ✅ Aucun conflit    | Conserver (interactif)                                        |
| 42  | `WinPrtScrFolder -Default`             | ✅ Aucun conflit    | Conserver                                                     |
| 43  | `RecommendedTroubleshooting -Default`  | ❌ CONFLIT CRITIQUE | ✅ Commenté — réactive télémétrie + WER désactivés par Atlas  |
| 44  | `F1HelpPage -Disable`                  | ✅ Aucun conflit    | Conserver                                                     |
| 45  | `NumLock -Enable`                      | ✅ Aucun conflit    | Conserver                                                     |
| 46  | `StickyShift -Disable`                 | ⚠️ Redondant        | ✅ Commenté — Atlas Flags=0 couvre déjà (plus agressif)       |
| 47  | `ThumbnailCacheRemoval -Disable`       | ✅ Aucun conflit    | Conserver (complémentaire avec Atlas Disk Cleanup)            |
| 48  | `SaveRestartableApps -Disable`         | ✅ Aucun conflit    | Conserver                                                     |
| 49  | `RestorePreviousFolders -Disable`      | ✅ Aucun conflit    | Conserver                                                     |
| 50  | `NetworkDiscovery -Disable`            | ✅ Aucun conflit    | Conserver (couche sécurité complémentaire aux services Atlas) |

### Bilan règles 41-50 : **1 conflit commenté — 1 redondance commentée — 0 side-effect**

- **Règle 43** (`RecommendedTroubleshooting -Default`) : ✅ Commenté — ❌ conflit critique avec Atlas (réactive télémétrie + WER)
- **Règle 46** (`StickyShift -Disable`) : ✅ Commenté — redondant avec Atlas `disable-annoying-features-shortcuts.yml` (StickyKeys Flags=0)
- 8 règles sont **indépendantes** d'Atlas ou complémentaires (41, 42, 44, 45, 47, 48, 49, 50)

---

## Règles 51-60

### Règle 51 — `DefaultTerminalApp -WindowsTerminal`

**Sophia** : Définit Windows Terminal comme terminal par défaut via `HKCU:\Console\%%Startup` (DelegationConsole + DelegationTerminal = GUID de WT).

**Atlas** : Épingle Windows Terminal dans le Start Menu (`config-start-menu.yml`) mais **ne définit pas** WT comme terminal par défaut. Aucune clé `DelegationConsole`/`DelegationTerminal` n'est touchée.

**Verdict** : ✅ Aucun conflit — Complémentaire. Atlas rend WT visible, Sophia le met par défaut.

---

### Règle 52 — `PreventEdgeShortcutCreation -Channels Stable, Beta, Dev, Canary`

**Sophia** : Crée des politiques `HKLM:\SOFTWARE\Policies\Microsoft\EdgeUpdate\CreateDesktopShortcut{GUID}=0` pour chaque canal Edge (Stable, Beta, Dev, Canary). Empêche la recréation de raccourcis bureau lors des mises à jour Edge.

**Atlas** : Possède un script `RemoveEdge.ps1` pour supprimer Edge entièrement (approche différente), mais **aucune politique EdgeUpdate pour les raccourcis**. Si Edge est conservé sur Atlas, aucune protection contre la recréation de raccourcis.

**Verdict** : ✅ Aucun conflit — Utile si Edge est conservé. Inoffensif si Edge a été supprimé (contrôle de présence intégré : `Get-Package -Name "Microsoft Edge"`).

---

### Règle 53 — `WindowsAI -Disable`

**Sophia** :

1. Supprime les chemins de politique `HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsAI` et `HKCU:\...\WindowsAI`
2. Supprime la valeur `PolicyManager\default\WindowsAI\DisableAIDataAnalysis`
3. Désactive Recall via `Disable-WindowsOptionalFeature -Online -FeatureName Recall`
4. Supprime l'appx Copilot (`Microsoft.Copilot`)

**Atlas** :

- `disable-recall-snap.yml` → importe un `.reg` qui ajoute `DisableAIDataAnalysis=1` sous `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsAI`
- `disable-copilot.yml` → ajoute `TurnOffWindowsCopilot=1` sous `WindowsCopilot`
- `appx.yml` → supprime `Microsoft.Copilot*`

**Analyse croisée** :

- Sophia supprime la clé `WindowsAI` qui contient `DisableAIDataAnalysis=1` posée par Atlas — mais remplace cette protection par une désactivation **plus agressive** au niveau Windows Feature (`Disable-WindowsOptionalFeature`). La feature est complètement retirée, pas juste bloquée par politique.
- La politique `WindowsCopilot\TurnOffWindowsCopilot=1` d'Atlas est **préservée** (code déjà modifié lors analyse règle précédente — seuls les chemins `WindowsAI` sont supprimés, pas `WindowsCopilot`).
- La suppression de l'appx Copilot est **redondante** (déjà fait par Atlas `appx.yml`), mais inoffensive (`Get-AppxPackage` retourne simplement vide).

**Verdict** : ✅ Aucun conflit — Sophia va plus loin qu'Atlas (feature-level disable > policy disable). La suppression de la politique Atlas est compensée par un mécanisme plus robuste. Copilot appx déjà absente = no-op silencieux.

---

### Règle 54 — `Uninstall-UWPApps`

**Sophia** : Affiche un dialogue interactif (popup WPF) pour que l'utilisateur sélectionne les UWP apps à désinstaller.

**Atlas** : `appx.yml` supprime automatiquement ~30 apps (Teams, Copilot, Clipchamp, Disney, Spotify, Cortana, Mail, Paint 3D, Tips, Films & TV, Bing Weather/News, Get Help, Solitaire, Sticky Notes, OneNote, People, PowerAutomate, Skype, Todos, Alarms, Camera, Feedback Hub, Maps, Sound Recorder, etc.).

**Analyse croisée** : La fonction Sophia est **interactive** — les apps déjà supprimées par Atlas n'apparaissent tout simplement pas dans la liste. L'utilisateur ne peut sélectionner que ce qui reste.

**Verdict** : ✅ Aucun conflit — Mécanisme interactif, auto-adaptatif. Les apps Atlas-removed sont absentes de la popup.

---

### Règle 55 — `XboxGameBar -Enable`

**Sophia** :

- `HKCU:\Software\Microsoft\Windows\CurrentVersion\GameDVR\AppCaptureEnabled = 1`
- `HKCU:\System\GameConfigStore\GameDVR_Enabled = 1`

**Atlas** : État par défaut = **"Enable FSO and Game Bar Support (default).cmd"** (state=1) :

- Supprime `AppCaptureEnabled` (retour au défaut = activé)
- `GameDVR_Enabled = 1`
- `AllowGameDVR` policy supprimée

Atlas propose aussi un script optionnel "Disable FSO and Game Bar Support.cmd" qui désactive tout (AppCaptureEnabled=0, GameDVR_Enabled=0, AllowGameDVR=0, supprime xboxgamingoverlay, etc.).

**Analyse croisée** : Les deux sont cohérents — Atlas et Sophia activent Game Bar par défaut. Sophia set explicitement `AppCaptureEnabled=1` là où Atlas le supprime (retour au défaut implicite de 1). Résultat identique.

**Verdict** : ✅ Aucun conflit — Même résultat par des mécanismes légèrement différents.

---

### Règle 56 — `XboxGameTips -Disable`

**Sophia** : `HKCU:\Software\Microsoft\GameBar\ShowStartupPanel = 0`

**Atlas** : L'état par défaut d'Atlas ("Enable FSO and Game Bar" script) **supprime** `ShowStartupPanel` et `GamePanelStartupTipIndex` (retour aux défauts Windows = tips activés). Le script optionnel "Disable" pose `ShowStartupPanel=0` et `GamePanelStartupTipIndex=3`.

**Analyse croisée** : Puisque Atlas default ne touche pas aux tips (les laisse activés), Sophia ajoute une personnalisation QoL que l'état Atlas par défaut ne couvre pas.

**Verdict** : ✅ Aucun conflit — Customisation complémentaire, Atlas ne désactive pas les tips par défaut.

---

### Règle 57 — `GPUScheduling -Disable`

**Sophia** : `HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\HwSchMode = 1` (désactive le hardware-accelerated GPU scheduling).

**Atlas** : **Aucune configuration équivalente trouvée.** Atlas ne touche pas à `HwSchMode`.

**Verdict** : ✅ Aucun conflit — Réglage unique à Sophia. Note : ce paramètre remet la valeur par défaut de Windows.

---

### Règle 58 — `CleanupTask -Register`

**Sophia** : Crée une tâche planifiée `\Sophia\Windows Cleanup` qui nettoie les fichiers inutilisés et mises à jour Windows tous les 30 jours. Réactive les notifications pour afficher des toasts interactifs.

**Atlas** : Désactive les notifications **uniquement pendant le déploiement** (`DISABLENOTIFS.cmd`), puis les réactive à la fin (`enable-notifications.yml` → `ENABLENOTIFS.cmd`). Atlas possède ses propres mécanismes de nettoyage (Disk Cleanup Config) mais **pas de tâche planifiée équivalente**.

**Analyse croisée** : Les `Remove-ItemProperty` de la fonction Sophia (notifications) touchent les mêmes clés qu'Atlas réactive en fin de déploiement — pas de conflit puisque Atlas a déjà réactivé les notifications. La tâche planifiée de nettoyage est indépendante et complémentaire.

**Verdict** : ✅ Aucun conflit — Tâche de maintenance utile, notifications compatibles avec l'état post-Atlas.

---

### Règle 59 — `SoftwareDistributionTask -Register`

**Sophia** : Crée une tâche planifiée `\Sophia\SoftwareDistribution` qui nettoie `%SystemRoot%\SoftwareDistribution\Download` tous les 90 jours. Attend que le service Windows Update termine avant de purger.

**Atlas** : **Aucune tâche équivalente.** Atlas conserve Windows Update fonctionnel mais ne nettoie pas le dossier SoftwareDistribution automatiquement.

**Verdict** : ✅ Aucun conflit — Maintenance complémentaire. Libère de l'espace disque sans impact sur les mises à jour.

---

### Règle 60 — `NetworkProtection -Enable`

**Sophia** : `Set-MpPreference -EnableNetworkProtection Enabled` — Active la protection réseau de Microsoft Defender Exploit Guard. Vérifie `$Global:DefenderDefaultAV` avant exécution.

**Atlas** : Defender est **OPTIONNEL** lors de l'installation Atlas (`components.yml` : `option: 'defender-disable'` vs `option: 'defender-enable'`). Le choix est fait par l'utilisateur dans le playbook AME Wizard.

**Analyse croisée** :

- **Si Defender est conservé** (option `defender-enable`) : La fonction marche correctement et ajoute une couche de sécurité réseau précieuse. ✅
- **Si Defender est retiré** (option `defender-disable`) : `$Global:DefenderDefaultAV` sera `$false` (la requête `AntiVirusProduct` dans `SecurityCenter2` ne trouvera pas Defender). La fonction **skip automatiquement** avec un message "ThirdPartyAVInstalled" (message techniquement inexact mais fonctionnellement correct).

**Verdict** : ✅ Aucun conflit — **Conditionnel mais sûr.** Skip automatique si Defender absent. Message d'avertissement légèrement trompeur (dit "AV tiers installé" alors que Defender est simplement retiré). Même analyse applicable aux règles suivantes `PUAppsDetection -Enable` et `DefenderSandbox -Enable`.

---

| Règle | Fonction                                                          | Verdict          | Action                                                            |
| ----- | ----------------------------------------------------------------- | ---------------- | ----------------------------------------------------------------- |
| 51    | `DefaultTerminalApp -WindowsTerminal`                             | ✅ Aucun conflit | Conserver (complémentaire — Atlas épingle, Sophia met par défaut) |
| 52    | `PreventEdgeShortcutCreation -Channels Stable, Beta, Dev, Canary` | ✅ Aucun conflit | Conserver (utile si Edge présent, inoffensif sinon)               |
| 53    | `WindowsAI -Disable`                                              | ✅ Aucun conflit | Conserver (Sophia > Atlas : feature-level disable > policy)       |
| 54    | `Uninstall-UWPApps`                                               | ✅ Aucun conflit | Conserver (interactif, apps Atlas-removed absentes de la popup)   |
| 55    | `XboxGameBar -Enable`                                             | ✅ Aucun conflit | Conserver (cohérent avec Atlas default state=1)                   |
| 56    | `XboxGameTips -Disable`                                           | ✅ Aucun conflit | Conserver (QoL complémentaire, Atlas ne désactive pas les tips)   |
| 57    | `GPUScheduling -Disable`                                          | ✅ Aucun conflit | Conserver (réglage unique à Sophia, valeur défaut Windows)        |
| 58    | `CleanupTask -Register`                                           | ✅ Aucun conflit | Conserver (maintenance utile, notifications compatibles)          |
| 59    | `SoftwareDistributionTask -Register`                              | ✅ Aucun conflit | Conserver (maintenance complémentaire, aucun équivalent Atlas)    |
| 60    | `NetworkProtection -Enable`                                       | ✅ Conditionnel  | Conserver (skip auto si Defender retiré par Atlas)                |

### Bilan règles 51-60 : **0 conflit — 0 redondance à commenter — 0 side-effect**

- **Aucune règle ne nécessite de commentaire** dans cette tranche.
- Règle 53 (`WindowsAI -Disable`) supprime une politique Atlas mais la remplace par un mécanisme plus robuste (feature-level). Déjà corrigé pour préserver les clés `WindowsCopilot`.
- Règle 60 (`NetworkProtection -Enable`) est **conditionnelle** : fonctionne si Defender présent, skip silencieux sinon. Même comportement pour `PUAppsDetection` et `DefenderSandbox` (règles suivantes).
- 10 règles sont **compatibles** avec Atlas ou complémentaires.

---

## Règles 61-70

### Règle 61 — `PUAppsDetection -Enable`

**Sophia** : `Set-MpPreference -PUAProtection Enabled` — Active la détection des applications potentiellement indésirables (PUA) par Defender. Vérifie `$Global:DefenderDefaultAV` avant exécution.

**Atlas** : Defender est **optionnel** (`components.yml` : `defender-disable` / `defender-enable`). Pas de configuration spécifique PUA dans le playbook.

**Analyse** : Même schéma que la Règle 60 (NetworkProtection). Si Defender est présent, la fonction ajoute une protection utile. Si Defender est retiré, `$Global:DefenderDefaultAV = $false` → skip automatique.

**Verdict** : ✅ Conditionnel mais sûr — Skip automatique si Defender absent.

---

### Règle 62 — `DefenderSandbox -Enable`

**Sophia** : `setx /M MP_FORCE_USE_SANDBOX 1` — Active le sandboxing pour le processus Defender (MsMpEngCP.exe tourne dans un conteneur AppContainer). Vérifie `$Global:DefenderDefaultAV`.

**Atlas** : Pas de configuration spécifique de sandboxing Defender.

**Analyse** : Même schéma conditionnel. Si Defender est présent, sandboxing = bonne pratique de sécurité. Si absent, skip automatique.

**Verdict** : ✅ Conditionnel mais sûr — Skip automatique si Defender absent.

---

### Règle 63 — `EventViewerCustomView -Disable`

**Sophia** : Supprime la vue personnalisée "Process Creation" de l'Event Viewer, retire la politique `ProcessCreationIncludeCmdLine_Enabled`, et supprime le fichier XML associé.

**Atlas** : Aucune configuration de l'Event Viewer. Atlas ne crée ni ne supprime de vues personnalisées.

**Verdict** : ✅ Aucun conflit — Réglage unique à Sophia. Retour au défaut Windows (pas de vue Process Creation).

---

### Règle 64 — `PowerShellModulesLogging -Disable`

**Sophia** : Supprime `HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging\EnableModuleLogging` et le wildcard `*` dans `ModuleNames`. Retour au défaut (pas de logging modules PS).

**Atlas** : Aucune configuration de logging PowerShell trouvée dans le playbook.

**Verdict** : ✅ Aucun conflit — Réglage unique à Sophia. Valeur par défaut Windows.

---

### Règle 65 — `PowerShellScriptsLogging -Disable`

**Sophia** : Supprime `HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging\EnableScriptBlockLogging`. Retour au défaut (pas de script block logging).

**Atlas** : Aucune configuration de script block logging trouvée.

**Verdict** : ✅ Aucun conflit — Réglage unique à Sophia. Valeur par défaut Windows.

---

### Règle 66 — `AppsSmartScreen -Disable`

**Sophia** : `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\SmartScreenEnabled = "Off"`. Vérifie `$Global:DefenderDefaultAV`.

**Atlas** : SmartScreen est configuré **uniquement dans le Defender Remover** (`sxsc/Atlas-Defender-Remover.yaml`) :

- `EnableSmartScreen = 0` (politique HKLM)
- `SmartScreenEnabled = "Off"` (même clé Explorer que Sophia)
- SmartScreen pour Edge désactivé

Le Defender Remover n'est appliqué que si l'utilisateur choisit `defender-disable`.

**Analyse croisée** :

- **Si Defender conservé** : Atlas ne touche pas SmartScreen → `$Global:DefenderDefaultAV = $true` → Sophia désactive SmartScreen. Fonctionne.
- **Si Defender retiré** : Atlas Defender Remover a déjà mis `SmartScreenEnabled = "Off"` → `$Global:DefenderDefaultAV = $false` → Sophia **skip** → pas de conflit (SmartScreen déjà off via Atlas).

**Verdict** : ✅ Conditionnel mais sûr — Les deux scénarios sont cohérents.

---

### Règle 67 — `SaveZoneInformation -Disable`

**Sophia** :

1. Supprime `HKLM:\...\Policies\Attachments\SaveZoneInformation` (politique machine)
2. Crée `HKCU:\...\Policies\Attachments\SaveZoneInformation = 1` (politique utilisateur)
3. Désactive le marquage des fichiers téléchargés comme dangereux (pas de Zone.Identifier ADS)

**Atlas** : `SaveZoneInformation = 1` est posé dans `Atlas-Defender-Remover.yaml` **uniquement** au scope HKLM, et seulement si `defender-disable` est choisi.

**Analyse croisée** :

- **Si Defender conservé** : Atlas ne touche pas SaveZoneInformation → Sophia pose la valeur en HKCU → fonctionne.
- **Si Defender retiré** : Atlas pose `HKLM\...\SaveZoneInformation = 1`. Sophia supprime la valeur HKLM et pose la HKCU. Le résultat final est identique (zone info désactivée), juste au scope utilisateur au lieu de machine. Fonctionnellement pas de différence pour l'utilisateur courant.

**Note** : La fonction **ne vérifie pas** `$Global:DefenderDefaultAV` — elle s'exécute toujours. C'est logique car SaveZoneInformation est une fonctionnalité Windows (Attachment Manager), pas une fonctionnalité Defender.

**Verdict** : ✅ Aucun conflit — Même résultat dans les deux scénarios Atlas.

---

### Règle 68 — `WindowsSandbox -Enable`

**Sophia** : Active Windows Sandbox (`Containers-DisposableClientVM`) via `Enable-WindowsOptionalFeature`. Vérifie l'édition Windows (Pro/Enterprise/Education) et la virtualisation matérielle/Hyper-V.

**Atlas** : Aucune configuration de Windows Sandbox. Atlas configure Hyper-V uniquement dans le contexte des mitigations de sécurité (scripts dans `7. Security/Mitigations/`), pas pour activer/désactiver Sandbox.

**Verdict** : ✅ Aucun conflit — Feature indépendante, utile pour l'isolation. Requirent Pro ou supérieur.

---

### Règle 69 — `DNSoverHTTPS -Quad9`

**Sophia** : Configure DNS-over-HTTPS avec Quad9 (9.9.9.9 / 149.112.112.112) comme serveur DNS pour chaque interface réseau active. Utilise les APIs Windows natives `Set-DnsClientServerAddress` et `Set-DnsClientDohServerAddress`.

**Atlas** : **Aucune configuration DNS-over-HTTPS** dans le playbook. Atlas ne modifie pas les paramètres DNS.

**Verdict** : ✅ Aucun conflit — Réglage réseau unique à Sophia. Amélioration sécurité/privacy significative.

---

### Règle 70 — `MSIExtractContext -Show`

**Sophia** : Ajoute l'option "Extract all" dans le menu contextuel des fichiers `.msi` (Windows Installer) via `HKCR:\Msi.Package\shell\Extract\Command`.

**Atlas** : Atlas supprime le menu contextuel "Extract" pour les **fichiers ZIP** (via shell extensions dans `remove-context-menus\extract.yml` — GUID `{b8cdcb65-b1bf-4b42-9428-1dfdb7ee92af}`, etc.). C'est un scope complètement différent :

- Atlas : retire l'extraction ZIP du menu contextuel
- Sophia : ajoute l'extraction MSI au menu contextuel

**Verdict** : ✅ Aucun conflit — Scope différent (MSI vs ZIP). Les deux modifications coexistent sans interférence.

---

| Règle | Fonction                            | Verdict          | Action                                                        |
| ----- | ----------------------------------- | ---------------- | ------------------------------------------------------------- |
| 61    | `PUAppsDetection -Enable`           | ✅ Conditionnel  | Conserver (skip auto si Defender retiré)                      |
| 62    | `DefenderSandbox -Enable`           | ✅ Conditionnel  | Conserver (skip auto si Defender retiré)                      |
| 63    | `EventViewerCustomView -Disable`    | ✅ Aucun conflit | Conserver (retour défaut Windows, unique à Sophia)            |
| 64    | `PowerShellModulesLogging -Disable` | ✅ Aucun conflit | Conserver (retour défaut Windows, unique à Sophia)            |
| 65    | `PowerShellScriptsLogging -Disable` | ✅ Aucun conflit | Conserver (retour défaut Windows, unique à Sophia)            |
| 66    | `AppsSmartScreen -Disable`          | ✅ Conditionnel  | Conserver (cohérent avec les 2 scénarios Atlas Defender)      |
| 67    | `SaveZoneInformation -Disable`      | ✅ Aucun conflit | Conserver (même résultat que Atlas Defender Remover si actif) |
| 68    | `WindowsSandbox -Enable`            | ✅ Aucun conflit | Conserver (feature indépendante, Pro uniquement)              |
| 69    | `DNSoverHTTPS -Quad9`               | ✅ Aucun conflit | Conserver (unique à Sophia, amélioration privacy)             |
| 70    | `MSIExtractContext -Show`           | ✅ Aucun conflit | Conserver (scope MSI, distinct du ZIP context d'Atlas)        |

### Bilan règles 61-70 : **0 conflit — 0 redondance à commenter — 0 side-effect**

- 4 règles sont **conditionnelles** Defender (61, 62, 66) : sûres grâce au check `$Global:DefenderDefaultAV`
- 6 règles sont **totalement indépendantes** d'Atlas (63, 64, 65, 67, 68, 69, 70)
- Aucune modification de `Sophia.ps1` nécessaire

---

## Règles 71-80 (Context Menu — dernières règles)

### Règle 71 — `CABInstallContext -Show`

**Sophia** : Ajoute l'option "Install" dans le menu contextuel des fichiers `.cab` via `HKCR:\CABFolder\Shell\RunAs\Command`.

**Atlas** : Aucune configuration de menus contextuels CAB.

**Verdict** : ✅ Aucun conflit — Réglage unique à Sophia.

---

### Règle 72 — `EditWithClipchampContext -Hide`

**Sophia** : Masque "Edit with Clipchamp" du menu contextuel des fichiers média via des clés Shell Extensions.

**Atlas** : `appx.yml` supprime l'appx `Clipchamp.Clipchamp*` entièrement. Si Clipchamp est absent, l'entrée de menu contextuel n'existe plus de toute façon.

**Analyse** : Si Atlas a déjà supprimé Clipchamp, la commande Sophia est un no-op silencieux (l'entrée n'existe pas). Si Clipchamp est réinstallé, Sophia protège contre le retour de l'entrée de menu.

**Verdict** : ✅ Aucun conflit — Ceinture et bretelles. Inoffensif même si Clipchamp absent.

---

### Règle 73 — `EditWithPhotosContext -Hide`

**Sophia** : Masque "Edit with Photos" du menu contextuel des fichiers média.

**Atlas** : Ne touche pas aux entrées contextuelles de l'app Photos. L'app Photos est conservée par Atlas.

**Verdict** : ✅ Aucun conflit — Réglage QoL unique à Sophia.

---

### Règle 74 — `EditWithPaintContext -Hide`

**Sophia** : Masque "Edit with Paint" du menu contextuel des fichiers média.

**Atlas** : Supprime l'appx `Microsoft.MSPaint*` (Paint 3D), mais conserve Paint classique (`mspaint.exe`). Le menu "Edit with Paint" peut persister.

**Verdict** : ✅ Aucun conflit — Réglage QoL indépendant.

---

### Règle 75 — `PrintCMDContext -Hide`

**Sophia** : Masque l'option "Print" des fichiers `.bat` et `.cmd` via suppression de clés shell dans `HKCR:\batfile\shell\print` et `HKCR:\cmdfile\shell\print`.

**Atlas** : Atlas supprime le contexte "Printing" de manière générale (`remove-context-menus\printing.yml` → appelle `Disable Printing.cmd /justcontext`), mais cela désactive le **service d'impression** et les items contextuels liés aux imprimantes. Scope différent.

**Analyse** : Atlas retire l'impression au niveau service, Sophia retire spécifiquement l'entrée "Print" pour .bat/.cmd. Les deux sont complémentaires — le résultat final est le même (pas de "Print" pour .bat/.cmd), mais les mécanismes sont indépendants.

**Verdict** : ✅ Aucun conflit — Si Atlas a déjà désactivé l'impression, Sophia nettoie une entrée résiduelle. Sinon, Sophia est le seul à la retirer.

---

### Règle 76 — `CompressedFolderNewContext -Hide`

**Sophia** : Masque "Compressed (zipped) Folder" du menu "New" via `HKCR:\.zip\CompressedFolder\ShellNew`.

**Atlas** : Atlas supprime le menu "Extract" pour ZIP (`remove-context-menus\extract.yml`) via shell extensions blocked GUID `{b8cdcb65-b1bf-4b42-9428-1dfdb7ee92af}`, mais ne touche pas au "New > Compressed Folder".

**Verdict** : ✅ Aucun conflit — Scope différent (New menu vs Extract context). Complémentaire.

---

### Règle 77 — `MultipleInvokeContext -Enable`

**Sophia** : Active les items "Open", "Print", "Edit" dans le menu contextuel quand plus de 15 fichiers sont sélectionnés, via `HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\MultipleInvokePromptMinimum = 300`.

**Atlas** : Aucune configuration de seuil de sélection multiple.

**Verdict** : ✅ Aucun conflit — Réglage QoL unique à Sophia.

---

### Règle 78 — `UseStoreOpenWith -Hide`

**Sophia** : Masque "Look for an app in the Microsoft Store" dans le dialogue "Open with" via `HKLM:\SOFTWARE\Policies\Microsoft\Windows\Explorer\NoUseStoreOpenWith = 1`.

**Atlas** : Pas de configuration spécifique de ce dialogue, mais Atlas est orienté vers la réduction de l'intégration Store. Complémentaire.

**Verdict** : ✅ Aucun conflit — Réglage privacy/QoL indépendant.

---

### Règle 79 — `OpenWindowsTerminalContext -Show`

**Sophia** : Affiche "Open in Windows Terminal" dans le menu contextuel des dossiers en supprimant le blocage shell extension GUID `{9F156763-7844-4DC4-B2B1-901F640F5155}`.

**Atlas** : Atlas a son propre système de menus "Terminals" (`AtlasTerminals` shell key) avec un menu cascading (CMD, PowerShell, WT). L'état par défaut d'Atlas = **"Remove Terminals Context Menu (default).cmd"** — c'est-à-dire que les terminaux Atlas custom sont retirés par défaut.

**Analyse** : Les deux mécanismes sont **indépendants** :

- Sophia : utilise l'intégration native Windows 11 "Open in Terminal" (GUID shell extension)
- Atlas : crée une entrée custom `AtlasTerminals` avec sous-menu (mécanisme HKCR séparé)
  Aucune interférence — les deux peuvent coexister ou être utilisés séparément.

**Verdict** : ✅ Aucun conflit — Mécanismes distincts. Sophia active l'intégration native W11, Atlas a son propre système.

---

### Règle 80 — `OpenWindowsTerminalAdminContext -Enable`

**Sophia** : Modifie le fichier `settings.json` de Windows Terminal pour ajouter `"elevate": true` dans `profiles.defaults`, et s'assure que le shell extension n'est pas bloqué.

**Atlas** : Ne modifie pas les settings de Windows Terminal.

**Verdict** : ✅ Aucun conflit — Configuration de l'application WT, totalement indépendante.

---

| Règle | Fonction                                  | Verdict          | Action                                                        |
| ----- | ----------------------------------------- | ---------------- | ------------------------------------------------------------- |
| 71    | `CABInstallContext -Show`                 | ✅ Aucun conflit | Conserver (unique à Sophia)                                   |
| 72    | `EditWithClipchampContext -Hide`          | ✅ Aucun conflit | Conserver (no-op si Clipchamp retiré par Atlas)               |
| 73    | `EditWithPhotosContext -Hide`             | ✅ Aucun conflit | Conserver (QoL, Photos conservé par Atlas)                    |
| 74    | `EditWithPaintContext -Hide`              | ✅ Aucun conflit | Conserver (QoL indépendant)                                   |
| 75    | `PrintCMDContext -Hide`                   | ✅ Aucun conflit | Conserver (complémentaire avec Atlas printing.yml)            |
| 76    | `CompressedFolderNewContext -Hide`        | ✅ Aucun conflit | Conserver (scope New menu, distinct du Extract d'Atlas)       |
| 77    | `MultipleInvokeContext -Enable`           | ✅ Aucun conflit | Conserver (unique à Sophia)                                   |
| 78    | `UseStoreOpenWith -Hide`                  | ✅ Aucun conflit | Conserver (privacy/QoL indépendant)                           |
| 79    | `OpenWindowsTerminalContext -Show`        | ✅ Aucun conflit | Conserver (intégration native W11, distinct d'AtlasTerminals) |
| 80    | `OpenWindowsTerminalAdminContext -Enable` | ✅ Aucun conflit | Conserver (config WT settings.json, unique à Sophia)          |

### Bilan règles 71-80 : **0 conflit — 0 redondance — 0 side-effect**

- 10 règles sont **totalement indépendantes** d'Atlas ou complémentaires
- Aucune modification de `Sophia.ps1` nécessaire

---

## Bilan global — Analyse complète (80 règles actives)

### Statistiques

| Tranche      | Conflits | Redondances | Conditionnels | Sûrs   | Commentés |
| ------------ | -------- | ----------- | ------------- | ------ | --------- |
| Règles 1-10  | 0        | 4           | 0             | 6      | 4         |
| Règles 11-20 | 2        | 3           | 0             | 5      | 5         |
| Règles 21-30 | 2        | 2           | 0             | 6      | 4         |
| Règles 31-40 | 0        | 1           | 0             | 9      | 1         |
| Règles 41-50 | 1        | 1           | 0             | 8      | 2         |
| Règles 51-60 | 0        | 0           | 1             | 9      | 0         |
| Règles 61-70 | 0        | 0           | 3             | 7      | 0         |
| Règles 71-80 | 0        | 0           | 0             | 10     | 0         |
| **Total**    | **5**    | **11**      | **4**         | **60** | **16**    |

### Résumé des actions

- **16 règles commentées** dans `Sophia.ps1` (conflits + redondances Atlas)
- **4 règles conditionnelles** Defender (60-62, 66) : fonctionnent si Defender présent, skip auto sinon
- **60 règles conservées** sans modification : compatibles ou complémentaires avec Atlas
- **0 erreur de stabilité** identifiée — toutes les modifications sont sûres pour le gaming et la stabilité système

### Conflits critiques neutralisés

1. `DiagnosticDataLevel -Minimal` — réactivait AllowTelemetry, contrait Atlas privacy
2. `ErrorReporting -Default` — réactivait WerSvc, contrait Atlas disable-win-error-reporting
3. `FeedbackFrequency -Automatically` — réactivait DoNotShowFeedbackNotifications=0
4. `RecommendedTroubleshooting -Default` — réactivait télémétrie + WER via politique OOBE
5. `LocalSecurityAuthority -Enable` — activait LSA/VBS, contrait les optimisations perf Atlas

### Architecture de protection

Le script Sophia modifié est **sûr** pour utilisation sur AtlasOS grâce à :

- Les **16 commentaires annotés** `[AtlasOS CONFLICT]` / `[AtlasOS REDUNDANT]` dans `Sophia.ps1`
- Le **check `$Global:DefenderDefaultAV`** natif qui protège les 4 fonctions Defender conditionnelles
- La **nature interactive** de `Uninstall-UWPApps` qui s'adapte aux apps déjà retirées par Atlas
- Les **contrôles de présence** (`Get-Package`, `Get-AppxPackage`) intégrés dans les fonctions

# Azure VM Backup Lab: Poore Steps

Hands-on lab: Azure VM ka backup lena, file galti se delete karke **File Recovery** se wapas lana, aur phir sahi tareeke se cleanup karna.

| | |
|---|---|
| **Region** | Central India |
| **Resource Group** | `gunjanrg` |
| **VM** | `Gunjanvm001` (Ubuntu, B1s, Standard HDD, Trusted Launch) |
| **Vault** | `Gunjan-recovery-vault` (Recovery Services Vault) |
| **Time** | ~1.5-2 ghante (pehla backup ~27 min) |

## Table of Contents

- [A. Setup](#a-setup)
- [B. Vault Settings](#b-vault-settings)
- [C. Test Data](#c-test-data)
- [D. Backup Enable](#d-backup-enable)
- [E. Pehla Backup](#e-pehla-backup)
- [F. Disaster Simulate](#f-disaster-simulate)
- [G. File Recovery](#g-file-recovery)
- [H. Unmount](#h-unmount-zaroori)
- [I. Cleanup](#i-cleanup-kharcha-band)
- [Lab me aayi Problems aur Fix](#lab-me-aayi-problems-aur-fix)
- [Interview Key Points](#interview-key-points)

---

## A. Setup

1. **Resource Group** banao (`gunjanrg`, Central India).
2. **VM** banao (`Gunjanvm001`, Ubuntu, B1s, Standard HDD).
3. **Recovery Services Vault** banao (`Gunjan-recovery-vault`, VM ke same region me).

> **Key point:** Vault aur VM ka region same hona chahiye.

## B. Vault Settings

4. Vault → **Settings → Properties → Backup Configuration → Update → Locally-redundant → Save**.
5. **Soft delete Always On** hota hai, band nahi hota (14 din).

> **Key point:** Redundancy (LRS/GRS) sirf **pehla item protect karne se pehle** badal sakte ho.

## C. Test Data

6. VM terminal me:

```bash
# Test file banata hai
echo "mera important data" > /home/azureuser/important.txt

# File ban gayi ye check karta hai
ls
```

## D. Backup Enable

7. Vault → **Getting started → Backup** → Workload **Azure** → Type **Virtual machine** → **Backup**.
8. Policy: **Enhanced** chuno (Standard fail hui kyunki VM **Trusted Launch** hai).
9. **Add** → VM tick → **OK** → **Enable backup**.
10. **Backup Jobs** me "Configure backup" **Completed** hone do (~1-2 min).

> **Key point:** Trusted Launch VM ke liye **Enhanced policy** zaroori hai. Enhanced me multiple backups per day, 30 din tak instant restore, aur snapshot cost thodi zyada.

## E. Pehla Backup

11. Vault → **Protected items → Backup items → Azure Virtual Machine** → VM ke saamne **"..." → Backup now**.
12. **Retain till** date: kal ya 2-3 din baad ki (portal format **Mahina/Din/Saal**, jaise `11/7/2026` = 7 Nov) → **OK**.
13. **Monitoring → Backup Jobs** me progress dekho (is lab me ~27 min lage).

**Backup ke 3 sub tasks:**

| Sub task | Kaam |
|---|---|
| **Take Snapshot** | Instant restore point banta hai |
| **Transfer data to vault** | Sabse lamba step |
| **Validate Backup** | Last me check |

> **Key point:** Pehla backup **full** hota hai, baad ke **incremental**. Progress bar 1% pe atka dikh sakta hai, sub tasks dekho.

## F. Disaster Simulate

14. VM terminal me:

```bash
# File delete karta hai
rm /home/azureuser/important.txt

# Confirm karta hai ki file chali gayi
ls
```

## G. File Recovery

15. Vault → **Backup items → VM → "..." → File Recovery**.
16. Recovery point select → **Download Script** → page pe wala **password** copy karo. **Page band mat karo.**
17. VM me script file banao:

```bash
# Khali file kholta hai (script paste karke Ctrl+O, Enter, Ctrl+X)
nano recover.py
```

18. Script chalao:

```bash
# Script ko python3 se chalata hai, password maangta hai, recovery point mount karta hai
sudo python3 recover.py
```

> `sudo ./recover.py` isliye fail hua tha ki Ubuntu 22.04 me `python` nahi, `python3` hota hai. Aur `./` me slash lagana mat bhoolna (`.recover.py` galat hai).

19. Output me **mount path** milta hai: `/home/azureuser/Gunjanvm001-<timestamp>/Volume1`
    - **Volume1** = root filesystem, file isi me milti hai.
    - `sda14` mount fail hona normal hai (BIOS boot partition).
20. `q` dabao, phir:

```bash
# Recovery point me file dikhata hai
sudo ls <mount-path>/Volume1/home/azureuser/

# File wapas copy karta hai
sudo cp <mount-path>/Volume1/home/azureuser/important.txt /home/azureuser/

# Content check: "mera important data" aaye to recovery successful
cat /home/azureuser/important.txt
```

## H. Unmount (Zaroori)

21. Portal → **File Recovery** page → **Unmount Disks**.
22. VM me:

```bash
# Mount folders saaf karta hai (portal se unmount ke baad hi chalana)
sudo python3 recover.py clean
```

> **Zaroori:** Unmount na karo to disk restore point par **SAS access khula reh jata hai**, aur baad me snapshot delete nahi hota (is lab me yehi hua, neeche dekho).

## I. Cleanup (Kharcha Band)

**Sahi order:**

23. Vault → **Backup items → VM → "..." → Stop backup → Delete Backup Data** → VM ka naam type karo → reason → **Stop backup**.
24. **Backup Jobs** me delete job **Completed** hone do (2-5 min). **Isse pehle RG delete mat karo.**
25. Phir **Resource groups → `gunjanrg` → Delete resource group** (naam type karke).
26. **Resource groups** me `AzureBackupRG_centralindia_1` check karo. Khud saaf ho jata hai. Na ho to neeche "SAS revoke" wala fix karo.
27. Agar vault bacha ho (soft delete ki wajah se), **14 din baad** vault delete karo.

> `AzureBackupRG_...` Azure Backup service khud banati hai (instant-restore snapshots ke liye). Isko pehle se mat chhedo.

### Agar `AzureBackupRG_...` delete nahi ho raha (SAS revoke fix)

Portal me sirf generic error aata hai: *"Failed to delete restore point"*. **Cloud Shell (Bash)** me asli error milta hai:

```bash
az restore-point delete -g AzureBackupRG_centralindia_1 \
  --collection-name <COLLECTION> --name <RESTORE_POINT> --yes
```

Error: `DiskRestorePointUsedByCustomer: There is an active shared access signature outstanding for disk restore point ... Call EndGetAccess before deleting`

Fix, ek ek karke:

```bash
# 1) Disk restore point par khula SAS access band karta hai
az rest --method post --url "https://management.azure.com/subscriptions/<SUB_ID>/resourceGroups/AzureBackupRG_centralindia_1/providers/Microsoft.Compute/restorePointCollections/<COLLECTION>/restorePoints/<RESTORE_POINT>/diskRestorePoints/<DISK_RP>/endGetAccess?api-version=2024-03-02"

# 2) Restore point delete karta hai
az restore-point delete -g AzureBackupRG_centralindia_1 --collection-name <COLLECTION> --name <RESTORE_POINT> --yes

# 3) Khali collection delete karta hai
az restore-point collection delete -g AzureBackupRG_centralindia_1 --collection-name <COLLECTION> --yes

# 4) Resource group delete karta hai
az group delete -n AzureBackupRG_centralindia_1 --yes --no-wait
```

- `<DISK_RP>` ka naam delete wale error me hi likha hota hai (`Gunjanvm001_OsDisk_1_...`).
- `api-version` ka error aaye to us error me supported versions likhe hote hain, unme se koi lagao (is lab me `2024-03-01` fail hua aur `2024-03-02` chala).
- `<SUB_ID>` apni subscription ID se badlo. **Public repo me asli ID mat daalna.**

---

## Lab me aayi Problems aur Fix

| Problem | Wajah | Fix |
|---|---|---|
| Soft delete band nahi hua | Naye portal me **Always-on** | Skip kiya, vault 14 din baad delete |
| Standard policy fail hui | VM **Trusted Launch** hai | **Enhanced policy** |
| Retention date confusing | Portal format Mahina/Din/Saal | `11/7/2026` = 7 Nov |
| `sudo ./recover.py` not found | Ubuntu 22.04 me `python` nahi, `python3` hai | `sudo python3 recover.py` |
| `.recover.py` command not found | `./` ki jagah `.` likha | `./` ya `python3` se chalao |
| Snapshot aur RG delete nahi hue | **Unmount nahi kiya**, disk restore point par SAS access khula raha | `EndGetAccess` revoke karke delete |
| Orphan snapshot bacha | Backup data delete hone se pehle RG delete kiya | Pehle **Stop backup + Delete data**, phir RG |
| `api-version 2024-03-01` fail | Is resource type ke liye supported nahi | `2024-03-02` use kiya |

---

## Interview Key Points

- **Backup flow:** Snapshot (instant tier) → Vault transfer → Validate.
- **Redundancy:** Pehle item se pehle set karo.
- **Trusted Launch VM:** Enhanced policy chahiye.
- **Soft delete:** 14 din, ransomware ya accidental delete se bachata hai.
- **File Recovery:** Recovery point ko VM me mount karke sirf file nikalte hain, poora VM restore nahi karna padta.
- **File Recovery ke baad Unmount Disks zaroori**, warna SAS access khula reh jata hai aur snapshot delete nahi hota.
- **Cleanup order:** Stop backup + Delete data → job complete → RG delete → vault 14 din baad.
- **`AzureBackupRG_...`** Azure Backup service ka apna RG hai, ise manually na chhedo.

**Interview me bolne ka tareeka:** *"Mere lab me X problem aayi, Y wajah thi, Z se fix kiya."*
Azure VM Backup Lab: Poore Steps (Short Notes)

A. Setup

1. Resource Group banao (gunjanrg, Central India).
2. VM banao (Gunjanvm001, Ubuntu, B1s, Standard HDD).
3. Recovery Services Vault banao (Gunjan-recovery-vault, same region as VM).

Key point: Vault aur VM ka region same hona chahiye.

B. Vault settings

4. Vault → Settings → Properties → Backup Configuration → Update → Locally-redundant → Save.
5. Soft delete Always On hota hai, band nahi hota (14 din).

Key point: Redundancy (LRS/GRS) sirf pehla item protect karne se pehle badal sakte ho.

C. Test data

6. VM terminal me:

   echo "mera important data" > /home/azureuser/important.txt

→ test file banata hai.

   ls

→ file ban gayi ye check karta hai.

D. Backup enable

7. Vault → Getting started → Backup → Workload Azure → Type Virtual machine → Backup.
8. Policy: Enhanced chuno (Standard fail hui kyunki VM Trusted Launch hai).
9. Add → VM tick → OK → Enable backup.
10. Backup jobs me “Configure backup” Completed hone do (~1-2 min).

Key point: Trusted Launch VM ke liye Enhanced policy zaroori hai. Enhanced me multiple backups/day, 30 din tak instant restore aur snapshot cost thodi zyada.

E. Pehla backup

11. Vault → Protected items → Backup items → Azure Virtual Machine → VM ke saamne “...” → Backup now.
12. Retain till date: kal ya 2-3 din baad ki (format Mahina/Din/Saal) → OK.
13. Monitoring → Backup Jobs me progress dekho (mera lab me ~27 min lage).

Backup ke 3 sub tasks:

Take Snapshot (instant restore point banta hai)
Transfer data to vault (sabse lamba)
Validate Backup

Key point: Pehla backup full hota hai, baad ke incremental. Progress bar 1% pe atka dikh sakta hai, sub tasks dekho.

F. Disaster simulate

14. VM terminal me:

    rm /home/azureuser/important.txt

→ file delete karta hai.

    ls

→ confirm ki file chali gayi.

G. File Recovery

15. Vault → Backup items → VM → “...” → File Recovery.
16. Recovery point select → Download Script → page pe wala password copy karo. Page band mat karo.
17. VM me script file banao:

    nano recover.py

→ khali file kholta hai (script paste karke `Ctrl+O`, `Enter`, `Ctrl+X`).

18. Script chalao:

    sudo python3 recover.py

→ script ko python3 se chalata hai, password maangega, recovery point mount karta hai.

(`sudo ./recover.py` isliye fail hua tha ki Ubuntu 22.04 me `python` nahi, `python3` hota hai. Aur `./` me slash lagana mat bhoolna.)

19. Output me mount path milta hai: /home/azureuser/Gunjanvm001-<timestamp>/Volume1. Volume1 = root filesystem, jahan file milti hai. sda14 fail hona normal hai (BIOS boot partition).
20. q dabao, phir:

    sudo ls <mount-path>/Volume1/home/azureuser/

→ recovery point me file dikhata hai.

    sudo cp <mount-path>/Volume1/home/azureuser/important.txt /home/azureuser/

→ file wapas copy karta hai.

    cat /home/azureuser/important.txt

→ content check, **"mera important data"** aaye to recovery successful.

H. Unmount

21. Portal → File Recovery page → Unmount Disks.
22. VM me:

    sudo python3 recover.py clean

→ mount folders saaf karta hai.

Key point: Unmount na karo to 12 ghante baad apne aap hota hai.

I. Cleanup (kharcha band)

23. Vault → Backup items → VM → “...” → Stop backup → Delete Backup Data → VM ka naam type karo → reason → Stop backup. Job Completed hone do.
24. Resource groups → gunjanrg → Delete resource group (naam type karke). Vault soft-deleted data ki wajah se delete nahi hoga, VM/disk/IP sab chale jayenge. Vault free hai.
25. AzureBackupRG_centralindia_1 RG mile to delete karo.
26. 19 November 2026 pe vault + gunjanrg delete karo (soft delete ke 14 din baad).

Interview ke Key Points (ek line me)

Backup flow: Snapshot (instant tier) → Vault transfer → Validate.
Redundancy: Pehle item se pehle set karo.
Trusted Launch VM: Enhanced policy chahiye.
Soft delete: 14 din, ransomware/accidental delete se bachata hai.
File Recovery: Recovery point ko VM me mount karke sirf file nikalte hain, poora VM restore nahi karna padta.
Cleanup order: Stop backup + Delete data → RG delete → vault 14 din baad.

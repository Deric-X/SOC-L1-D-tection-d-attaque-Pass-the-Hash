# 🔐 SOC L1 – Détection d’une Attaque Pass-the-Hash

## 📌 1. Objectif du Projet

Ce projet a pour objectif de :

* Simuler une attaque **Pass-the-Hash** dans un environnement Active Directory.
* Observer les traces laissées sur les systèmes Windows.
* Détecter et analyser l’attaque à l’aide du SIEM **Wazuh**.
* Cartographier l’attaque selon le framework **MITRE ATT&CK**.
* Proposer des recommandations de mitigation.

Ce projet démontre les compétences d’un **analyste SOC niveau 1** :
collecte de logs, analyse d’événements, corrélation et rédaction d’alerte.

---

# 🏗 2. Architecture du Lab
<img width="861" height="769" alt="Screenshot_20260221_185159" src="https://github.com/user-attachments/assets/824cf66b-1189-4aed-a581-dee80ed9e973" />


## 🧩 Composants utilisés

| Système        | Rôle                                     |
| -------------- | ---------------------------------------- |
| pfSense        | Firewall & segmentation réseau           |
| Wazuh          | SIEM / HIDS                              |
| Kali Linux     | Machine attaquante                       |
| Windows Server | Contrôleur de domaine (Active Directory) |
| Windows Client | Machine victime                          |

---

# 🚨 3. Phase 1 – Simulation de l’Attaque

---

## 3.1 LLMNR Poisoning – Capture du Hash

L’attaquant utilise :

Responder

<img width="679" height="450" alt="image" src="https://github.com/user-attachments/assets/55d4d1d2-cadc-4566-9bfa-eb90549a876b" /> 
<img width="1097" height="132" alt="Screenshot_20260221_231804" src="https://github.com/user-attachments/assets/71ffe378-67f3-429d-9d21-09b1c3ecfcb0" />

Objectif :

* Écouter le trafic réseau
* Empoisonner les requêtes LLMNR / NetBIOS
* Capturer un hash NTLMv2

Lorsqu’une machine tente de résoudre un nom inexistant :

1. Elle envoie une requête LLMNR.
2. L’attaquant répond frauduleusement.
3. La victime envoie un hash NTLM.

⚠ À ce stade :
Aucune alerte Windows directe n’est générée.

---

## 3.2 Crackage du mot de passe

Commande utilisée :

```bash
hashcat -m 5600 hash.txt rockyou.txt
```
ici on uttiliser hashcat pour cracker le hash on obtien avec l'attaque respoder
Outil : Hashcat

* `-m 5600` → format NetNTLMv2
* Attaque par dictionnaire
<img width="1082" height="209" alt="Screenshot_20260221_232522-1" src="https://github.com/user-attachments/assets/921445ed-8374-40df-b6c6-182421e01609" />

Objectif :
Trouver le mot de passe en clair correspondant au hash capturé.

---

## 3.3 Mouvement latéral via SMB

Commande utilisée :

```bash
crackmapexec smb 192.168.2.14 -u Adrianot -p Todisoa123 -d iet.local --exec-method smbexec -x "whoami /all"
```

Outil : CrackMapExec

Objectif :

* S’authentifier à distance
* Exécuter une commande
* Vérifier les privilèges

Cette étape confirme la compromission du système.

---

# 🛡 4. Phase 2 – Détection & Analyse SOC

<img width="1920" height="1080" alt="Screenshot_20260221_194943" src="https://github.com/user-attachments/assets/1e58979b-a195-483f-a7c3-fbf0a3af10e5" />

---

## 4.1 Événements détectés par Wazuh

Wazuh collecte les logs Windows suivants :

| Event ID    | Description         | Indicateur           |
| ----------- | ------------------- | -------------------- |
| 4624        | Successful Logon    | Connexion réussie    |
| 4625        | Failed Logon        | Tentatives suspectes |
| 4776        | NTLM Authentication | Usage NTLM           |
| LogonType 3 | Network Logon       | Connexion distante   |

---

## 4.2 Tableau d’analyse SOC

| Élément                      | Analyse SOC                     |
| ---------------------------- | ------------------------------- |
| AuthenticationPackage = NTLM | Kerberos aurait dû être utilisé |
| LogonType = 3                | Connexion réseau distante       |
| Source IP = Kali             | IP non autorisée                |
| Exécution SMB                | Mouvement latéral               |

🔎 data.win.system.eventID:4624

# Event ID 4624 – Successful Logon (NTLM)
```bash
EventID: 4624
LogonType: 3
AuthenticationPackage: NTLM
WorkstationName: KALI
SourceIP: 192.168.2.10
TargetUser: Adrianot
Domain: IET
```
| Élément      | Valeur       | Interprétation                       |
| ------------ | ------------ | ------------------------------------ |
| Event ID     | 4624         | Connexion réussie                    |
| Logon Type   | 3            | Connexion réseau (SMB / RDP / WinRM) |
| Auth Package | NTLM         | Authentification NTLM                |
| Workstation  | KALI         | Machine attaquante                   |
| IP Source    | 192.168.2.10 | Origine de la connexion              |
| Compte       | Adrianot     | Compte utilisé                       |
| Key Length   | 128          | NTLMv2                               |

# Échec d’accès au compte (Account Access Removal)

| Champ            | Valeur                                              | Interprétation SOC                             |
| ---------------- | --------------------------------------------------- | ---------------------------------------------- |
| Event ID         | 4625                                                | Échec d’ouverture de session                   |
| Logon Type       | 3                                                   | Connexion réseau                               |
| Authentication   | NTLM                                                | Authentification réseau                        |
| Source IP        | 192.168.2.10                                        | Machine attaquante (Kali)                      |
| Target User      | Adrianot                                            | Compte AD visé                                 |
| Domain           | iet.local                                           | Domaine cible                                  |
| Failure Reason   | Nom d’utilisateur inconnu ou mot de passe incorrect | Possible tentative Pass-the-Hash ou bruteforce |
| Status/SubStatus | 0xC000006D / 0xC000006A                             | Code NTLM pour mauvaise authentification       |

Analyse SOC :

* Tentative de connexion réseau à partir d’une machine Kali.

* Utilisation d’authentification NTLM.

* Échec de login → indicateur possible de brute force ou hash incorrect.

* Logon Type 3 → accès distant via SMB / RDP / WinRM.

MITRE ATT&CK :

Tactique	Technique	ID
Impact	Account Access Removal	T1531

Recommandation :

* Surveiller les EventID 4625 répétitifs.

* Restreindre l’accès administratif.

* Activer alertes Wazuh sur tentatives NTLM échouées.

# Connexion distante réussie (Remote Desktop Protocol)

| Champ          | Valeur        | Interprétation SOC                      |
| -------------- | ------------- | --------------------------------------- |
| Event ID       | 4624          | Connexion réussie                       |
| Logon Type     | 3             | Connexion réseau (SMB / RDP)            |
| Authentication | NTLM / NTLMv2 | Authentification possible Pass-the-Hash |
| Source IP      | 192.168.2.10  | Machine Kali                            |
| Target User    | Adrianot      | Compte AD compromis                     |
| Workstation    | KALI          | Machine attaquante                      |
| Key Length     | 128           | NTLMv2 valide                           |


🎯 Conclusion SOC :

Comportement anormal compatible avec une attaque Pass-the-Hash.

* Échec 4625 → tentative d’accès avec hash incorrect ou brute force → alerte précoce.

* Succès 4624 → Pass-the-Hash confirmé → mouvement latéral effectué.

* Corrélation : même IP source, même compte, même domaine → attaque Pass-the-Hash typique.

* Action SOC recommandée : isolation de la machine, rotation des mots de passe, audits NTLM/SMB.

---

# 🧠 5. Cartographie MITRE ATT&CK

Framework utilisé : MITRE Corporation

| Tactique          | Technique          | ID        |
| ----------------- | ------------------ | --------- |
| Credential Access | Brute Force        | T1110     |
| Credential Access | Pass-the-Hash      | T1550.002 |
| Lateral Movement  | SMB / Admin Shares | T1021.002 |
| Persistence       | Valid Accounts     | T1078     |

---

# 🔍 6. Ce que voit réellement le SOC

Important :

LLMNR poisoning seul ne génère pas d’événement Windows visible.

La détection commence uniquement lorsque :

* Le hash est réutilisé
* Une authentification NTLM réussie apparaît
* Une commande distante est exécutée

C’est pourquoi Wazuh détecte le Pass-the-Hash mais pas directement LLMNR.

---

# 🛠 7. Recommandations de Sécurité

Pour prévenir ce type d’attaque :

✅ Désactiver LLMNR via GPO
✅ Désactiver NTLM si possible
✅ Activer SMB Signing
✅ Implémenter LAPS
✅ Restreindre privilèges administrateurs
✅ Surveiller Event ID 4624 avec NTLM
✅ Segmentation réseau stricte

---

# 📊 8. Compétences Démontrées

* Simulation d’attaque réelle
* Analyse logs Windows
* Utilisation SIEM (Wazuh)
* Corrélation d’événements
* Mapping MITRE ATT&CK
* Rédaction d’analyse SOC

---

# 🎓 9. Conclusion

Ce projet démontre une chaîne d’attaque complète :

LLMNR Poisoning
→ Capture Hash
→ Crackage
→ Mouvement Latéral
→ Détection SOC

Il met en évidence l’importance :

* De la surveillance des authentifications NTLM
* De la corrélation des logs
* D’une politique de sécurité robuste

---


On passe au niveau supérieur ? 💪

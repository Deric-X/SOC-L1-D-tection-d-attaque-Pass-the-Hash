# SOC-L1-D-tection-d-attaque-Pass-the-Hash

## 🎯 Objectif
Ce projet a pour but de simuler une attaque **Pass-the-Hash** dans un environnement Active Directory et de démontrer la détection et l’alerte via un SOC niveau 1 avec **Wazuh**.

---

## 🏗️ Architecture du Lab
<img width="861" height="769" alt="Screenshot_20260221_185159" src="https://github.com/user-attachments/assets/704bb87f-5843-44db-8e77-4c847e7ace75" />


### 🔧 Composants

- **Firewall** : pfSense  
- **SIEM / HIDS** : Wazuh  
- **Attaquant** : Kali Linux  
- **Server** : Windows Server (AD Domain Controller)  
- **Cible** : poste utilisateur simulé  

### 🔴 Phase 1 – Simulation de l’attaque
1️⃣ **Capture des hashes NTLM**  
- **L'attaque lance responder
NTLM (New Technology LAN Manager) est une suite de protocoles d’authentification développés par Microsoft permettant de confirmer l’identité des utilisateurs et de protéger l’intégrité et la confidentialité de leurs activités sur un réseau.
- <img width="940" height="803" alt="responder" src="https://github.com/user-attachments/assets/4e0420d0-de68-42dc-8ccf-c8e135a78d96" />
- Responder intercepte le hash du compte `Adrianot`.
- <img width="1097" height="132" alt="Screenshot_20260221_231804" src="https://github.com/user-attachments/assets/6de571bf-9f04-48df-8143-9429651e1a37" />
- 2️⃣ **Crackage du mot de passe**  
- Hashcat permet d’obtenir le mot de passe en clair.
- <img width="1094" height="176" alt="image" src="https://github.com/user-attachments/assets/f7f2e14e-242c-482c-8ccf-880c87fc6b54" />
- <img width="1082" height="209" alt="Screenshot_20260221_232522-1" src="https://github.com/user-attachments/assets/0b2a665b-bd20-498e-8fef-ff0beddc3dc0" />
3️⃣ **Mouvement latéral (Pass-the-Hash)**
- CrackMapExec SMB permet l’authentification distante via NTLM sans mot de passe en clair.  
- LogonType détecté : 3 (Network Logon)  
- Processus : NtLmSsp
## 🔎 Phase 2 – Détection par le SOC (Wazuh)


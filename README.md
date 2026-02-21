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

### 📡 Réseau

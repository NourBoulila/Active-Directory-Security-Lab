
Active Directory Security Lab: Kerberoasting & gMSA-Absicherung

Haftungsausschluss (Disclaimer): Dieses Projekt wurde ausschließlich in einer isolierten, lokalen virtuellen Testumgebung zu Lern- und Forschungszwecken durchgeführt.

📌 Projektübersicht

Dieses Lab simuliert eine typische Active-Directory-Unternehmensumgebung, um zu demonstrieren, wie ein Angreifer mit geringen Benutzerrechten über Kerberoasting seine Rechte ausweiten (Privilege Escalation) kann und wie sich dieser Angriffsvektor durch Group Managed Service Accounts (gMSA) vollständig schließen lässt.

Lab-Architektur

• Domäne: domain.local
• Domain Controller (DC01): Windows Server (Active Directory Domain Services, DNS, KDS Root Key)
• Mitgliedsserver (FIL01): Windows Server (Fileserver)
• Angreifer-System: Kali Linux (mit Impacket und Hashcat)

Angriffs- und Schutzszenario

1. Initial Access: Simulierte Kompromittierung eines Standard-Domänenbenutzers (intern).
2. Enumeration & Exploitation: Auffinden eines Dienstkontos (svc-backup) mit registriertem Service Principal Name (SPN) und lokalen Administratorrechten auf FIL01. Extraktion des TGS-Tickets und Offline-Cracking des Passworts.
3. Absicherung (Remediation): Ablösung des herkömmlichen Dienstkontos durch ein gMSA mit automatisch rotiertem 120-Zeichen-Passwort.


⚔️ Teil 1: Angriffssimulation (Kerberoasting)


1. SPN-Enumeration

Aus dem Kontext des kompromittierten Benutzers intern wurde das Active Directory nach Konten mit registrierten SPNs durchsucht:
powershell
    -setspn -Q  `*/*`   

<img width="1060" height="297" alt="image" src="https://github.com/user-attachments/assets/d8afdce5-0683-4062-86e1-30bfb4e4e864" />

 
Ergebnis: Identifikation des Dienstkontos svc-backup, das einem Dienst auf FIL01 zugewiesen war. Eine weitere Überprüfung bestätigte, dass svc-backup über lokale Administratorrechte auf FIL01 verfügte.

2. Anfordern des TGS-Tickets

Mit impacket-GetUserSPNs wurde vom Kali-System aus ein Kerberos Ticket Granting Service (TGS)-Ticket für svc-backup angefordert und lokal gespeichert:
impacket-GetUserSPNs domain.local/intern:<PASSWORT> \
  -dc-ip <DC_IP> \
  -request-user svc-backup \
  -outputfile backup-hash.txt

  
<img width="1366" height="357" alt="image" src="https://github.com/user-attachments/assets/842a3da5-6051-406d-93e0-f87331a480b6" />



3. Offline-Passwort-Cracking

Da das TGS-Ticket mit dem Passwort-Hash von svc-backup verschlüsselt ist, konnte der Hash offline mit Hashcat (Modus 19700 für Kerberos 5 TGS-REP etype 17/18) geknackt werden:
bash
hashcat -m 19700 backup-hash.txt /Desktop/lab-password.txt

<img width="1376" height="665" alt="image" src="https://github.com/user-attachments/assets/52159b77-1ed3-4645-b983-7dcb22d45b82" />

Ergebnis: Erfolgreiche Wiederherstellung des Klartext-Passworts für svc-backup, wodurch administrativer Zugriff auf FIL01 erlangt wurde.





 ## 🛡️ Teil 2: Schutzmaßnahmen (Implementierung von gMSA)

Da das Szenario zu Demonstrationszwecken mit einem simulierten SPN aufgebaut wurde, lag der Fokus der Absicherung auf der Ablösung des verwundbaren Standard-Dienstkontos (`svc-backup`) durch ein **Group Managed Service Account (gMSA)** sowie der Verifizierung auf dem Zielsystem `FIL01`.

### 1. Vorbereitung auf dem Domain Controller (`DC01`)
* **KDS Root Key erstellt:** Initialisierung des Key Distribution Services zur automatischen Passwortgenerierung (`Add-KdsRootKey`).
* **Sicherheitsgruppe angelegt:** Erstellung der Gruppe `gMSA_FIL01_Hosts` und Aufnahme des Computerkontos `FIL01$`, sodass ausschließlich dieser Server das verwaltete Passwort abrufen darf.
* **gMSA erstellt & SPN zugewiesen:** Anlage des Kontos `gmsa-backup$` (`lab.local`) inklusive Zuweisung des entsprechenden SPNs und Deaktivierung des alten Kontos `svc-backup`.


<img width="1018" height="418" alt="image" src="https://github.com/user-attachments/assets/5b29d3bd-48da-4383-84f3-48b500c47e5b" />

### 2. Installation & Verifizierung auf dem Mitgliedsserver (`FIL01`)
Nach der Installation des gMSA auf `FIL01` wurde über PowerShell geprüft, ob der Server das automatisch generierte 120-Zeichen-Passwort erfolgreich vom Domain Controller abrufen kann:

```powershell
Install-ADServiceAccount -Identity "gmsa-backup"
Test-ADServiceAccount -Identity "gmsa-backup"
```
**Ergebnis:** `True` – `FIL01` ist autorisiert und kann das gMSA ohne manuelle Passwortverwaltung nutzen.

### 3. Gegenprobe (Kerberoasting-Test)
Ein erneuter Versuch mit `impacket-GetUserSPNs` zeigt, dass zwar weiterhin ein TGS-Ticket für den registrierten SPN angefordert werden kann, ein Offline-Cracking des Passworts jedoch durch die **kryptografisch zufällige Länge von 120 Zeichen** und die **automatische 30-Tage-Rotation** vollständig verhindert wird.





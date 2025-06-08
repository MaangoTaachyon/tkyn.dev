# Zoho Quick Assist Arbitrary File Delete to SYSTEM

[https://youtu.be/OtG5gkOxDDc](https://youtu.be/OtG5gkOxDDc)

### Overview

A critical vulnerability exists in Zoho Quick Assist, a remote desktop support software that allows technicians to view, control, and troubleshoot end-user devices. While the main application runs with user privileges, it includes background processes with elevated permissions to facilitate system-level operations. The service fails to properly validate and sanitize user-controlled file paths in recursive delete operations when using the "Send Logs" function available from the system tray icon. This allows an unprivileged local attacker to delete arbitrary files or directories on the system, potentially leading to system integrity loss, denial of service, or **Local Privilege Escalation (LPE)** through tampering with security-critical files.

I was added to the Zoho hall of fame for this discovery [https://www.zoho.com/security/hall-of-fame/](https://www.zoho.com/security/hall-of-fame/).

[https://www.zoho.com/assist/](https://www.zoho.com/assist/)

---

### Timeline:

![](/assets/images/zoho1.png)

![](/assets/images/zoho2.png)

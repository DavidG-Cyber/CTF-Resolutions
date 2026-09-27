# 🛡️ Penetration Testing y Write-ups de CTF

Bienvenido a mi portafolio de ciberseguridad. Mi nombre es David Guaño y este repositorio contiene documentación detallada, narrativas de ataques y metodologías aplicadas durante mis prácticas de penetration testing en diversas plataformas. 

Mi enfoque actual está en el penetration testing práctico de redes y aplicaciones web, preparándome activamente para la certificación **eJPTv2** y construyendo una base sólida hacia la **OSCP**.

## 🎯 Objetivos y Metodología
Los write-ups en este repositorio enfatizan una metodología estructurada:
1. **Reconocimiento y Enumeración:** Escaneo exhaustivo de puertos y enumeración de servicios.
2. **Análisis de Vulnerabilidades:** Identificación de malas configuraciones y servicios explotables.
3. **Explotación:** Obtención de acceso inicial con bajos privilegios.
4. **Escalada de Privilegios:** Enumeración local profunda para lograr acceso `root` o `SYSTEM`.

## 🛠️ Herramientas y Tecnologías
A través de estas máquinas, demuestro competencia práctica con herramientas estándar de la industria:
* **Redes y Web:** Nmap, Burp Suite, Wireshark, FoxyProxy, Hydra
* **SO y Scripting:** Linux (Parrot OS), Bash, Python
* **Infraestructura:** Docker, conceptos básicos de AWS para monitoreo defensivo

## 📚 Write-ups de Máquinas (Índice)

### TryHackMe
| Nombre de la Máquina | Dificultad | SO | Enlace al Write-up | Enfoque / Vulnerabilidad |
| :--- | :---: | :---: | :--- | :--- |
| Vulnversity | Fácil | Linux | [Leer Write-up](./TryHackMe/Vulnversity) | Upload Bypass / SUID (systemctl) |
| Pickle Rick | Fácil | Linux | [Leer Write-up](./TryHackMe/Pickle-Rick) | Enumeración Web / Reverse Shell / Sudo Misconfiguration |


### HackTheBox
*Próximamente...*

---
*Descargo de responsabilidad: Todos los write-ups tienen fines estrictamente educativos. Las técnicas de explotación se aplicaron únicamente dentro de entornos de laboratorio autorizados.*

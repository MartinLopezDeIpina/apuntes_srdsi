# Laboratorio DNS - Guía Completa (Ejercicios 1-6)

## Configuración Inicial del Laboratorio

### Datos del entorno:
- **Dominio:** `srdsi.lab`
- **Servidor DNS primario:** `dns1.srdsi.lab`
- **IP del servidor:** `172.16.0.136` (ejemplo, usar tu IP asignada)
- **Sistema operativo:** Debian/Ubuntu
- **Recomendación:** VM con 1GB RAM y 1 CPU

---

## Ejercicio 1: Configuración de un Servidor de Nombres (Servidor Caché)

### ¿Qué se pide?
Instalar y configurar un servidor DNS básico que funcione como servidor caché, capaz de resolver consultas DNS utilizando servidores externos.

### Objetivos:
1. Instalar BIND9
2. Configurar el hostname del servidor
3. Configurar resolución local
4. Hacer que el servidor actúe como caché DNS

---

### Paso 1: Instalación del servicio DNS

```bash
sudo apt-get update
sudo apt-get install bind9 bind9utils dnsutils
```

**Explicación:** 
- `bind9`: El servidor DNS
- `bind9utils`: Utilidades de administración
- `dnsutils`: Herramientas de consulta DNS (dig, nslookup, etc.)

### Paso 2: Configurar el hostname del servidor

```bash
# Cambiar el nombre del host
sudo nano /etc/hostname
```
Contenido: `dns1`

```bash
# Configurar el archivo hosts
sudo nano /etc/hosts
```
Contenido:
```
127.0.0.1 localhost
127.0.1.1 dns1.srdsi.lab dns1
```

```bash
# Actualizar el hostname
sudo hostnamectl set-hostname dns1
```

### Paso 3: Verificar el hostname

```bash
hostname --fqdn
```
**Resultado esperado:** `dns1.srdsi.lab`

### Paso 4: Configurar resolución DNS local

```bash
sudo nano /etc/resolv.conf
```
Contenido:
```
domain srdsi.lab
nameserver 127.0.0.1
```

```bash
# Proteger el archivo contra cambios automáticos
sudo chattr +i /etc/resolv.conf
```

### Paso 5: Reiniciar el servicio

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

### Paso 6: Verificar funcionamiento

```bash
# Verificar estado del servidor
sudo rndc status

# Probar resolución DNS
dig @127.0.0.1 google.com
```

**¿Qué hace esto?** El servidor ahora actúa como caché DNS, reenvía consultas a servidores raíz/externos y almacena las respuestas para consultas futuras.

---

## Ejercicio 2: Configurar Servidor Autorizado para la Zona srdsi.lab

### ¿Qué se pide?
Convertir el servidor caché en un servidor autorizado para la zona `srdsi.lab`, creando una zona primaria con registros DNS propios.

### Objetivos:
1. Crear zona de resolución directa
2. Crear zona de resolución inversa  
3. Configurar registros DNS básicos
4. Verificar funcionamiento

---

### Paso 1: Configurar las zonas en BIND

```bash
sudo nano /etc/bind/named.conf.local
```

Agregar:
```bind
// zona de resolución directa "srdsi.lab"
zone "srdsi.lab" {
    type primary;
    file "/etc/bind/db.srdsi.lab";
};

// zona de resolución inversa "0.16.172.in-addr.arpa"
zone "0.16.172.in-addr.arpa" {
    type primary;
    file "/etc/bind/db.0.16.172";
};
```

**Explicación:**
- `type primary`: Este servidor es la fuente autoritativa
- `file`: Ubicación del archivo de zona
- La zona inversa permite resolver IPs a nombres

### Paso 2: Crear archivo de zona directa

```bash
sudo nano /etc/bind/db.srdsi.lab
```

Contenido:
```dns
$ORIGIN srdsi.lab.
$TTL 2h
@ IN SOA dns1.srdsi.lab. admin.srdsi.lab. (
    20140401 ; Serial (formato YYYYMMDDNN)
    604800   ; Refresh (1 semana)
    86400    ; Retry (1 día)
    2419200  ; Expire (4 semanas)
    604800 ) ; Negative Cache TTL (1 semana)
;
; Servidores de nombres
srdsi.lab. IN NS dns1.srdsi.lab.
;
; Registros A (nombre -> IP)
dns1  IN A 172.16.0.136
host1 IN A 172.16.0.138
;
; Registros CNAME (alias)
www   IN CNAME host1
```

**Explicación de registros:**
- **SOA**: Start of Authority, define la zona y sus parámetros
- **NS**: Name Server, especifica qué servidor es autoritativo
- **A**: Asocia nombre con dirección IPv4
- **CNAME**: Crea alias (www apunta a host1)

### Paso 3: Crear archivo de zona inversa

```bash
sudo nano /etc/bind/db.0.16.172
```

Contenido:
```dns
$TTL 604800
@ IN SOA dns1.srdsi.lab. admin.srdsi.lab. (
    20140401 ; Serial
    604800   ; Refresh
    86400    ; Retry
    2419200  ; Expire
    604800 ) ; Negative Cache TTL
;
; Servidor de nombres para la zona inversa
@ IN NS dns1.srdsi.lab.
;
; Registros PTR (IP -> nombre)
136 IN PTR dns1.srdsi.lab.
138 IN PTR host1.srdsi.lab.
```

**Explicación:**
- **PTR**: Pointer record, resuelve IP a nombre
- `136` representa `.136` en la red `172.16.0.x`

### Paso 4: Verificar configuración

```bash
# Verificar sintaxis de configuración
sudo named-checkconf /etc/bind/named.conf

# Verificar zona directa
sudo named-checkzone srdsi.lab /etc/bind/db.srdsi.lab

# Verificar zona inversa
sudo named-checkzone 0.16.172.in-addr.arpa /etc/bind/db.0.16.172
```

### Paso 5: Recargar configuración

```bash
sudo systemctl restart bind9
# O alternativamente:
sudo rndc reload
```

### Paso 6: Probar funcionamiento

```bash
# Consulta directa (nombre -> IP)
dig @172.16.0.136 dns1.srdsi.lab
dig @172.16.0.136 host1.srdsi.lab
dig @172.16.0.136 www.srdsi.lab

# Consulta inversa (IP -> nombre)
dig @localhost -x 172.16.0.136
dig @localhost -x 172.16.0.138
```

---

## Ejercicio 3: Configurar Servidor de Nombres Secundario

### ¿Qué se pide?
Configurar un segundo servidor DNS que actúe como servidor secundario, sincronizando automáticamente las zonas desde el servidor primario para proporcionar redundancia.

### Objetivos:
1. Instalar BIND en segunda máquina
2. Configurar como servidor secundario
3. Configurar transferencias de zona
4. Verificar sincronización

---

### Configuración del Servidor Secundario

**Datos del servidor secundario:**
- Nombre: `dns2.srdsi.lab`
- IP: `172.16.0.137` (ejemplo)

### Paso 1: Instalación (igual que ejercicio 1)

```bash
sudo apt-get install bind9 bind9utils dnsutils
```

### Paso 2: Configurar hostname

```bash
# En /etc/hostname
dns2

# En /etc/hosts
127.0.0.1 localhost
127.0.1.1 dns2.srdsi.lab dns2

# Actualizar
sudo hostnamectl set-hostname dns2
```

### Paso 3: Configurar resolución DNS

```bash
sudo nano /etc/resolv.conf
```
Contenido:
```
domain srdsi.lab
nameserver 127.0.0.1
nameserver 172.16.0.136
```

```bash
sudo chattr +i /etc/resolv.conf
```

### Paso 4: Configurar zonas secundarias

```bash
sudo nano /etc/bind/named.conf.local
```

Contenido:
```bind
zone "srdsi.lab" {
    type secondary;
    file "/var/cache/bind/secondary/db.srdsi.lab";
    masterfile-format text;
    masters { 172.16.0.136; };
};

zone "0.16.172.in-addr.arpa" {
    type secondary;
    file "/var/cache/bind/secondary/db.0.16.172";
    masterfile-format text;
    masters { 172.16.0.136; };
};
```

**Explicación:**
- `type secondary`: Servidor esclavo
- `masters`: IP del servidor primario
- Los archivos se almacenan en `/var/cache/bind/secondary/`

### Paso 5: Crear directorio y permisos

```bash
sudo mkdir -p /var/cache/bind/secondary
sudo chown bind:bind /var/cache/bind/secondary
```

### Paso 6: Configurar seguridad (prevenir transferencias no autorizadas)

```bash
sudo nano /etc/bind/named.conf.options
```

Agregar:
```bind
allow-transfer { none; };
```

### Configuración en el Servidor Primario

### Paso 7: Permitir transferencias desde el primario

```bash
sudo nano /etc/bind/named.conf.local
```

Modificar cada zona agregando:
```bind
zone "srdsi.lab" {
    type primary;
    file "/etc/bind/db.srdsi.lab";
    allow-transfer { 172.16.0.137; };
    also-notify { 172.16.0.137; };
};

zone "0.16.172.in-addr.arpa" {
    type primary;
    file "/etc/bind/db.0.16.172";
    allow-transfer { 172.16.0.137; };
    also-notify { 172.16.0.137; };
};
```

**Explicación:**
- `allow-transfer`: Permite transferencias al secundario
- `also-notify`: Notifica cambios automáticamente

### Paso 8: Agregar el servidor secundario a la zona

```bash
sudo nano /etc/bind/db.srdsi.lab
```

Modificar:
```dns
; Incrementar serial
20140402

; Agregar servidor secundario
srdsi.lab. IN NS dns1.srdsi.lab.
srdsi.lab. IN NS dns2.srdsi.lab.

dns2 IN A 172.16.0.137
```

### Paso 9: Reiniciar servicios

```bash
# En el servidor primario
sudo rndc reload

# En el servidor secundario
sudo systemctl restart bind9
```

### Paso 10: Verificar transferencias de zona

```bash
# Verificar transferencia completa (AXFR)
dig @172.16.0.136 srdsi.lab axfr
dig @172.16.0.137 srdsi.lab axfr

# Verificar logs
sudo tail /var/log/syslog
sudo tail /var/log/daemon.log
```

### Paso 11: Monitorear con Wireshark (opcional)

```bash
# En dns1 (para ver transferencias)
sudo systemctl stop bind9
# Ejecutar Wireshark con filtro: ip.addr==172.16.0.137
sudo systemctl start bind9
```

---

## Ejercicio 4: Transaction Signature (TSIG)

### ¿Qué se pide?
Implementar autenticación y autorización segura para las transferencias de zona entre servidores primario y secundario usando claves compartidas TSIG.

### Objetivos:
1. Generar claves TSIG compartidas
2. Configurar autenticación en transferencias
3. Verificar seguridad mejorada

---

### Configuración en el Servidor Primario

### Paso 1: Generar clave TSIG

```bash
cd /etc/bind/
sudo tsig-keygen dns1-dns2_tsigkey > Kdns1-dns2_tsigkey
```

El archivo generado contiene:
```bind
key "dns1-dns2_tsigkey" {
    algorithm hmac-sha256;
    secret "Ok1qR5IW1ROAzOn21Ju==";
};
```

### Paso 2: Incluir la clave en la configuración

```bash
sudo nano /etc/bind/named.conf
```

Agregar:
```bind
include "/etc/bind/Kdns1-dns2_tsigkey";
```

### Paso 3: Configurar zonas para usar TSIG

```bash
sudo nano /etc/bind/named.conf.local
```

Modificar:
```bind
zone "srdsi.lab" {
    type primary;
    file "/etc/bind/db.srdsi.lab";
    also-notify { 172.16.0.137; };
    allow-transfer { key dns1-dns2_tsigkey; };
};
```

### Paso 4: Reiniciar servidor primario

```bash
sudo systemctl restart bind9
```

### Configuración en el Servidor Secundario

### Paso 5: Distribuir la clave (método seguro)

```bash
# Copiar la clave al servidor secundario (usar SCP o método seguro)
sudo nano /etc/bind/Kdns1-dns2_tsigkey
```

Contenido:
```bind
key "dns1-dns2_tsigkey" {
    algorithm hmac-sha256;
    secret "Ok1qR5IW1ROAzOn21Ju==";
};

server 172.16.0.136 {
    keys { dns1-dns2_tsigkey; };
};
```

### Paso 6: Incluir configuración

```bash
sudo nano /etc/bind/named.conf
```

Agregar:
```bind
include "/etc/bind/Kdns1-dns2_tsigkey";
```

### Paso 7: Modificar configuración de zonas

```bash
sudo nano /etc/bind/named.conf.local
```

Modificar:
```bind
zone "srdsi.lab" {
    type secondary;
    file "/var/cache/bind/secondary/db.srdsi.lab";
    masterfile-format text;
    masters { 172.16.0.136 key dns1-dns2_tsigkey; };
};
```

### Paso 8: Reiniciar servidor secundario

```bash
sudo rndc reload
```

### Verificación de TSIG

### Paso 9: Probar transferencias autenticadas

```bash
# Transferencia sin clave (debería fallar)
dig @172.16.0.136 srdsi.lab axfr

# Transferencia con clave (debería funcionar)
sudo dig @172.16.0.136 srdsi.lab axfr -k Kdns1-dns2_tsigkey
```

### Paso 10: Sincronización de tiempo

**Importante:** TSIG requiere sincronización temporal.

```bash
# Si es necesario, sincronizar manualmente
sudo date --set HH:MM:SS
```

### Paso 11: Analizar con Wireshark

```bash
# Capturar tráfico DNS y observar las diferencias
# Con TSIG: Las transferencias incluyen firma digital
# Sin TSIG: Transferencias en texto plano
```

**¿Qué cambia con TSIG?**
- Las transferencias incluyen firmas digitales
- Solo servidores con la clave correcta pueden transferir
- Protección contra ataques man-in-the-middle

---

## Ejercicio 5: DNSSEC (DNS Security Extensions)

### ¿Qué se pide?
Implementar DNSSEC para firmar criptográficamente las zonas DNS, proporcionando autenticación de origen y verificación de integridad de datos.

### Objetivos:
1. Generar claves criptográficas (KSK y ZSK)
2. Firmar la zona con las claves
3. Configurar validación DNSSEC
4. Establecer cadena de confianza

---

### Conceptos clave:
- **KSK (Key-Signing Key)**: Firma las claves ZSK
- **ZSK (Zone-Signing Key)**: Firma los registros de la zona
- **RRSIG**: Firmas de los registros
- **DNSKEY**: Claves públicas
- **DS**: Delegation Signer (para cadena de confianza)

### Paso 1: Generar claves para la zona

```bash
cd /etc/bind/

# Generar KSK (Key-Signing Key)
sudo dnssec-keygen -a RSASHA256 -b 1024 -f KSK -n ZONE srdsi.lab.

# Generar ZSK (Zone-Signing Key)
sudo dnssec-keygen -a RSASHA256 -b 1024 -n ZONE srdsi.lab.
```

**Resultado:** Se generan archivos como:
- `Ksrdsi.lab.+008+39015.key` (clave pública)
- `Ksrdsi.lab.+008+39015.private` (clave privada)

### Paso 2: Agregar claves DNSKEY al archivo de zona

```bash
sudo nano /etc/bind/db.srdsi.lab
```

Agregar al final:
```dns
; Incrementar serial
20140403

; Incluir claves DNSSEC
$INCLUDE Ksrdsi.lab.+008+39015.key
$INCLUDE Ksrdsi.lab.+008+57448.key
```

### Paso 3: Firmar la zona

```bash
cd /etc/bind/
sudo dnssec-signzone -t -o srdsi.lab. -k Ksrdsi.lab.+008+39015.private \
    db.srdsi.lab Ksrdsi.lab.+008+57448.private
```

**Parámetros:**
- `-t`: Imprime estadísticas
- `-o`: Origen de la zona
- `-k`: Especifica la KSK
- El último parámetro es la ZSK

**Resultado:** Se crea `db.srdsi.lab.signed`

### Paso 4: Habilitar DNSSEC en BIND (por defecto está habilitado)

```bash
sudo nano /etc/bind/named.conf.options
```

Verificar que existe:
```bind
options {
    ...
    dnssec-enable yes;
    dnssec-validation yes;
    ...
};
```

### Paso 5: Actualizar configuración para usar archivo firmado

```bash
sudo nano /etc/bind/named.conf.local
```

Modificar:
```bind
zone "srdsi.lab" {
    type primary;
    file "/etc/bind/db.srdsi.lab.signed";
};
```

### Paso 6: Verificar y recargar

```bash
sudo named-checkconf
sudo rndc reload
```

### Paso 7: Probar DNSSEC

```bash
# Consulta con información DNSSEC
dig @localhost srdsi.lab +dnssec

# Consultar claves DNSKEY
dig @localhost srdsi.lab dnskey +dnssec

# Consultar firmas RRSIG
dig @localhost srdsi.lab rrsig +dnssec

# Consultar registros NSEC
dig @localhost srdsi.lab nsec +dnssec
```

### Configurar Trust Anchor (Validación)

### Paso 8: Crear trust anchor

```bash
sudo nano /etc/bind/trustanchor.key
```

Contenido (extraer de la KSK):
```bind
trust-anchors {
    srdsi.lab. static-key 257 3 8 "AwE...Wi5";
};
```

### Paso 9: Incluir trust anchor

```bash
sudo nano /etc/bind/named.conf
```

Agregar:
```bind
include "/etc/bind/trustanchor.key";
```

### Paso 10: Validar cadena de confianza

```bash
# Usar delv para validación
delv @localhost srdsi.lab -a /etc/bind/trustanchor.key \
    +root=srdsi.lab srdsi.lab SOA
```

### Mantenimiento de DNSSEC

### Re-firmar la zona (cuando se hacen cambios)

```bash
# Al modificar registros, re-firmar
sudo dnssec-signzone -o srdsi.lab. -f db.srdsi.lab.signed.new \
    db.srdsi.lab.signed

# Reemplazar archivo
sudo mv db.srdsi.lab.signed db.srdsi.lab.signed.bak
sudo mv db.srdsi.lab.signed.new db.srdsi.lab.signed

# Recargar
sudo rndc reload srdsi.lab
```

---

## Ejercicio 6: Delegación Segura

### ¿Qué se pide?
Crear una subzona delegada (`sub.srdsi.lab`) dentro del dominio principal e implementar una delegación segura usando DNSSEC, estableciendo una cadena de confianza entre la zona padre y la subzona.

### Objetivos:
1. Crear subzona delegada
2. Firmar la subzona con DNSSEC
3. Crear registro DS en la zona padre
4. Establecer cadena de confianza segura

---

### 6.1 Crear Subzona Delegada

### Paso 1: Configurar nueva zona

```bash
sudo nano /etc/bind/named.conf.local
```

Agregar:
```bind
// zona de resolución directa "sub.srdsi.lab"
zone "sub.srdsi.lab" {
    type primary;
    file "/etc/bind/db.sub.srdsi.lab";
};
```

### Paso 2: Agregar delegación en zona padre

```bash
sudo nano /etc/bind/db.srdsi.lab
```

Agregar:
```dns
; Incrementar serial
20140404

; Registros de delegación
sub IN NS dns1.srdsi.lab.
sub IN A  172.16.0.136
```

### Paso 3: Crear archivo de subzona

```bash
sudo nano /etc/bind/db.sub.srdsi.lab
```

Contenido:
```dns
$TTL 2h
@ IN SOA dns1.srdsi.lab. admin.srdsi.lab. (
    20140407 ; Serial
    604800   ; Refresh
    86400    ; Retry
    2419200  ; Expire
    604800 ) ; Negative Cache TTL
;
sub.srdsi.lab. IN NS dns1.srdsi.lab.
;
host1 IN A 172.16.145.45
host2 IN A 172.16.145.46
```

### Paso 4: Verificar y recargar

```bash
sudo named-checkzone sub.srdsi.lab /etc/bind/db.sub.srdsi.lab
sudo rndc reload
```

### Paso 5: Probar delegación

```bash
dig host1.sub.srdsi.lab
dig host2.sub.srdsi.lab
```

### 6.2 Firmar la Subzona

### Paso 6: Generar claves para subzona

```bash
cd /etc/bind/
sudo dnssec-keygen -a RSASHA256 -b 1024 -f KSK -n ZONE sub.srdsi.lab.
sudo dnssec-keygen -a RSASHA256 -b 1024 -n ZONE sub.srdsi.lab.
```

### Paso 7: Agregar claves al archivo de subzona

```bash
sudo nano /etc/bind/db.sub.srdsi.lab
```

Agregar:
```dns
; Incrementar serial
20140408

; Incluir claves
$INCLUDE Ksub.srdsi.lab.+008+20974.key
$INCLUDE Ksub.srdsi.lab.+008+57662.key
```

### Paso 8: Firmar la subzona

```bash
sudo dnssec-signzone -t -o sub.srdsi.lab. -k Ksub.srdsi.lab.+008+57662.private \
    db.sub.srdsi.lab Ksub.srdsi.lab.+008+20974.private
```

### Paso 9: Actualizar configuración

```bash
sudo nano /etc/bind/named.conf.local
```

Modificar:
```bind
zone "sub.srdsi.lab" {
    type primary;
    file "/etc/bind/db.sub.srdsi.lab.signed";
};
```

### 6.3 Crear Delegación Segura

### Paso 10: Obtener registro DS

El comando `dnssec-signzone` crea automáticamente `dsset-sub.srdsi.lab.`

```bash
cat dsset-sub.srdsi.lab.
```

### Paso 11: Agregar DS a zona padre

```bash
sudo nano /etc/bind/db.srdsi.lab
```

Agregar el contenido del archivo DS:
```dns
; Incrementar serial
20140409

; Registro DS para delegación segura
sub.srdsi.lab. IN DS 7662 8 2 B67DBD0A5E467E2CD2A1F4D60169D16A5BC841...
```

### Paso 12: Re-firmar zona padre

```bash
sudo dnssec-signzone -o srdsi.lab. -k Ksrdsi.lab.+008+39015.private \
    db.srdsi.lab Ksrdsi.lab.+008+57448.private
sudo rndc reload
```

### Paso 13: Verificar delegación segura

```bash
# Verificar registros DS en zona padre
dig @localhost srdsi.lab DS +dnssec

# Verificar subzona firmada
dig @localhost sub.srdsi.lab +dnssec

# Validar cadena de confianza
delv @localhost host1.sub.srdsi.lab -a /etc/bind/trustanchor.key
```

---

## Ejercicio 7: NSEC3

### ¿Qué se pide?
Implementar NSEC3 en la zona de resolución inversa (`0.16.172.in-addr.arpa`) para proporcionar autenticación de respuestas negativas con protección adicional contra ataques de enumeración de zona.

### Objetivos del ejercicio:
1. Comprender las diferencias entre NSEC y NSEC3
2. Generar claves DNSSEC para la zona inversa
3. Firmar la zona usando NSEC3 en lugar de NSEC estándar
4. Configurar parámetros NSEC3 con salt aleatorio
5. Verificar el funcionamiento de NSEC3

---

### ¿Qué es NSEC3 y por qué usarlo?

### **NSEC vs NSEC3:**

**NSEC (Problema):**
```dns
; NSEC expone todos los nombres en orden alfabético
a.srdsi.lab NSEC dns1.srdsi.lab A NS
dns1.srdsi.lab NSEC www.srdsi.lab A
www.srdsi.lab NSEC z.srdsi.lab A
```
**Problema:** ¡Un atacante puede enumerar TODOS los nombres de la zona!

**NSEC3 (Solución):**
```dns
; NSEC3 usa hashes - no se pueden leer directamente
A12B34CD NSEC3 1 0 10 SALT F34E56AB A
F34E56AB NSEC3 1 0 10 SALT A12B34CD A
```
**Ventaja:** Los nombres están hasheados, no se puede hacer enumeración fácil.

### **¿Qué protege NSEC3?**
- **Enumeración de zona:** No puedes obtener lista de todos los nombres
- **Ataques de diccionario:** El salt hace más difíciles los ataques
- **Privacidad:** Los nombres no se exponen en texto claro

---

### Paso 1: Generar claves DNSSEC para la zona inversa

```bash
cd /etc/bind/

# Generar KSK (Key-Signing Key) para zona inversa
sudo dnssec-keygen -a RSASHA256 -b 1024 -f KSK -n ZONE 0.16.172.in-addr.arpa

# Generar ZSK (Zone-Signing Key) para zona inversa
sudo dnssec-keygen -a RSASHA256 -b 1024 -n ZONE 0.16.172.in-addr.arpa
```

**Resultado:** Se generan archivos como:
```
K0.16.172.in-addr.arpa.+008+07525.key
K0.16.172.in-addr.arpa.+008+07525.private
K0.16.172.in-addr.arpa.+008+29530.key  
K0.16.172.in-addr.arpa.+008+29530.private
```

### Paso 2: Identificar automáticamente las claves generadas

```bash
# Automatizar identificación de claves (para facilitar el proceso)
KSK_KEY=$(grep -l "DNSKEY 257" K0.16.172.in-addr.arpa.+008+*.key)
echo "KSK Key: $KSK_KEY"

ZSK_KEY=$(grep -l "DNSKEY 256" K0.16.172.in-addr.arpa.+008+*.key)
echo "ZSK Key: $ZSK_KEY"
```

**Explicación:**
- **Flag 257**: Identifica la KSK (Key-Signing Key)
- **Flag 256**: Identifica la ZSK (Zone-Signing Key)

### Paso 3: Incluir las claves en el archivo de zona inversa

```bash
# Agregar automáticamente las claves al archivo de zona
sudo bash -c "cat >> /etc/bind/db.0.16.172 << EOF
; Incluir claves DNSSEC
\$INCLUDE $KSK_KEY
\$INCLUDE $ZSK_KEY
EOF"
```

**Resultado:** El archivo `/etc/bind/db.0.16.172` ahora incluye:
```dns
$TTL 604800
@ IN SOA dns1.srdsi.lab. admin.srdsi.lab. (
    20140401 ; Serial
    604800   ; Refresh
    86400    ; Retry
    2419200  ; Expire
    604800 ) ; Negative Cache TTL
;
@ IN NS dns1.srdsi.lab.
;
136 IN PTR dns1.srdsi.lab.
138 IN PTR host1.srdsi.lab.
;
; Incluir claves DNSSEC
$INCLUDE K0.16.172.in-addr.arpa.+008+07525.key
$INCLUDE K0.16.172.in-addr.arpa.+008+29530.key
```

### Paso 4: Verificar sintaxis del archivo de zona

```bash
sudo named-checkzone 0.16.172.in-addr.arpa /etc/bind/db.0.16.172
```

**Resultado esperado:**
```
zone 0.16.172.in-addr.arpa/IN: loaded serial 20140401
OK
```

### Paso 5: Generar salt aleatorio para NSEC3

```bash
# Generar salt aleatorio de 16 caracteres hexadecimales
SALT=$(head -c 16 /dev/random | sha1sum | cut -b 1-16)
echo "Salt generado: $SALT"
```

**¿Qué es el salt?**
- **Valor aleatorio** que se añade al hash
- **Previene ataques de diccionario** precomputados
- **Único por zona** y se puede cambiar periódicamente

**Ejemplo de salt:** `dee4cb0715103b7`

### Paso 6: Firmar la zona con NSEC3

```bash
cd /etc/bind/
sudo dnssec-signzone -A -3 $SALT -t -S -o 0.16.172.in-addr.arpa \
    -f db.0.16.172.signed db.0.16.172
```

**Parámetros explicados:**
- **`-A`**: Incluye todas las firmas automáticamente
- **`-3`**: Activa NSEC3 en lugar de NSEC estándar
- **`-t`**: Imprime estadísticas del proceso de firma
- **`-S`**: Genera y firma claves automáticamente
- **`-o`**: Especifica el origen/nombre de la zona
- **`-f`**: Nombre del archivo de salida firmado

**Resultado:** Se crea `db.0.16.172.signed` con registros NSEC3

### Paso 7: Actualizar configuración de BIND para usar archivo firmado

```bash
# Cambiar la configuración para usar el archivo firmado
sudo sed -i 's|file "/etc/bind/db.0.16.172";|file "/etc/bind/db.0.16.172.signed";|' \
    /etc/bind/named.conf.local
```

**Verificar el cambio:**
```bash
grep "db.0.16.172" /etc/bind/named.conf.local
```

**Resultado esperado:**
```
file "/etc/bind/db.0.16.172.signed";
```

### Paso 8: Configurar opciones DNSSEC en BIND

```bash
# Configurar named.conf.options para validación DNSSEC
sudo bash -c 'cat > /etc/bind/named.conf.options << EOF
options {
    directory "/var/cache/bind";
    
    dnssec-validation yes;
    
    listen-on { any; };
    listen-on-v6 { any; };
    
    allow-query { any; };
};
EOF'
```

**Explicación de opciones:**
- **`dnssec-validation yes`**: Habilita validación DNSSEC
- **`listen-on { any; }`**: Escucha en todas las interfaces IPv4
- **`allow-query { any; }`**: Permite consultas desde cualquier IP

### Paso 9: Verificar configuración y reiniciar BIND

```bash
# Verificar que no hay errores de sintaxis
sudo named-checkconf

# Reiniciar BIND para aplicar cambios
sudo systemctl restart bind9

# Verificar que el servicio está funcionando
sudo systemctl status bind9
```

### Paso 10: Configurar trust anchor (opcional para validación completa)

```bash
# Crear trust anchor con la clave KSK para validación local
# Primero extraer la clave pública de la KSK
KSK_CONTENT=$(cat $KSK_KEY | grep -v '^;' | awk '{print $7}')

sudo bash -c "cat > /etc/bind/trust-anchor.key << EOF
trusted-keys {
    0.16.172.in-addr.arpa. 257 3 8 \"$KSK_CONTENT\";
};
EOF"

# Incluir en la configuración principal
sudo bash -c 'cat >> /etc/bind/named.conf << EOF
include "/etc/bind/trust-anchor.key";
EOF'
```

---

### Verificación y Pruebas de NSEC3

### Paso 11: Verificar parámetros NSEC3

```bash
# Consultar parámetros NSEC3PARAM
dig @127.0.0.1 0.16.172.in-addr.arpa NSEC3PARAM +dnssec
```

**Resultado esperado:**
```dns
;; ANSWER SECTION:
0.16.172.in-addr.arpa. 7200 IN NSEC3PARAM 1 0 10 FE6BFD0A40C1181B

;; ADDITIONAL SECTION:
0.16.172.in-addr.arpa. 7200 IN RRSIG NSEC3PARAM ...
```

**Interpretación:**
- **`1`**: Algoritmo SHA-1
- **`0`**: Flags (0 = sin opt-out)
- **`10`**: Iteraciones de hash
- **`FE6BFD0A40C1181B`**: Salt usado

### Paso 12: Consulta de resolución inversa exitosa

```bash
# Consultar IP existente con DNSSEC
dig @127.0.0.1 -x 172.16.0.136 +dnssec
```

**Resultado esperado:**
```dns
;; ANSWER SECTION:
136.0.16.172.in-addr.arpa. 604800 IN PTR dns1.srdsi.lab.
136.0.16.172.in-addr.arpa. 604800 IN RRSIG PTR 8 4 604800 ...

;; AUTHORITY SECTION:
0.16.172.in-addr.arpa. 604800 IN SOA dns1.srdsi.lab. admin.srdsi.lab. ...
0.16.172.in-addr.arpa. 604800 IN RRSIG SOA 8 3 604800 ...
```

**¿Qué vemos?**
- **PTR**: Respuesta normal de resolución inversa
- **RRSIG**: Firma DNSSEC que valida la respuesta

### Paso 13: Consulta de IP inexistente (respuesta negativa con NSEC3)

```bash
# Consultar IP que NO existe para ver NSEC3
dig @127.0.0.1 -x 172.16.0.150 +dnssec
```

**Resultado esperado:**
```dns
;; status: NXDOMAIN

;; AUTHORITY SECTION:
0.16.172.in-addr.arpa. 604800 IN SOA dns1.srdsi.lab. admin.srdsi.lab. ...
0.16.172.in-addr.arpa. 604800 IN RRSIG SOA 8 3 604800 ...

;; Registros NSEC3 que prueban la no-existencia
A1B2C3D4E5F6G7H8. 604800 IN NSEC3 1 0 10 SALT F9E8D7C6B5A4 PTR
A1B2C3D4E5F6G7H8. 604800 IN RRSIG NSEC3 8 3 604800 ...
```

**¿Qué significa esto?**
- **NXDOMAIN**: La IP no existe
- **NSEC3**: Prueba criptográfica de que no existe
- **Hashes**: Los nombres están hasheados, no legibles

### Paso 14: Consultar registros NSEC3 directamente

```bash
# Ver todos los registros NSEC3 en formato legible
dig @127.0.0.1 0.16.172.in-addr.arpa NSEC3 +dnssec +multi
```

**Resultado esperado:**
```dns
;; ANSWER SECTION:
A1B2C3D4E5F6G7H8.0.16.172.in-addr.arpa. (
    604800 IN NSEC3 1 0 10 DEE4CB0715103B7 (
        F9E8D7C6B5A43210 PTR RRSIG )
    604800 IN RRSIG NSEC3 8 3 604800 (
        20240520120000 20240420120000 12345
        0.16.172.in-addr.arpa. [firma_digital] )
```

---

### Diferencias Visuales: NSEC vs NSEC3

### **Con NSEC (inseguro para enumeración):**
```dns
; Nombres visibles en orden alfabético
136.0.16.172.in-addr.arpa NSEC 138.0.16.172.in-addr.arpa PTR
138.0.16.172.in-addr.arpa NSEC 140.0.16.172.in-addr.arpa PTR
```
**Problema:** ¡Un atacante ve que existen .136, .138 y puede deducir que .137, .139 no existen!

### **Con NSEC3 (seguro contra enumeración):**
```dns
; Nombres hasheados - no legibles
A1B2C3.in-addr.arpa NSEC3 1 0 10 SALT F9E8D7 PTR
F9E8D7.in-addr.arpa NSEC3 1 0 10 SALT A1B2C3 PTR
```
**Ventaja:** ¡Un atacante NO puede saber qué IPs existen sin calcular millones de hashes!

### **Comparación práctica:**

| Aspecto | NSEC | NSEC3 |
|---------|------|-------|
| **Enumeración** | ❌ Vulnerable | ✅ Protegido |
| **Nombres** | 📖 Texto claro | 🔒 Hasheados |
| **Salt** | ❌ No usa | ✅ Usa salt |
| **Rendimiento** | ⚡ Más rápido | 🐌 Algo más lento |
| **Seguridad** | 🔓 Básica | 🛡️ Mejorada |
| **Compatibilidad** | ✅ Universal | ✅ Moderna |

---

### Comandos de Diagnóstico y Monitoreo

### **Verificar funcionamiento general:**
```bash
# Estado del servidor
sudo rndc status

# Logs de DNSSEC
sudo tail -f /var/log/daemon.log | grep -i dnssec

# Verificar zonas cargadas
sudo rndc status | grep "zones"
```

### **Análisis de registros NSEC3:**
```bash
# Ver estructura de NSEC3
dig @localhost 0.16.172.in-addr.arpa NSEC3 +short

# Ver parámetros NSEC3
dig @localhost 0.16.172.in-addr.arpa NSEC3PARAM +short

# Verificar firmas
dig @localhost 0.16.172.in-addr.arpa RRSIG +short
```

### **Validación con herramientas externas:**
```bash
# Usar delv para validación completa (si configuraste trust anchor)
delv @localhost -x 172.16.0.136 +root=0.16.172.in-addr.arpa

# Verificar cadena de firma
dig @localhost -x 172.16.0.136 +trace +dnssec
```

---

### Mantenimiento y Consideraciones

### **Rotación de salt:**
```bash
# Generar nuevo salt periódicamente
NEW_SALT=$(head -c 16 /dev/random | sha1sum | cut -b 1-16)

# Re-firmar con nuevo salt
sudo dnssec-signzone -A -3 $NEW_SALT -t -S -o 0.16.172.in-addr.arpa \
    -f db.0.16.172.signed.new db.0.16.172

# Actualizar archivo
sudo mv db.0.16.172.signed db.0.16.172.signed.bak
sudo mv db.0.16.172.signed.new db.0.16.172.signed
sudo rndc reload
```

### **Re-firma periódica:**
```bash
# Las firmas DNSSEC expiran, re-firmar regularmente
sudo dnssec-signzone -A -3 $SALT -t -S -o 0.16.172.in-addr.arpa \
    -f db.0.16.172.signed.new db.0.16.172.signed
```

### **Monitoreo de expiración:**
```bash
# Verificar fechas de expiración de firmas  
dig @localhost 0.16.172.in-addr.arpa RRSIG | grep -E '[0-9]{14}'
```

---

### ¿Por qué NSEC3 en zona inversa?

### **Especialmente importante en zonas inversas:**

**Sin NSEC3:**
```bash
# Un atacante puede enumerar todas las IPs en uso
dig example.com NSEC
# Resultado: Descubre 192.168.1.10, 192.168.1.15, 192.168.1.20...
# ¡Mapeo completo de la red!
```

**Con NSEC3:**
```bash
# Un atacante ve solo hashes
dig example.com NSEC3  
# Resultado: A1B2C3, F9E8D7, 3C4D5E...
# ¡No puede deducir qué IPs están en uso!
```

### **Beneficios en seguridad de red:**
- **Oculta topología** de red interna
- **Previene reconnaissance** automático
- **Dificulta ataques dirigidos** a IPs específicas
- **Cumple mejores prácticas** de seguridad DNS

---

### Resumen del Ejercicio 7

**¿Qué hemos logrado?**

1. ✅ **Implementado NSEC3** en zona de resolución inversa
2. ✅ **Protegido contra enumeración** de zona  
3. ✅ **Configurado salt aleatorio** para mayor seguridad
4. ✅ **Verificado funcionamiento** de respuestas negativas seguras
5. ✅ **Establecido validación DNSSEC** completa

**NSEC3 = "Pruebo que algo no existe, pero sin revelar qué más sí existe"**

La zona inversa ahora proporciona **respuestas negativas autenticadas** sin exponer la estructura de la red, mejorando significativamente la seguridad DNS.

---

## Comandos de Utilidad y Verificación

### Gestión del servidor BIND

```bash
# Estado del servicio
sudo systemctl status bind9
sudo systemctl start|stop|restart bind9

# Control de BIND
sudo rndc status
sudo rndc reload [zona]
sudo rndc flush         # Limpiar caché
sudo rndc dumpdb -cache # Volcar caché a archivo
```

### Verificación de configuración

```bash
# Verificar configuración general
sudo named-checkconf

# Verificar zona específica
sudo named-checkzone zona archivo_zona

# Ver logs
sudo tail -f /var/log/syslog
sudo tail -f /var/log/daemon.log
```

### Herramientas de consulta

```bash
# Consultas DNS básicas
dig @servidor nombre [tipo]
dig @servidor nombre +short
dig @servidor nombre +trace

# Consultas inversas
dig @servidor -x IP

# Consultas DNSSEC
dig @servidor nombre +dnssec
dig @servidor nombre DNSKEY
dig @servidor nombre RRSIG

# Transferencia de zona
dig @servidor zona AXFR
```

### Archivos importantes

```
/etc/bind/named.conf           # Configuración principal
/etc/bind/named.conf.local     # Zonas locales
/etc/bind/named.conf.options   # Opciones del servidor
/etc/bind/db.*                 # Archivos de zona
/var/log/daemon.log           # Logs de BIND
/var/cache/bind/              # Caché y archivos temporales
```

---

## Resumen de Progresión

1. **Ejercicio 1**: Servidor DNS básico (caché)
2. **Ejercicio 2**: Servidor autoritativo con zonas propias
3. **Ejercicio 3**: Redundancia con servidor secundario
4. **Ejercicio 4**: Seguridad en transferencias con TSIG
5. **Ejercicio 5**: Autenticación de datos con DNSSEC
6. **Ejercicio 6**: Delegación segura de subzonas

Cada ejercicio construye sobre el anterior, creando un sistema DNS completo, seguro y redundante.

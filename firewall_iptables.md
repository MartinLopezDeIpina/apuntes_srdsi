# Guía Completa de Parámetros iptables

## Comandos Principales

### `-A` (Append)
**Función:** Añade una regla al **final** de la cadena especificada.

**Sintaxis:** `-A [CADENA]`

**Ejemplo:**
```bash
$IPTABLES -A INPUT -j ACCEPT
# Añade una regla al final de la cadena INPUT
```

---

## Cadenas (Chains)

### `INPUT`
**Función:** Procesa paquetes **dirigidos AL firewall** (destino = firewall).

**Cuándo se usa:** Cuando el firewall es el **destino final** del paquete.

**Ejemplo del laboratorio:**
```bash
$IPTABLES -A INPUT -i $LAN_IFACE -s $LAN_IP -d $LAN_IFACE_IP \
    -p icmp -m icmp --icmp-type echo-request \
    -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
*PC_LAN hace ping **AL firewall** (192.168.1.1)*

### `OUTPUT`
**Función:** Procesa paquetes **generados POR el firewall** (origen = firewall).

**Cuándo se usa:** Cuando el firewall **genera** o **responde** paquetes.

**Ejemplo del laboratorio:**
```bash
$IPTABLES -A OUTPUT -o $LAN_IFACE -d $LAN_IP -s $LAN_IFACE_IP \
    -p icmp -m icmp --icmp-type echo-reply \
    -m state --state RELATED,ESTABLISHED -j ACCEPT
```
*Firewall responde el ping **DE VUELTA** al PC_LAN*

### `FORWARD`
**Función:** Procesa paquetes que **pasan A TRAVÉS** del firewall (ni origen ni destino = firewall).

**Cuándo se usa:** Cuando el firewall actúa como **router/gateway** entre redes.

**Ejemplo del laboratorio:**
```bash
$IPTABLES -A FORWARD -i $LAN_IFACE -s $LAN_IP \
    -o $DMZ_IFACE -d $DMZ_WEB_IP \
    -p tcp -m multiport --dport 80,443 -j ACCEPT
```
*PC_LAN navega al servidor web en DMZ **A TRAVÉS** del firewall*

---

## Parámetros de Interfaz

### `-i` (Input Interface)
**Función:** Especifica la **interfaz de entrada** del paquete.

**Solo válido en:** INPUT y FORWARD

**Ejemplo:**
```bash
-i $LAN_IFACE    # Paquetes que entran por eth1 (LAN)
-i $DMZ_IFACE    # Paquetes que entran por eth2 (DMZ)
```

### `-o` (Output Interface)
**Función:** Especifica la **interfaz de salida** del paquete.

**Solo válido en:** OUTPUT y FORWARD

**Ejemplo:**
```bash
-o $DMZ_IFACE    # Paquetes que salen por eth2 (hacia DMZ)
-o $LAN_IFACE    # Paquetes que salen por eth1 (hacia LAN)
```

---

## Parámetros de Direcciones

### `-s` (Source)
**Función:** Especifica la **dirección IP de origen**.

**Ejemplo:**
```bash
-s $LAN_IP           # Origen: 192.168.1.0/24 (cualquier PC de LAN)
-s $DMZ_WEB_IP       # Origen: 10.10.10.30 (servidor web específico)
-s $LAN_IFACE_IP     # Origen: 192.168.1.1 (IP del firewall en LAN)
```

### `-d` (Destination)
**Función:** Especifica la **dirección IP de destino**.

**Ejemplo:**
```bash
-d $LAN_IFACE_IP     # Destino: 192.168.1.1 (firewall)
-d $DMZ_WEB_IP       # Destino: 10.10.10.30 (servidor web)
-d $LAN_IP           # Destino: 192.168.1.0/24 (cualquier PC de LAN)
```

---

## Parámetros de Protocolo

### `-p` (Protocol)
**Función:** Especifica el **protocolo** de red.

**Valores comunes:** `tcp`, `udp`, `icmp`, `all`

**Ejemplo:**
```bash
-p icmp    # Protocolo ICMP (ping)
-p tcp     # Protocolo TCP (web, ssh, etc.)
```

---

## Módulos de Coincidencia

### `-m` (Match)
**Función:** Carga un **módulo específico** para opciones avanzadas.

#### Módulo `icmp`
**Función:** Opciones específicas para protocolo ICMP.

**Ejemplo:**
```bash
-m icmp --icmp-type echo-request    # Solo pings (tipo 8)
-m icmp --icmp-type echo-reply      # Solo respuestas ping (tipo 0)
```

#### Módulo `state`
**Función:** Seguimiento del **estado de la conexión**.

**Estados:**
- `NEW`: Nueva conexión
- `ESTABLISHED`: Conexión establecida
- `RELATED`: Conexión relacionada con otra existente

**Ejemplo:**
```bash
-m state --state NEW,ESTABLISHED,RELATED    # Permite nuevas + establecidas
-m state --state ESTABLISHED,RELATED        # Solo respuestas a existentes
```

#### Módulo `multiport`
**Función:** Especificar **múltiples puertos** en una sola regla.

**Ejemplo:**
```bash
-m multiport --dport 80,443    # Puertos destino 80 Y 443 (HTTP + HTTPS)
-m multiport --sport 80,443    # Puertos origen 80 Y 443
```

---

## Acciones

### `-j` (Jump)
**Función:** Especifica la **acción** a realizar con el paquete.

**Valores comunes:** `ACCEPT`, `DROP`, `REJECT`

**Ejemplo:**
```bash
-j ACCEPT    # Permitir el paquete
-j DROP      # Descartar silenciosamente
-j REJECT    # Rechazar con respuesta de error
```

---

## Ejemplos Completos Analizados

### Ejemplo 1: Ping LAN → Firewall (INPUT)
```bash
$IPTABLES -A INPUT -i $LAN_IFACE -s $LAN_IP -d $LAN_IFACE_IP \
    -p icmp -m icmp --icmp-type echo-request \
    -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

**Desglose:**
- `-A INPUT`: Añadir a cadena INPUT
- `-i $LAN_IFACE`: Entra por eth1 (LAN)
- `-s $LAN_IP`: Origen 192.168.1.0/24
- `-d $LAN_IFACE_IP`: Destino 192.168.1.1 (firewall)
- `-p icmp`: Protocolo ICMP
- `-m icmp --icmp-type echo-request`: Solo pings
- `-m state --state NEW,ESTABLISHED,RELATED`: Estados permitidos
- `-j ACCEPT`: Permitir

**Traducción:** *"Permitir que cualquier PC de la LAN haga ping al firewall"*

### Ejemplo 2: Respuesta Firewall → LAN (OUTPUT)
```bash
$IPTABLES -A OUTPUT -o $LAN_IFACE -d $LAN_IP -s $LAN_IFACE_IP \
    -p icmp -m icmp --icmp-type echo-reply \
    -m state --state RELATED,ESTABLISHED -j ACCEPT
```

**Desglose:**
- `-A OUTPUT`: Añadir a cadena OUTPUT
- `-o $LAN_IFACE`: Sale por eth1 (hacia LAN)
- `-d $LAN_IP`: Destino 192.168.1.0/24
- `-s $LAN_IFACE_IP`: Origen 192.168.1.1 (firewall)
- `-p icmp`: Protocolo ICMP
- `-m icmp --icmp-type echo-reply`: Solo respuestas
- `-m state --state RELATED,ESTABLISHED`: Solo respuestas a conexiones existentes
- `-j ACCEPT`: Permitir

**Traducción:** *"Permitir que el firewall responda los pings a la LAN"*

### Ejemplo 3: Navegación LAN → DMZ (FORWARD)
```bash
$IPTABLES -A FORWARD -i $LAN_IFACE -s $LAN_IP \
    -o $DMZ_IFACE -d $DMZ_WEB_IP \
    -p tcp -m multiport --dport 80,443 -j ACCEPT
```

**Desglose:**
- `-A FORWARD`: Añadir a cadena FORWARD
- `-i $LAN_IFACE`: Entra por eth1 (LAN)
- `-s $LAN_IP`: Origen 192.168.1.0/24
- `-o $DMZ_IFACE`: Sale por eth2 (DMZ)
- `-d $DMZ_WEB_IP`: Destino 10.10.10.30 (servidor web)
- `-p tcp`: Protocolo TCP
- `-m multiport --dport 80,443`: Puertos HTTP y HTTPS
- `-j ACCEPT`: Permitir

**Traducción:** *"Permitir que los PCs de la LAN naveguen al servidor web de la DMZ"*

### Ejemplo 4: Respuesta DMZ → LAN (FORWARD)
```bash
$IPTABLES -A FORWARD -o $LAN_IFACE -d $LAN_IP \
    -i $DMZ_IFACE -s $DMZ_WEB_IP \
    -p tcp -m multiport --sport 80,443 -j ACCEPT
```

**Desglose:**
- `-A FORWARD`: Añadir a cadena FORWARD
- `-o $LAN_IFACE`: Sale por eth1 (hacia LAN)
- `-d $LAN_IP`: Destino 192.168.1.0/24
- `-i $DMZ_IFACE`: Entra por eth2 (DMZ)
- `-s $DMZ_WEB_IP`: Origen 10.10.10.30 (servidor web)
- `-p tcp`: Protocolo TCP
- `-m multiport --sport 80,443`: Puertos origen HTTP y HTTPS
- `-j ACCEPT`: Permitir

**Traducción:** *"Permitir que el servidor web de la DMZ responda a los PCs de la LAN"*

---

## Resumen de Flujos

### Flujo 1: Ping al Firewall
```
PC_LAN → [INPUT] → Firewall → [OUTPUT] → PC_LAN
```

### Flujo 2: Navegación a través del Firewall
```
PC_LAN → [FORWARD] → PC_DMZ → [FORWARD] → PC_LAN
```

### Diferencias Clave
- **INPUT/OUTPUT**: El firewall es **participante** en la comunicación
- **FORWARD**: El firewall es **intermediario** en la comunicación

---

## Comandos de Verificación

```bash
# Ver reglas actuales
iptables -L -n -v

# Ver reglas con números de línea
iptables -L -n -v --line-numbers

# Ver solo cadena específica
iptables -L INPUT -n -v
iptables -L FORWARD -n -v
iptables -L OUTPUT -n -v
```
# PREROUTING vs POSTROUTING - Resumen Conciso

## Concepto Clave

**NAT** modifica direcciones IP de los paquetes en dos momentos diferentes:

| **PREROUTING** | **POSTROUTING** |
|----------------|-----------------|
| **ENTRADA** de paquetes | **SALIDA** de paquetes |
| Modifica **DESTINO** (DNAT) | Modifica **ORIGEN** (SNAT) |
| "¿A dónde va realmente?" | "¿De dónde viene realmente?" |

---

## PREROUTING (Destination NAT)

### ¿Cuándo se ejecuta?
```
Internet ──→ [PREROUTING] ──→ Decisión routing ──→ LAN/DMZ
```
**Momento:** Justo **DESPUÉS** de recibir el paquete, **ANTES** de decidir su ruta.

### Ejemplo Práctico: Acceso público al servidor web DMZ
```bash
# Regla PREROUTING - DNAT
$IPTABLES -t nat -A PREROUTING -i $INTERNET_IFACE \
    -d $INTERNET_IFACE_IP -p tcp --dport 80 \
    -j DNAT --to-destination $DMZ_WEB_IP:80
```

### Flujo:
```
1. Internet envía:     Cliente → 203.0.113.10:80 (IP pública)
2. PREROUTING cambia:  Cliente → 10.10.10.30:80  (IP privada DMZ)
3. Firewall enruta:    Paquete va hacia DMZ
```

### Propósito:
**Permitir acceso desde Internet a un servidor interno** usando la IP pública del firewall.

---

## POSTROUTING (Source NAT)

### ¿Cuándo se ejecuta?
```
LAN/DMZ ──→ Decisión routing ──→ [POSTROUTING] ──→ Internet
```
**Momento:** Justo **ANTES** de enviar el paquete, **DESPUÉS** de decidir su ruta.

### Ejemplo Práctico: LAN accede a Internet
```bash
# Regla POSTROUTING - SNAT
$IPTABLES -t nat -A POSTROUTING -s $LAN_IP \
    -o $INTERNET_IFACE \
    -j SNAT --to-source $INTERNET_IFACE_IP
```

### Flujo:
```
1. LAN envía:          192.168.1.10 → Servidor_Web
2. POSTROUTING cambia: 203.0.113.10 → Servidor_Web (IP pública)
3. Internet ve:        Solo la IP pública del firewall
```

### Propósito:
**Ocultar IPs privadas internas** y permitir navegación a Internet.

---

## Casos de Uso Típicos

### PREROUTING (DNAT)
- ✅ **Port Forwarding**: Redirigir puertos públicos a servidores internos
- ✅ **Publicar servicios**: Web, FTP, SSH desde DMZ hacia Internet
- ✅ **Load Balancing**: Distribuir conexiones entre varios servidores

**Ejemplo típico:**
```bash
# Publicar servidor web DMZ en puerto 80 público
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 10.10.10.30
```

### POSTROUTING (SNAT)
- ✅ **Masquerading**: LAN navega por Internet
- ✅ **Ocultación de topología**: Ocultar IPs internas
- ✅ **Salida controlada**: Una sola IP pública para toda la organización

**Ejemplo típico:**
```bash
# Toda la LAN sale por la IP pública del firewall
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

---

## Flujo Completo: Navegación LAN → Internet

### 1. Paquete de ida (POSTROUTING):
```
PC_LAN (192.168.1.10) ──SNAT──→ Internet (8.8.8.8)
Internet ve: 203.0.113.10 → 8.8.8.8
```

### 2. Paquete de vuelta (automático):
```
Internet (8.8.8.8) ──des-NAT──→ PC_LAN (192.168.1.10)
PC recibe: 8.8.8.8 → 192.168.1.10
```

---

## Flujo Completo: Internet → Servidor DMZ

### 1. Paquete de entrada (PREROUTING):
```
Internet ──DNAT──→ DMZ_Server
Cliente: 1.2.3.4 → 203.0.113.10:80 se convierte en 1.2.3.4 → 10.10.10.30:80
```

### 2. Paquete de respuesta (automático):
```
DMZ_Server ──des-NAT──→ Internet
Servidor: 10.10.10.30:80 → 1.2.3.4 se convierte en 203.0.113.10:80 → 1.2.3.4
```

---

## Comandos de Verificación

```bash
# Ver reglas NAT
iptables -t nat -L -n -v

# Ver traducciones activas
conntrack -L

# Ver solo PREROUTING
iptables -t nat -L PREROUTING -n -v

# Ver solo POSTROUTING  
iptables -t nat -L POSTROUTING -n -v
```

---

## Resumen Visual

```
         ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
Internet │             │ LAN/DMZ │             │ Internet │             │ LAN/DMZ
  ──────→│ PREROUTING  │────────→│   ROUTING   │────────→│POSTROUTING  │────────→
         │   (DNAT)    │         │  DECISION   │         │   (SNAT)    │
         └─────────────┘         └─────────────┘         └─────────────┘
         Cambia DESTINO           Decide ruta            Cambia ORIGEN
```

**Clave:** PREROUTING modifica **hacia dónde** va el paquete, POSTROUTING modifica **de dónde** viene.

# NAT: Modificación vs Restricción

## Concepto Clave

**NAT siempre MODIFICA direcciones IP, pero el EFECTO depende del destino**

---

## POSTROUTING (SNAT)

### ¿Qué hace?
**SIEMPRE modifica IP origen, NUNCA restringe**

```bash
$IPTABLES -t nat -A POSTROUTING -s $LAN_IP -o $DMZ_IFACE -j SNAT --to-source $FW_IP
```

### Flujo:
```
PC_LAN (192.168.1.10) ──SNAT──→ FW (10.10.10.1) ──→ DMZ_Server
```

### Resultado:
- ✅ **Tráfico fluye normalmente**
- ✅ **Solo cambia identidad del emisor**
- ❌ **NO bloquea ni restringe**

**Propósito:** Ocultar/modificar quién envía

---

## PREROUTING (DNAT)

### ¿Qué hace?
**SIEMPRE modifica IP destino, pero el EFECTO depende del nuevo destino**

### Caso 1: DNAT → Servidor Real (PERMITE)
```bash
$IPTABLES -t nat -A PREROUTING -i eth0 -d 203.0.113.10 -p tcp --dport 80 \
    -j DNAT --to-destination 10.10.10.30:80
```

**Flujo:**
```
Internet → (203.0.113.10:80) ──DNAT──→ (10.10.10.30:80) → DMZ_Server
```

**Resultado:** ✅ **PERMITE acceso** (Port Forwarding)

### Caso 2: DNAT → Agujero Negro (BLOQUEA)
```bash
$IPTABLES -t nat -A PREROUTING -i eth0 -d 10.10.10.30 \
    -j DNAT --to-destination 127.0.0.1
```

**Flujo:**
```
Internet → (10.10.10.30) ──DNAT──→ (127.0.0.1) → Connection refused
```

**Resultado:** 🚫 **BLOQUEA acceso** (Redirección a loopback)

---

## Tabla Resumen

| **Cadena** | **Modifica** | **¿Restringe?** | **Depende de** |
|------------|--------------|-----------------|----------------|
| **POSTROUTING** | IP origen | ❌ Nunca | - |
| **PREROUTING** | IP destino | ⚠️ Depende | **¿Hacia dónde?** |

---

## Ejemplos del Laboratorio

### POSTROUTING - Solo Modificación:
```
# Oculta LAN hacia DMZ
LAN → SNAT → Firewall_IP → DMZ  ✅ Solo cambia apariencia
```

### PREROUTING - Permite:
```
# Acceso público a DMZ
Internet → IP_Pública → DNAT → DMZ_Real  ✅ Port forwarding
```

### PREROUTING - Bloquea:
```
# Acceso directo bloqueado
Internet → IP_Privada → DNAT → Loopback  ❌ Redirección a agujero negro
```

---

## Regla de Oro

### POSTROUTING:
**Siempre modifica, nunca restringe**

### PREROUTING:
**Siempre modifica, pero el efecto depende del destino:**
- **DNAT → Servidor real** = Permite
- **DNAT → Loopback/inexistente** = Bloquea

**En resumen:** La técnica es **modificación**, el efecto es **permitir o bloquear** según hacia dónde apunte.

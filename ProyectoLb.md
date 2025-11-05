
# 🧬 **Infraestructura Elástica de OCI con Control de Acceso Granular**

## Configuración de Red, Balanceador de Carga, Autoescalado, DNS Privado y File System

### 🎯 **Objetivo**

- 🛠️ VCN con subredes públicas, listas de seguridad, gateway de Internet, tablas de rutas.
    
- ⚖️ Balanceador de carga conectado a varias instancias backend.
    
- 📈 Grupo de instancias con autoescalado basado en métricas.
    
- 🌐 Zona DNS privada y un registro tipo A para resolución interna.
    
- 💾 File System con acceso controlado mediante NFS desde dos VMs diferentes.
    
<img width="1959" height="867" alt="image" src="https://github.com/user-attachments/assets/4af45118-c5cd-4bfa-bfb7-c01ceeca0772" />

---

## 1️⃣ Crear Red Virtual y Componentes (VCN)

### 1.1 Crear VCN

1. Desde el menú principal, ve a **Networking → Virtual Cloud Networks**.
    
2. Haz clic en **Create VCN**.
    
3. Completa los siguientes datos:
    
    - **Nombre:** `FRA-AA-LAB-1-VCN-01`
        
    - **Compartimento:** `<nombre del compartimento asignado>`
        
    - **CIDR IPv4:** `172.17.0.0/16`
        
    
    > Puedes dejar el resto de opciones como predeterminadas.
    
4. Haz clic en **Create VCN**.
    

### 1.2 Crear subredes

1. En la VCN creada, haz clic en **Create Subnet** y configura:
    
    - **Nombre:** `FRA-AA-LAB-1-SNET-01`
        
    - **CIDR:** `172.17.1.0/24`
        
    - **Tipo:** Regional
        
    - **Acceso:** Pública
        
    
    > Deja el resto predeterminado y haz clic en **Create Subnet**.
    
2. Repite el proceso para crear una segunda subred:
    
    - **Nombre:** `FRA-AA-LAB-1-SNET-02`
        
    - **CIDR:** `172.17.2.0/24`
        
    - **Acceso:** Pública
        
    - **Etiqueta DNS:** `FRAAAALAB1SNE2`
        

### 1.3 Crear Gateway de Internet

1. En la página de la VCN, ve a la pestaña **Gateways** → **Create Internet Gateway**.
    
2. Configura:
    
    - **Nombre:** `FRA-AA-LAB-1-IG-01`
        
3. Haz clic en **Create Internet Gateway**.
    

### 1.4 Crear regla de ruta

1. En la pestaña **Route Tables**, abre `Default Route Table for FRA-AA-LAB-1-VCN-01`.
    
2. Ve a **Route Rules → Add Route Rule** y completa:
    
    - **Destino:** `0.0.0.0/0`
        
    - **Tipo de destino:** Internet Gateway
        
    - **Target:** `FRA-AA-LAB-1-IG-01`
        
3. Guarda con **Add Route Rules**.
    

### 1.5 Configurar seguridad

1. En la VCN, ve a **Security → Default Security List**.
    
2. Agrega las siguientes reglas **entrantes**:

| Origen        | Protocolo | Puerto Destino |
| ------------- | --------- | -------------- |
| 172.16.0.0/12 | TCP       | 80             |
| 0.0.0.0/0     | TCP       | 80             |
| 0.0.0.0/0     | TCP       | 3389           |

3. Crea una nueva lista de seguridad:
    
    - **Nombre:** `FRA-AA-LAB-1-SL-01`
        
    - No agregues reglas aún.
        
4. Asigna esta lista a la subred `FRA-AA-LAB-1-SNET-02` y elimina la lista predeterminada de esa subred.
    
<img width="1352" height="582" alt="2" src="https://github.com/user-attachments/assets/1b436b45-93f4-4126-9ac0-d3558395c491" />

---

## 2️⃣ Crear Instancias de Cómputo (VMs)

### 2.1 Claves SSH (para Linux)

1. Abre **Cloud Shell** desde la consola de OCI.
    
2. Ejecuta:
    
    ```bash
    $ mkdir .ssh && cd .ssh
    $ ssh-keygen -b 2048 -t rsa -f ociaalab9key
    $ cat ociaalab9key.pub
    ```
    
3. Copia el contenido de la clave pública.
    

### 2.2 Crear instancia de Linux (VM-01)

1. Ve a **Compute → Instances → Create Instance**.
    
2. Configura:
    
    - **Nombre:** `FRA-AA-LAB-1-VM-01`
        
    - **Imagen:** Oracle Linux 8
        
    - **Forma:** VM.Standard.A1.Flex (1 OCPU, 6 GB)
        
    - **Subred:** `FRA-AA-LAB-1-SNET-01`
        
    - **Dirección IP pública:** Asignar
        
    - **Claves SSH:** Pega tu clave pública
        
3. En **Opciones avanzadas → Cloud-init**, pega:
    
    ```bash
    #!/bin/bash -x
    iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT
    yum -y install httpd
    systemctl enable httpd.service
    systemctl start httpd.service
    firewall-offline-cmd --add-service=http
    firewall-offline-cmd --add-service=https
    systemctl enable firewalld
    systemctl restart firewalld
    echo ¡Hola mundo! Mi nombre es FRA-AA-LAB-WS-01> /var/www/html/index.html
    ```
    
4. Haz clic en **Create**.
    

### 2.3 Crear instancia Windows (VM-02)

1. Repite el proceso para una VM Windows:
    
    - **Nombre:** `FRA-AA-LAB-1-VM-02`
        
    - **Imagen:** Windows Server 2022 Standard
        
    - **Subred:** `FRA-AA-LAB-1-SNET-01`
        
    - **SSH:** No requerido
        

### 2.4 Crear instancia base para autoescalado (VM-03)

1. Crea otra VM Linux:
    
    - **Nombre:** `FRA-AA-LAB-1-VM-03`
        
    - **Imagen:** Oracle Linux 9
        
    - **Clave SSH:** Pegar
        
2. Conéctate, instala `httpd` y `stress-ng`, luego crea una **Imagen personalizada** llamada `FRA-AA-LAB-1-CIM-01`.
    
---

## 3️⃣ Crear Balanceador de Carga (Load Balancer)

1. Desde **Networking → Load Balancers → Create Load Balancer**, configura:
    
    - **Nombre:** `FRA-AA-LAB-1-LB-01`
        
    - **Tipo:** Público
        
    - **Ancho de banda:** Min 10 / Max 20 Mbps
        
    - **Red/Subred:** `FRA-AA-LAB-1-VCN-01 / FRA-AA-LAB-1-SNET-01`
        
2. Configura backend set:
    
    - **Nombre:** `FRA-AA-LAB-1-LB-BS-01`
        
    - **Política:** Round Robin ponderado
        
    - **Check:** HTTP, puerto 80
        
3. Agrega Listener:
    
    - **Nombre:** `FRA-AA-LAB-1-LB-LS-01`
        
    - **Tipo:** HTTP, puerto 80
        
4. Finaliza creando el balanceador.
    

---

## 4️⃣ Configurar Autoescalado

### 4.1 Crear configuración de instancia (Instance Config)

- **Nombre:** `FRA-AA-LAB-1-AS-CF-01`
    
- **Imagen:** `FRA-AA-LAB-1-CIM-01`
    
- **Forma:** VM.Standard.A1.Flex
    
- **Subred:** `FRA-AA-LAB-1-SNET-01`
    

### 4.2 Crear Grupo de Instancias

- **Nombre:** `FRA-AA-LAB-1-INST-PL-01`
    
- **Instancias Iniciales:** 2
    
- **LB Backend:** `FRA-AA-LAB-1-LB-BS-01`
    
<img width="2095" height="555" alt="10" src="https://github.com/user-attachments/assets/4bb39366-0563-4622-adbc-8dd4a7de660c" />
<img width="1707" height="456" alt="8" src="https://github.com/user-attachments/assets/fe7d8241-d440-4dbe-b2ed-c5cf0b69af6e" />


### 4.3 Crear Política de Autoescalado

- **Nombre:** `FRA-AA-LAB-1-AS-POL-01`
    
- **Métrica:** CPU
    
- **Escalar ↑:** >70% → +1 instancia
    
- **Escalar ↓:** <20% → -1 instancia
    
- **Capacidad:** Min 1 / Max 3 / Inicial 2
    
<img width="1958" height="593" alt="1" src="https://github.com/user-attachments/assets/66d0400d-862e-43dd-be11-1bbad48de87e" />

---

## 5️⃣ Configurar DNS Privado

1. Encuentra la **IP Privada** de `FRA-AA-LAB-1-VM-01`.
    
2. Ve a **Networking → DNS → Zonas → Crear zona**.
    
    - **Nombre:** `FRA-AA-LAB-PrivateZone-01.com`
        
    - **Vista privada:** `FRA-AA-LAB-1-VCN-01`
        
3. Dentro de la zona, agrega un registro tipo A:
    
    - **Nombre:** `www`
        
    - **Dirección:** `<IP privada de VM-01>`
        
4. Publica cambios.
    
5. Desde VM-02 (Windows), prueba navegación a `http://www.FRA-AA-LAB-PrivateZone-01.com`.
    

---

## 6️⃣ Crear File System (NFS) y Montaje

### 6.1 Crear File System

- **Nombre:** `FRA-AA-LAB-1-FS-01`
    
- **Export Path:** `/FRA-AA-LAB-1-EP-01`
    
- **Mount Target:** `FRA-AA-LAB-1-MRT-01` en subred `FRA-AA-LAB-1-SNET-02`
    
<img width="1772" height="402" alt="4" src="https://github.com/user-attachments/assets/2bff0e14-0dbd-49be-b634-9a2b7801fff4" />

### 6.2 Configurar Reglas de Seguridad

- **En SL-01 (SNET-02):** Permitir puertos 111, 2048–2050 (TCP/UDP) desde `172.17.1.0/24`
    
- **En Default Security List:** Permitir NFS desde y hacia la subred del Mount Target
    
<img width="1211" height="745" alt="5" src="https://github.com/user-attachments/assets/888b8123-87af-4aca-8360-779db76963c8" />

### 6.3 Configurar Opciones de Exportación

- VM-01 → Solo lectura
    
- VM-02 → Lectura/Escritura
    
<img width="2555" height="1075" alt="6" src="https://github.com/user-attachments/assets/d7254fde-b72c-4ecf-ac3b-26266994ee6b" />

### 6.4 Montar y Probar

- Desde ambas VMs, monta el sistema usando los comandos NFS.
    
- Valida que VM-01 no pueda escribir y VM-02 sí.
    
<img width="2108" height="286" alt="7" src="https://github.com/user-attachments/assets/950b18c9-bb22-4b8b-8d6b-b8029480741e" />
<img width="1221" height="40" alt="9" src="https://github.com/user-attachments/assets/8ae19b74-7296-4849-b3e9-993ef06870f1" />

---

## 🧪 Validaciones

|Componente|Validación|
|---|---|
|Balanceador (LB)|Acceso web desde la IP pública 🟢|
|DNS Privado|Resolución interna desde VM-02 hacia VM-01 🟢|
|Autoescalado|Instancias se incrementan/disminuyen según CPU 🟢|
|File System (NFS)|VM-01 solo lectura / VM-02 lectura-escritura 🟢|

---

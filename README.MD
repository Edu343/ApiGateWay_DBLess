# Arquitectura de API Gateway 

Este repositorio contiene la Infraestructura como código para desplegar un API Gateway usando Kong (DB-Less), Apache, monitoreo de métricas con Prometheus/Grafana y gestión centralizada de identidades con Keycloak.

## Pre-requisitos del Servidor
* Servidor Linux con Docker y Docker Compose instalados.
* Git instalado.
* Puertos libres: `8080`, `8443` (Tráfico Apache), `8082` (Admin Keycloak), `3000` (Grafana).

## Instrucciones de Despliegue

### 1. Clonar el repositorio y preparar el entorno

```bash
git clone https://github.com/Edu343/ApiGateWay_DBLess.git
cd PracticasApiGateWay_DBLess_S
```

### 2. Certificados de Seguridad (Apache)
Por lineamientos de seguridad, los certificados TLS/SSL no están versionados en este repositorio. Antes de iniciar, es obligatorio colocar los certificados reales dentro de la carpeta `./certs/`:

* `./certs/server.crt`
* `./certs/server.key`

### 3. Despliegue Automatizado
El entorno está configurado para auto-aprovisionarse. Keycloak importará automáticamente la estructura del reino, clientes y configuraciones base desde el archivo `api-realm-export.json`.

Ejecuta el siguiente comando para construir la red, aprovisionar los volúmenes y levantar todos los servicios:

```bash
docker compose up -d
```

> **Nota:** Para detener el ecosistema por mantenimiento, utiliza únicamente `docker compose down`. No utilizar la bandera `-v`, ya que esto destruirá los volúmenes persistentes y eliminará el historial de métricas de Prometheus y las bases de datos de usuarios.

### 4. Sincronización Criptográfica (Firma JWT)
Aunque el entorno nace configurado, Keycloak genera un nuevo par de llaves criptográficas seguras (RSA) la primera vez que arranca en un servidor nuevo. Es necesario sincronizar la llave pública con el API Gateway:

1. Ingresa a la consola de Keycloak en `http://<IP_DEL_SERVIDOR>:8082` (Usuario: `admin` / Password: `admin`).
2. Selecciona el reino `api-realm`.
3. Ir a **Realm Settings > Keys**, busca el algoritmo `RS256 (SIG)` y copia la Llave Pública en **Public Key**.
4. Edita el archivo `kong.yaml` y reemplaza el bloque de texto debajo de `rsa_public_key:` con la llave copiada, respetando las cabeceras `-----BEGIN PUBLIC KEY-----` y `-----END PUBLIC KEY-----`.

### 5. Aplicar la Seguridad 
Una vez pegada la llave correcta, reinicia únicamente el Gateway para que lea la nueva configuración criptográfica en memoria:

```bash
docker compose restart kong
```

 ## Pruebas de Funcionamiento

El Gateway ahora está completamente blindado bajo un esquema de doble validación (JWT + ApiKey). Para consumir la API protegida, sigue estos dos pasos:

### Paso 1: Solicitar un Token de Acceso a Keycloak

```bash
curl -s -X POST http://localhost:8082/realms/api-realm/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=api-cliente" \
  -d "username=prueba" \
  -d "password=1234" \
  -d "grant_type=password"
```

> *(Extrae el valor de la cadena `access_token` de la respuesta JSON).*

### Paso 2: Consumir la API a través de Kong

Sustituye el token en el siguiente comando para atravesar el Gateway:

```bash
curl -k -i -w "\nEstatus: %{http_code}\n" \
  -H "Authorization: Bearer <PEGAR_AQUÍ_EL_ACCESS_TOKEN>" \
  -H "apikey: super-secreto-123" \
  https://localhost:8443/visual
```

Si la sincronización fue exitosa, Kong validará matemáticamente la firma del token y devolverá un estatus `200 OK`.
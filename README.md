# 🔐 Laboratorio: Azure AD OAuth2 PKCE + API Local

## 🧾 Descripción
Laboratorio completo que implementa el flujo OAuth2 Authorization Code con PKCE utilizando Microsoft Entra ID (Azure AD) como proveedor de identidad y una API local Node.js/Express protegida mediante validación JWT con JWKS.

Este laboratorio demuestra cómo:
- Generar credenciales PKCE (`code_verifier` y `code_challenge`)
- Construir la URL de autorización para Azure AD
- Obtener un `authorization_code` tras autenticación
- Intercambiar el código por un `access_token`
- Validar el JWT en una API local usando JWKS
- Acceder a recursos protegidos con el token

---

## 🏗️ Arquitectura del flujo

```
Usuario/CLI → Azure AD → Cliente PKCE → Azure AD → API Local → Respuesta protegida
```

### Flujo resumido
1. El cliente genera PKCE  
2. Solicita autorización a Azure AD  
3. Azure AD devuelve `authorization_code`  
4. El cliente intercambia el código usando `code_verifier`  
5. Azure AD emite `access_token` (JWT)  
6. La API valida el token mediante JWKS  
7. Se accede al recurso protegido

---

## 📋 Requisitos previos
- Node.js 18+  
- npm  
- bash / WSL / macOS / Linux  
- curl  
- openssl  
- Cuenta Azure AD con permisos para registrar apps  
- Git

---

## 🚀 Instalación y ejecución

### 1. Clonar el repositorio
```bash
git clone https://github.com/lanuzalorenzo/lab-azuread-oauth2-pkce-api.git
cd lab-azuread-oauth2-pkce-api
```

### 2. Instalar dependencias
```bash
cd api-local
npm install
```

### 3. Configurar `.env`
Crear archivo:

```bash
AZURE_TENANT_ID=<tenant-id>
AZURE_CLIENT_ID=<client-id>
AZURE_API_CLIENT_ID=<api-application-id>
AZURE_ISSUER=https://login.microsoftonline.com/<tenant-id>/v2.0
AZURE_JWKS_URI=https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys

PORT=4010
REDIRECT_URI=http://localhost/callback
API_AUDIENCE=api://<api-application-id>/access_as_user
VALID_ISSUERS=https://login.microsoftonline.com/<tenant-id>/v2.0
```

### 4. Ejecutar la API
```bash
npm start
```

Endpoints:
- `http://localhost/health`
- `http://localhost/api/products` (protegido)

---

## 🔑 Flujo PKCE paso a paso

### 1. Generar PKCE
```bash
cd api-local/scripts
./generate-pkce.sh
```

Salida:
```
code_verifier=...
code_challenge=...
```

### 2. Construir URL de autorización
Incluye:
- `client_id`
- `redirect_uri`
- `scope`
- `code_challenge`
- `code_challenge_method=S256`

### 3. Autenticarse y obtener `authorization_code`
Azure AD redirige a:
```
http://localhost/callback?code=<authorization_code>
```

### 4. Intercambiar código por token
```bash
./exchange-token.sh
```

### 5. Consumir API protegida
```bash
export ACCESS_TOKEN="<token>"
./call-api.sh
```

---

## 🔍 Validación JWT en la API
La API valida:
- firma RS256  
- `iss`  
- `aud`  
- `exp`  
- `nbf`  
- `kid` presente en JWKS  

Código relevante:
- `api-local/utils/validateToken.js`

---

## ⚠️ Errores comunes

### 501481 — PKCE no habilitado
Solución:
- `code_challenge_method=S256`
- `redirect_uri` correcto
- habilitar *public client flows*

### 90013 — Falta consentimiento
Solución:
- conceder permisos delegados
- aceptar consentimiento administrativo

### Token inválido
Solución:
- revisar `aud`
- revisar `iss`
- revisar expiración
- revisar configuración `.env`

---

## 🗂️ Estructura del proyecto

```
lab-azuread-oauth2-pkce-api/
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   └── laboratorio-pkce-azure.md
├── api-local/
│   ├── .env.example
│   ├── package.json
│   ├── server.js
│   ├── README.md
│   ├── scripts/
│   │   ├── generate-pkce.sh
│   │   ├── exchange-token.sh
│   │   └── call-api.sh
│   └── utils/
│       └── validateToken.js
```

---

## ⚖️ Aviso legal
Este laboratorio es exclusivamente educativo.  
No debe usarse para acceder a sistemas sin autorización.

---

## 📜 Licencia
MIT

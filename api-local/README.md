# 🔐 API Local — OAuth2 PKCE + Azure AD

## 🧾 Descripción
Laboratorio técnico que implementa una API local en Node.js/Express protegida mediante validación JWT usando JWKS de Azure AD.  
La autenticación sigue el flujo OAuth2 Authorization Code con PKCE, incluyendo generación de PKCE, obtención del token y validación completa del JWT.

---

## 📦 Requisitos
- Node.js 18+  
- npm  
- Azure AD / Microsoft Entra ID con aplicación registrada  
- curl  
- openssl  

---

## 🗂️ Estructura del laboratorio
```
api-local/
├── scripts/
│   ├── generate-pkce.sh
│   ├── exchange-token.sh
│   └── call-api.sh
├── utils/
│   └── validateToken.js
├── .env
├── package.json
├── package-lock.json
├── server.js
└── README.md
```

---

## ⚙️ Instalación
```bash
cd api-local
npm install
```

---

## 🔧 Configuración
Editar `.env` con los valores reales:

```
AZURE_TENANT_ID=
AZURE_CLIENT_ID=
API_APPLICATION_ID=
REDIRECT_URI=http://localhost/callback
```

Asegúrate de que tu aplicación Azure AD tenga:

- Redirect URI: `http://localhost/callback`  
- Plataforma Web o SPA  
- API expuesta con scope `access_as_user`  
- Permiso delegado concedido  

---

## 🚀 Ejecutar la API
```bash
cd api-local
npm start
```

Endpoints disponibles:
```
http://localhost/health
http://localhost/api/products
```

---

## 🧪 Probar la API sin token
```bash
curl http://localhost/api/products
```

Respuesta esperada: `401 Unauthorized`

---

## 🔐 Generar PKCE
```bash
cd api-local
./scripts/generate-pkce.sh
```

El script genera:
- `code_verifier`
- `code_challenge`

---

## 🔑 Obtener token desde Azure AD

### Variables necesarias
```bash
export AZURE_TENANT_ID="<tenant-id>"
export AZURE_CLIENT_ID="<client-id>"
export REDIRECT_URI="http://localhost/callback"
export CODE_VERIFIER="<code_verifier>"
export AUTHORIZATION_CODE="<authorization_code>"
```

### Intercambiar el authorization_code por token
```bash
./scripts/exchange-token.sh
```

---

## 📡 Consumir la API con token
```bash
export ACCESS_TOKEN="<access_token>"
./scripts/call-api.sh
```

---

## 🔍 Validación JWT
La API valida el token usando JWKS de Azure AD comprobando:

- `iss`  
- `aud`  
- `exp`  
- `nbf`  
- firma RS256  
- `kid` presente en JWKS  

---

## 🛠️ Errores comunes

### ❌ Error 501481  
La app no permite flujo PKCE o la política no coincide.  
**Solución:** habilitar *Allow public client flows* o ajustar el tipo de aplicación.

### ❌ Error 90013  
El usuario no tiene permisos o falta consentimiento.  
**Solución:** conceder permisos delegados y aceptar consentimiento.

---

## 🧪 Ejemplo de flujo completo
```bash
cd api-local
npm install
npm start

./scripts/generate-pkce.sh
# Construir URL de autorización con code_challenge

./scripts/exchange-token.sh
export ACCESS_TOKEN="..."
./scripts/call-api.sh
```

---

## ✔️ Notas
Este laboratorio demuestra:

- Seguridad del flujo PKCE  
- Validación de tokens JWT con JWKS  
- Integración real con Azure AD  

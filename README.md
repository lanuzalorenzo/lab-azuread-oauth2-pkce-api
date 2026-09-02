# Lab Azure AD OAuth2 PKCE API

**Laboratorio completo de autenticación OAuth2 Authorization Code Flow con PKCE usando Azure AD**

Este repositorio contiene una implementación educativa y profesional del flujo OAuth2 Authorization Code con PKCE, integrando Microsoft Entra ID (Azure AD) como proveedor de identidades y una API local Node.js/Express protegida con validación JWT mediante JWKS.

---

## 🎯 ¿Qué es este laboratorio?

Este proyecto demuestra cómo:

1. **Generar PKCE credentials** (`code_verifier` y `code_challenge`) de forma segura
2. **Construir una URL de autorización** para Microsoft Entra ID
3. **Obtener un authorization code** tras la autenticación del usuario
4. **Intercambiar el code por un access_token** usando el `code_verifier`
5. **Validar el JWT** en una API local usando la clave pública de Azure AD (JWKS)
6. **Acceder a recursos protegidos** con el access_token

Está diseñado para investigadores de ciberseguridad, desarrolladores y estudiantes que deseen comprender en profundidad el flujo OAuth2 moderno con PKCE.

---

## 🏗️ Arquitectura

```
┌─────────────────┐
│  Usuario/CLI    │
│  (navegador)    │
└────────┬────────┘
         │
         │ 1. Solicita acceso (con PKCE)
         v
┌──────────────────────────┐
│  Microsoft Entra ID      │
│  (Azure AD)              │
└────────┬─────────────────┘
         │
         │ 2. Emite authorization code
         │
         v
┌─────────────────┐
│  Cliente CLI    │
│  (bash script)  │
└────────┬────────┘
         │
         │ 3. Intercambia code + code_verifier
         │    por access_token
         v
┌──────────────────────────┐
│  Microsoft Entra ID      │
│  Token Endpoint          │
└────────┬─────────────────┘
         │
         │ 4. Retorna access_token
         │    (JWT firmado RS256)
         v
┌──────────────────────────┐
│  API Local Node.js       │
│  (puerto 4010)           │
│                          │
│  GET /api/products       │
│  (protegido con JWT)     │
└──────────────────────────┘
```

---

## ⚡ ¿Por qué PKCE?

**PKCE (Proof Key for Code Exchange)** es el estándar moderno para aplicaciones públicas:

- ✅ **Sin secreto del cliente**: Elimina la necesidad de almacenar credenciales confidenciales
- ✅ **Resistente a interceptación**: El `code_verifier` se genera en el cliente y nunca se transmite al navegador
- ✅ **Seguro para SPA y móviles**: Protege contra ataques de autorización code interception (PKCE bypasses)
- ✅ **Estándar OAuth2**: RFC 7636

---

## 📋 Requisitos previos

Antes de comenzar, asegúrate de tener:

- **Node.js** 18 o superior
- **npm** (incluido en Node.js)
- **bash** shell (Linux, macOS o WSL en Windows)
- **openssl** (generalmente pre-instalado)
- **curl** para hacer peticiones HTTP
- **Cuenta en Azure/Microsoft 365** con permisos de administrador de aplicaciones
- **Git** (para clonar el repositorio)

---

## 🚀 Guía de instalación y ejecución

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

### 3. Configurar variables de entorno

Crea un archivo `.env` en `api-local/`:

```bash
cat > api-local/.env << 'EOF'
# Configuración de Azure AD
AZURE_TENANT_ID=<tu-tenant-id>
AZURE_CLIENT_ID=<tu-client-id>
AZURE_API_CLIENT_ID=<tu-api-application-id>
AZURE_ISSUER=https://login.microsoftonline.com/<tu-tenant-id>/v2.0
AZURE_JWKS_URI=https://login.microsoftonline.com/<tu-tenant-id>/discovery/v2.0/keys

# Configuración de la API
PORT=4010

# Configuración de OAuth2
REDIRECT_URI=http://localhost:8080/callback
API_AUDIENCE=api://<tu-api-application-id>/access_as_user
VALID_ISSUERS=https://login.microsoftonline.com/<tu-tenant-id>/v2.0
EOF
```

**¿Dónde encontrar estos valores?**

- `AZURE_TENANT_ID`: Azure Portal → Directory ID (en la página de inicio de Entra ID)
- `AZURE_CLIENT_ID`: Application ID de la aplicación cliente registrada
- `AZURE_API_CLIENT_ID`: Application ID de la API registrada
- `<tu-tenant-id>`: Mismo valor que `AZURE_TENANT_ID`

### 4. Ejecutar la API local

```bash
cd api-local
npm start
```

Deberías ver:

```
API local ejecutándose en http://localhost:4010
Health check: http://localhost:4010/health
Endpoint protegido: http://localhost:4010/api/products
Azure JWKS: https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys
```

### 5. Probar el endpoint de salud (sin token)

En otra terminal:

```bash
curl http://localhost:4010/health
```

Respuesta esperada:

```json
{
  "ok": true,
  "status": "healthy",
  "service": "lab-azuread-oauth2-pkce-api",
  "timestamp": "2026-09-02T10:30:00.000Z",
  "port": 4010
}
```

---

## 🔑 Flujo completo: Paso a paso

### Paso 1: Generar PKCE credentials

```bash
cd api-local/scripts
./generate-pkce.sh
```

Salida:

```
code_verifier=abc123...xyz
code_challenge=def456...uvw
```

**Guarda estos valores** en variables de entorno o en un archivo temporal.

### Paso 2: Construir la URL de autorización

Crea la URL usando los parámetros generados:

```bash
export TENANT_ID="<tu-tenant-id>"
export CLIENT_ID="<tu-client-id>"
export CODE_CHALLENGE="<valor-del-paso-1>"
export STATE="state123456"

echo "https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/authorize?\
client_id=${CLIENT_ID}\
&response_type=code\
&redirect_uri=http://localhost:8080/callback\
&response_mode=query\
&scope=api://<tu-api-application-id>/access_as_user\
&state=${STATE}\
&code_challenge=${CODE_CHALLENGE}\
&code_challenge_method=S256"
```

Abre esta URL en tu navegador.

### Paso 3: Autenticarte y obtener el authorization code

1. Inicia sesión con tu cuenta de Azure AD
2. Consiente los permisos solicitados
3. Serás redirigido a `http://localhost:8080/callback?code=...&state=...`
4. **Copia el valor del parámetro `code`**

### Paso 4: Intercambiar el code por access_token

```bash
export AZURE_TENANT_ID="<tu-tenant-id>"
export AZURE_CLIENT_ID="<tu-client-id>"
export REDIRECT_URI="http://localhost:8080/callback"
export CODE_VERIFIER="<valor-del-paso-1>"
export AUTHORIZATION_CODE="<valor-del-paso-3>"

cd api-local/scripts
./exchange-token.sh
```

Respuesta esperada:

```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ...",
  "refresh_token": "0.AS4AXxZ...",
  "scope": "api://<api-id>/access_as_user"
}
```

### Paso 5: Llamar a la API protegida

```bash
export ACCESS_TOKEN="<access_token-del-paso-4>"

cd api-local/scripts
./call-api.sh
```

O manualmente:

```bash
curl -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  http://localhost:4010/api/products
```

Respuesta esperada:

```json
{
  "ok": true,
  "message": "Acceso autorizado mediante OAuth2 PKCE + Azure AD.",
  "user": {
    "oid": "...",
    "preferred_username": "user@example.com",
    "name": "User Name",
    "aud": "api://<api-id>",
    "iss": "https://login.microsoftonline.com/<tenant-id>/v2.0"
  },
  "products": [
    { "id": 1, "name": "Laptop Pro 14", "price": 1299.99, "category": "Hardware" },
    { "id": 2, "name": "Monitor 27\" 4K", "price": 799.99, "category": "Hardware" },
    ...
  ]
}
```

---

## 📚 Documentación detallada

Para entender el flujo PKCE en profundidad, consulta:

- **[docs/laboratorio-pkce-azure.md](docs/laboratorio-pkce-azure.md)** - Explicación técnica detallada del flujo PKCE y su implementación
- **[docs/bitacora.md](docs/bitacora.md)** - Registro del trabajo realizado, decisiones técnicas y conclusiones
- **[api-local/README.md](api-local/README.md)** - Guía específica de la API local

---

## 🛠️ Estructura del proyecto

```
lab-azuread-oauth2-pkce-api/
├── README.md                          # Este archivo
├── LICENSE                            # MIT License
├── .gitignore                         # Configuración de Git
│
├── docs/
│   ├── laboratorio-pkce-azure.md      # Explicación técnica del flujo PKCE
│   ├── bitacora.md                    # Registro del trabajo realizado
│   └── capturas/                      # Capturas de pantalla del flujo
│
├── api-local/                         # API local protegida con OAuth2
│   ├── .env                           # Variables de entorno (no versionado)
│   ├── .env.example                   # Plantilla de .env
│   ├── package.json                   # Dependencias de Node.js
│   ├── server.js                      # Servidor Express
│   ├── README.md                      # Guía de la API local
│   │
│   ├── scripts/                       # Scripts de utilidad
│   │   ├── generate-pkce.sh           # Genera code_verifier y code_challenge
│   │   ├── exchange-token.sh          # Intercambia code por access_token
│   │   └── call-api.sh                # Llama al endpoint protegido
│   │
│   └── utils/
│       └── validateToken.js           # Validación de JWT usando JWKS
│
└── my-project/                        # Proyecto de ejemplo (sin usar)
    └── src/
        ├── components/
        ├── services/
        └── types/
```

---

## 🔐 Seguridad y validación

### Validación en la API local

El archivo `api-local/utils/validateToken.js` implementa:

1. **Decodificación del JWT sin verificación**: Lee el header para obtener el `kid` (Key ID)
2. **Obtención de la clave pública**: Consulta el JWKS endpoint de Azure AD
3. **Verificación de la firma**: Usa la clave pública para validar la firma RS256
4. **Validación de claims**:
   - `exp` (expiration time): Token no expirado
   - `nbf` (not before): Token aún no válido si es futuro
   - `iss` (issuer): Emitido por Azure AD
   - `aud` (audience): Emitido para esta API
   - `kid` (key ID): Identifica la clave pública usada

### Validación en Azure AD (antes de emitir el token)

- Verifica que el usuario esté autenticado
- Verifica que el cliente tenga permisos delegados para el scope solicitado
- Verifica que el `code_verifier` coincida con el `code_challenge` original
- Emite un token JWT firmado con la clave privada de Azure AD

---

## ⚠️ Aviso legal

Este laboratorio es **solo para fines educativos y de investigación**. 

**NO** está permitido:
- Usar este código para acceder a sistemas sin autorización
- Modificar, interceptar o redirigir tokens de producción
- Replicar este flujo sin consentimiento explícito del propietario del tenant de Azure AD

**El autor no es responsable** de cualquier daño, acceso no autorizado o violación de términos de servicio causado por el mal uso de este código.

Siempre:
- ✅ Obtén permisos escritos antes de realizar testing de seguridad
- ✅ Usa ambientes aislados para experimentos
- ✅ Documenta todo lo que hagas
- ✅ Sigue las políticas de responsabilidad disclosure

---

## 🤝 Contribuciones

Este laboratorio es un proyecto educativo. Si encuentras errores o tienes sugerencias:

1. Abre un issue en GitHub
2. Crea un pull request con tus mejoras
3. Comparte feedback constructivo

---

## 🔗 Referencias y recursos

- **[RFC 7636 - PKCE](https://tools.ietf.org/html/rfc7636)** - Estándar oficial de PKCE
- **[Azure AD Documentation](https://learn.microsoft.com/en-us/entra/identity/platform/)** - Documentación de Microsoft Entra ID
- **[OAuth 2.0 Authorization Code Flow](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-07#section-1.3.1)** - Especificación del flujo
- **[JWT (JSON Web Tokens)](https://jwt.io/)** - Explicación y validación de JWT
- **[JWKS - JSON Web Key Set](https://tools.ietf.org/html/rfc7517)** - Formato de clave pública

---

## 💼 Autor

**Lorenzo Lanuza Jiménez**

- 🔗 [Portfolio de Ciberseguridad](https://github.com/lanuzalorenzo/portfolio-ciberseguridad-lorenzo)
- 📧 [Contacto profesional]

---

## 📄 Licencia

Este proyecto está licenciado bajo la **Licencia MIT**. Ver [LICENSE](LICENSE) para más detalles.

---

## ✨ Agradecimientos

Este laboratorio fue creado como parte de la investigación en OAuth2, PKCE y seguridad de autenticación. Agradecemos a la comunidad de ciberseguridad por el feedback y las contribuciones.

---

**Última actualización**: 2 de septiembre de 2026  
**Versión**: 1.0.0

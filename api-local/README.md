# API local con OAuth2 PKCE + Azure AD

Este laboratorio incluye una API local en Node.js/Express protegida con validación JWT usando JWKS de Azure AD, siguiendo el flujo OAuth2 Authorization Code con PKCE.

## Requisitos

- Node.js 18+
- npm
- Azure AD / Microsoft Entra ID con una aplicación registrada
- curl
- openssl

## Estructura

```text
api-local/
  .env
  package.json
  server.js
  scripts/
    generate-pkce.sh
    exchange-token.sh
    call-api.sh
  utils/
    validateToken.js
```

## Instalación

```bash
cd api-local
npm install
```

## Configuración

1. Edita el archivo `.env` con el Tenant ID, Client ID y API Application ID reales.
2. Asegúrate de que tu aplicación Azure AD tenga:
   - Redirect URI: `http://localhost:8080/callback`
   - Plataforma Web o SPA según corresponda
   - API expuesta con scope `access_as_user`
   - Permiso delegado al scope de la API

## Ejecutar la API

```bash
cd api-local
npm start
```

La API queda disponible en:

- `http://localhost:4010/health`
- `http://localhost:4010/api/products`

## Probar la API sin token

```bash
curl http://localhost:4010/api/products
```

Debe responder con `401`.

## Generar PKCE

```bash
cd api-local
./scripts/generate-pkce.sh
```

Copia el `code_verifier` y el `code_challenge` generados.

## Obtener el token desde Azure AD

Ejemplo con `curl`:

```bash
export AZURE_TENANT_ID="<tenant-id>"
export AZURE_CLIENT_ID="<client-id>"
export REDIRECT_URI="http://localhost:8080/callback"
export CODE_VERIFIER="<code_verifier>"
export AUTHORIZATION_CODE="<authorization_code>"

./scripts/exchange-token.sh
```

## Consumir la API local

```bash
export ACCESS_TOKEN="<access_token>"
./scripts/call-api.sh
```

## Validación JWT

La validación se realiza usando `jwks-rsa` con el JWKS de Azure AD y comprobando:

- `iss`
- `aud`
- `exp`
- `nbf`
- firma RS256
- `kid` con clave publicada en JWKS

## Solución de problemas comunes

### Error 501481

Se produce cuando la app no permite el flujo de autenticación de tipo `PKCE` o la política de acceso está configurada para un tipo de flujo distinto.

Solución:

- En Azure AD, en la aplicación, habilitar `Allow public client flows` o configurar la aplicación apropiada.
- Usar la URI correcta y el `redirect_uri` esperado.

### Error 90013

Se produce cuando el usuario no tiene permiso para la aplicación o el cliente no tiene un consentimiento válido.

Solución:

- Confirmar que la aplicación tiene permisos del tipo `Delegado`.
- Aceptar el consentimiento.
- Verificar que el `scope` consulte el recurso correcto.

## Ejemplo de flujo completo

```bash
cd api-local
npm install
npm start
./scripts/generate-pkce.sh
# Construir la URL de autorización en Azure AD
# Usar code_challenge obtenida
# Intercambiar authorization_code por token
./scripts/exchange-token.sh
export ACCESS_TOKEN="..."
./scripts/call-api.sh
```

## Notas

Este laboratorio está pensado para demostrar la seguridad del flujo PKCE y la validación del token en la API local.

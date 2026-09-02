# Bitácora del laboratorio PKCE + Azure AD

## Fecha

2026-09-02

## Objetivo

Crear un laboratorio completo para autenticar una app cliente con OAuth2 Authorization Code Flow + PKCE y validar el token JWT en una API local con Azure AD.

## 1. Generación de PKCE

Se creó el script `api-local/scripts/generate-pkce.sh` para generar:

- `code_verifier`
- `code_challenge`

Comando ejecutado:

```bash
cd api-local
./scripts/generate-pkce.sh
```

Resultado esperado:

```text
code_verifier=...
code_challenge=...
```

## 2. Construcción de la URL de autorización

La URL base de Microsoft Entra ID es:

```text
https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize
```

Parámetros configurados:

- `client_id`
- `response_type=code`
- `redirect_uri=http://localhost:8080/callback`
- `scope=api://<api-client-id>/access_as_user`
- `code_challenge=<challenge>`
- `code_challenge_method=S256`
- `state=<valor aleatorio>`

## 3. Errores observados

### Error 501481

Se presenta cuando el flujo requiere una configuración no compatible o el código no se emite correctamente.

Causa probable:

- `redirect_uri` incorrecto
- no se habilita PKCE
- el cliente no está configurado para el tipo de flujo correcto

Solución aplicada:

- verificar la aplicación cliente
- configurar URL correcta
- usar `code_challenge_method=S256`

### Error 90013

Se presenta cuando la aplicación no tiene consentimiento o permisos adecuados.

Causa probable:

- falta consentimiento delegado
- scope incorrecto
- permisos no aceptados por el tenant

Solución aplicada:

- validar que el scope sea `api://<api-client-id>/access_as_user`
- confirmar que el usuario tiene acceso
- conceder consentimiento si es necesario

## 4. Intercambio de token

Se creó el script `api-local/scripts/exchange-token.sh` para realizar el intercambio:

```bash
export AZURE_TENANT_ID="<tenant-id>"
export AZURE_CLIENT_ID="<client-id>"
export REDIRECT_URI="http://localhost:8080/callback"
export CODE_VERIFIER="<code_verifier>"
export AUTHORIZATION_CODE="<authorization_code>"
./scripts/exchange-token.sh
```

La respuesta esperada incluye:

```json
{
  "token_type": "Bearer",
  "expires_in": 3599,
  "access_token": "...",
  "refresh_token": "...",
  "scope": "api://<api-client-id>/access_as_user"
}
```

## 5. Validación del token en la API local

Se implementó `api-local/utils/validateToken.js` con validación de:

- `kid` en el header
- `aud`
- `iss`
- `nbf`
- `exp`
- firma RS256 mediante JWKS

Se usa `jwks-rsa` con el endpoint:

```text
https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys
```

## 6. Pruebas realizadas

### 6.1 Verificar salud de la API

```bash
curl http://localhost:4010/health
```

Resultado esperado:

```json
{
  "ok": true,
  "status": "healthy",
  "service": "lab-azuread-oauth2-pkce-api"
}
```

### 6.2 Llamada sin token

```bash
curl http://localhost:4010/api/products
```

Resultado esperado:

```json
{
  "ok": false,
  "message": "Token no válido o caducado."
}
```

### 6.3 Llamada con token válido

```bash
export ACCESS_TOKEN="<access_token>"
curl -H "Authorization: Bearer $ACCESS_TOKEN" http://localhost:4010/api/products
```

Resultado esperado:

```json
{
  "ok": true,
  "message": "Acceso autorizado mediante OAuth2 PKCE + Azure AD.",
  "products": [ ... ]
}
```

## 7. Conclusiones

- PKCE es una mejora crítica para flujos con clientes públicos.
- La validación del JWT debe hacerse siempre en la API.
- Azure AD emite tokens seguros y firmados con claves rotativas.
- La combinación entre `code_verifier`, challenge y validación de `JWKS` produce un flujo robusto y seguro.

## 8. Estado del laboratorio

Completado con:

- API local funcional
- validación JWT con JWKS
- scripts de PKCE y token exchange
- documentación y bitácora
- ejemplos de pruebas

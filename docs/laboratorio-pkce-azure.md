# Laboratorio: OAuth2 Authorization Code Flow con PKCE en Azure AD

## Objetivo

Este laboratorio muestra cómo proteger una API local con JWT emitido por Microsoft Entra ID (Azure AD) usando el flujo OAuth2 Authorization Code con PKCE. La idea principal es que el cliente autenticado obtiene un código de autorización y, en lugar de enviar un secreto del cliente, utiliza un `code_verifier` y su `code_challenge` para demostrar que el flujo está correctamente ligado al cliente original.

## Arquitectura

```text
Usuario / Browser
       |
       | 1. Solicita acceso a la App
       v
Azure AD / Microsoft Entra ID
       |
       | 2. Emite authorization code + PKCE challenge
       v
Cliente / Script / SPA
       |
       | 3. Exchange authorization code
       |    con code_verifier
       v
Azure AD
       |
       | 4. Emite access_token + refresh_token
       v
API local (Node.js + Express)
       |
       | 5. Valida JWT usando JWKS
       v
Respuesta con datos protegidos
```

## Pregunta clave

¿Por qué PKCE?

Porque elimina la necesidad de un secreto del cliente y hace que el flujo sea seguro incluso en aplicaciones públicas o clientes sin capacidad de almacenar credenciales confidenciales.

## Flujo detallado

### 1. Registro de la aplicación

En Azure AD se registran dos piezas:

- Aplicación cliente o SPA / public client
- API protegida (Resource API)

La API debe exponer un scope como por ejemplo:

```text
api://<api-client-id>/access_as_user
```

La aplicación cliente debe tener permisos delegados para acceder a ese scope.

### 2. Generación de code_verifier y code_challenge

El cliente crea un `code_verifier` aleatorio y un `code_challenge`:

```text
code_verifier = valor aleatorio de 43 a 128 caracteres
code_challenge = BASE64URL(SHA256(code_verifier))
```

La API local incluye el script:

```bash
./api-local/scripts/generate-pkce.sh
```

### 3. Construcción de la URL de autorización

La URL se construye apuntando a Azure AD:

```text
https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize
  ?client_id=<client-id>
  &response_type=code
  &redirect_uri=http://localhost:8080/callback
  &response_mode=query
  &scope=api://<api-client-id>/access_as_user
  &state=123456
  &code_challenge=<code_challenge>
  &code_challenge_method=S256
```

### 4. Obtención del authorization_code

Tras la autenticación del usuario, Azure AD redirige al navegador a `redirect_uri` con un `code` y un `state`.

### 5. Intercambio del code por token

Se realiza una petición POST al token endpoint usando el mismo `code_verifier` original:

```bash
curl -X POST "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "client_id=<client-id>" \
  --data-urlencode "scope=api://<api-client-id>/access_as_user" \
  --data-urlencode "code=<authorization_code>" \
  --data-urlencode "redirect_uri=http://localhost:8080/callback" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "code_verifier=<code_verifier>"
```

La respuesta contiene:

- `access_token`
- `refresh_token`
- `token_type`
- `expires_in`

### 6. Validación en la API local

La API local no confía en el cliente. Valida el JWT usando la clave pública publicada por Azure AD en el JWKS:

```text
https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys
```

Se verifica:

- `iss`
- `aud`
- `exp`
- `nbf`
- firma RS256
- `kid`

## Código de la API local

La API local se define en `api-local/server.js` y protege el endpoint:

```text
GET /api/products
```

La validación real ocurre en `api-local/utils/validateToken.js`.

## Errores comunes y soluciones

### Error 501481

Se presenta cuando la autenticación no está permitida para el cliente o no se ha habilitado el flujo público/PKCE correspondiente.

Ejemplo:

```text
AADSTS501481: The request is missing a valid authorization code or refresh token.
```

Soluciones:

- Verificar que se está usando `response_type=code`
- Usar `code_challenge_method=S256`
- Confirmar `redirect_uri` exacto
- Habilitar el flujo de cliente público si aplica
- Verificar que la app está configurada como `public client` o `single-page application`

### Error 90013

Se presenta cuando el usuario no tiene permiso para la aplicación o el consentimiento no está aceptado.

Ejemplo:

```text
AADSTS90013: The app needs access to a service that your organization has not subscribed to.
```

Soluciones:

- Confirmar permisos delegados
- Conceder consentimiento administrativo si es necesario
- Revisar que el `scope` sea el correcto
- Validar que el usuario tenga acceso al tenant

### Error de token inválido en la API local

Explicación:

- El token no tiene el `aud` esperado
- El `iss` no coincide con el tenant configurado
- La llave pública no coincide con la firma
- El token está expirado

Soluciones:

- Revisar `.env`
- Usar `API_AUDIENCE` correcta
- Revisar `AZURE_JWKS_URI`
- Confirmar `AZURE_TENANT_ID` y `AZURE_CLIENT_ID`

## Pasos del laboratorio

1. Instalar dependencias:

```bash
cd api-local
npm install
```

2. Iniciar la API:

```bash
npm start
```

3. Generar PKCE:

```bash
./scripts/generate-pkce.sh
```

4. Construir la URL de autorización en Azure AD.

5. Ejecutar login y capturar el `authorization_code`.

6. Intercambiar el código por token:

```bash
export AZURE_TENANT_ID="<tenant-id>"
export AZURE_CLIENT_ID="<client-id>"
export REDIRECT_URI="http://localhost:8080/callback"
export CODE_VERIFIER="<code_verifier>"
export AUTHORIZATION_CODE="<authorization_code>"
./scripts/exchange-token.sh
```

7. Consumir la API:

```bash
export ACCESS_TOKEN="<access_token>"
./scripts/call-api.sh
```

8. Verificar la respuesta:

```json
{
  "ok": true,
  "message": "Acceso autorizado mediante OAuth2 PKCE + Azure AD.",
  "products": [ ... ]
}
```

## Buenas prácticas

- Nunca enviar secretos del cliente en un cliente público
- Guardar `code_verifier` solo en memoria o durante la sesión
- Usar `S256` siempre
- Validar JWT en la API, nunca solo en el front-end
- Mantener `redirect_uri` exacto
- Revisar permisos y consentimiento

## Conclusión

PKCE mejora significativamente la seguridad del flujo de autorización en aplicaciones públicas y móviles. Al combinarlo con validación del JWT en la API usando JWKS de Azure AD, se logra un modelo sólido para proteger recursos y permitir acceso seguro a usuarios autenticados.

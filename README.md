# API de Facturación Billivo - Guía de Integración

## Índice

1. [Introducción](#introducción)
2. [Autenticación](#autenticación)
3. [Modos de Operación](#modos-de-operación)
4. [Endpoints Disponibles](#endpoints-disponibles)
5. [Ejemplos de Uso](#ejemplos-de-uso)
6. [Estructura de Datos](#estructura-de-datos)
7. [Manejo de Errores](#manejo-de-errores)
8. [Códigos de Respuesta](#códigos-de-respuesta)
9. [Infraestructura y Seguridad](#infraestructura-y-seguridad)
10. [Mejores Prácticas](#mejores-prácticas)
11. [Soporte](#soporte)

---

## Introducción

La API de Billivo permite la creación, consulta y gestión de facturas electrónicas de forma programática. Esta API está diseñada para integrarse fácilmente en sistemas ERP, plataformas de e-commerce y aplicaciones de gestión empresarial.

### Arquitectura Tecnológica

La API está construida sobre infraestructura AWS serverless. Esta arquitectura garantiza alta disponibilidad, escalabilidad automática y seguridad empresarial.

### Características Principales

- Autenticación OAuth2
- Creación de facturas ordinarias y rectificativas
- Consulta de facturas con paginación
- Actualización de estado de facturas
- Generación automática de PDF en base64
- Soporte para múltiples tipos de IVA y descuentos
- Campos personalizables para adaptarse a necesidades específicas
- Tokens JWT (JSON Web Tokens) para autenticación segura

---

## Autenticación

La API utiliza **OAuth2** como proveedor de identidad. El flujo de autenticación sigue el estándar OAuth2 con tokens JWT.

### 1. Obtener Token de Acceso

**Endpoint**: `POST /token`

**Tipo de Autenticación**: HTTP Basic Authentication

**Headers**:

```
Authorization: Basic {base64(username:password)}
```

**Respuesta Exitosa (200)**:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "rt_eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Respuesta de Error (401)**:

```json
{
  "error": "invalid_grant",
  "error_description": "Incorrect username or password.",
  "code": "NotAuthorizedException"
}
```

### 2. Usar el Token en las Peticiones

Una vez obtenido el `access_token`, debe incluirse en todas las peticiones posteriores:

**Headers**:

```
Authorization: Bearer {access_token}
```

**Nota importante**: Los tokens expiran después de 3600 segundos (1 hora - configurable). Utiliza el `refresh_token` para obtener un nuevo token cuando sea necesario.

### 3. Detalles de Autenticación OAuth2

#### Flujo de Autenticación

La API implementa el flujo **Resource Owner Password Credentials** de OAuth2:

1. El cliente envía credenciales (username/password) mediante HTTP Basic Auth
2. Billivo valida las credenciales
3. Si son válidas, retorna:
   - `access_token`: Token JWT para autorizar peticiones a la API
   - `id_token`: Token JWT con información del usuario (claims)
   - `refresh_token`: Token para renovar el access_token sin re-autenticar
   - `expires_in`: Tiempo de expiración del token en segundos

#### Tipos de Tokens

**Access Token (JWT)**:

- Se incluye en el header `Authorization: Bearer {token}` de cada petición
- Validado automáticamente por AWS API Gateway
- Contiene los scopes y permisos del usuario
- Expira en 3600 segundos (1 hora - configurable)

**Refresh Token**:

- Permite obtener un nuevo `access_token` sin pedir credenciales nuevamente
- Mayor tiempo de expiración que el access_token
- Se debe almacenar de forma segura

---

## Modos de Operación

La API de Billivo soporta dos modos de operación según las necesidades del cliente:

### Modo ERP

**Características**:

- El cliente controla la numeración de las facturas
- Debe proporcionar el campo `invoiceNumber` en la petición
- Útil para empresas que ya tienen un sistema de numeración establecido
- Mayor control sobre la secuencia de facturas

**Cuándo usar**:

- Sistemas ERP que ya gestionan numeración de facturas
- Cuando se requiere sincronización con sistemas legacy

**Ejemplo**:

```json
{
  "header": {
    "invoiceNumber": "FAC-2025-001234",
    "invoiceType": "F1",
    ...
  }
}
```

### Modo NO_ERP

**Características**:

- Billivo gestiona automáticamente la numeración de facturas
- No es necesario incluir el campo `invoiceNumber` en la petición
- La API asigna el siguiente número disponible en la secuencia
- Numeración automática con formato `YYYY-NNNNNNNNNN` para facturas normales y `R-YYYY-NNNNNNNNNN` para rectificativas

**Cuándo usar**:

- Nuevas implementaciones sin sistema de numeración previo
- Aplicaciones que prefieren delegar la gestión de numeración
- Sistemas más simples sin requisitos específicos de numeración

**Ejemplo**:

```json
{
  "header": {
    "invoiceType": "F1",
    ...
  }
}
```

**Respuesta** (la API devuelve el número generado):

```json
{
  "invoice": {
    "header": {
      "invoiceNumber": "2025-0000000028",
      ...
    }
  }
}
```

---

## Endpoints Disponibles

### 1. Obtener Token

```
POST /token
```

Autenticación con HTTP Basic para obtener un token de acceso.

---

### 2. Crear Factura

```
POST /invoice
```

Crea una nueva factura (ordinaria o rectificativa).

**Headers**:

```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Body**: Ver [Estructura de Factura](#estructura-de-factura)

**Respuestas**:

- `201 Created`: Factura creada exitosamente
- `400 Bad Request`: Error de validación
- `401 Unauthorized`: Token inválido o ausente
- `500 Internal Server Error`: Error del servidor

---

### 3. Consultar Facturas

```
GET /invoice?userEmail={email}&invoiceNumber={numero}&page={pagina}&limit={limite}&status={estado}
```

Obtiene todas las facturas de un usuario o una factura específica.

**Parámetros Query**:

| Parámetro       | Tipo    | Requerido | Descripción                                                                                               |
| --------------- | ------- | --------- | --------------------------------------------------------------------------------------------------------- |
| `userEmail`     | string  | Sí        | Email del usuario propietario de las facturas                                                             |
| `invoiceNumber` | string  | No        | Número de factura específica a consultar                                                                  |
| `page`          | integer | No        | Número de página (por defecto: 1)                                                                         |
| `limit`         | integer | No        | Elementos por página (por defecto: 20, máximo: 100)                                                       |
| `status`        | string  | No        | Filtrar por estado: `draft`, `sent`, `paid`, `overdue`, `validated`, `validated_with_errors`, `cancelled` |

**Ejemplo de Respuesta (200)**:

```json
{
  "body": [
    {
      "header": {
        "invoiceType": "F1",
        "invoiceNumber": "2025-0000000027",
        "previousInvoiceNumber": "R-2025-0000000007",
        "dueDate": "2025-11-30",
        "currency": "EUR",
        "customer": { ... },
        "issuer": { ... }
      },
      "items": [ ... ]
    }
  ],
  "message": "GET successful",
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 34,
    "totalPages": 2
  }
}
```

---

### 4. Actualizar Estado de Factura

```
PATCH /invoice?userEmail={email}&invoiceNumber={numero}
```

Actualiza el estado de una factura existente.

**Parámetros Query**:

- `userEmail` (requerido): Email del usuario propietario
- `invoiceNumber` (requerido): Número de la factura a actualizar

**Body**:

```json
{
  "status": "sent"
}
```

**Estados válidos**:

- `sent`: Factura enviada
- `validated`: Factura validada
- `paid`: Factura pagada
- `cancelled`: Factura cancelada

**Respuestas**:

- `200 OK`: Estado actualizado correctamente
- `400 Bad Request`: Petición inválida
- `401 Unauthorized`: No autorizado
- `404 Not Found`: Factura no encontrada
- `500 Internal Server Error`: Error del servidor

---

## Ejemplos de Uso

### Ejemplo 1: Crear Factura en Modo NO_ERP (Numeración Automática)

```bash
curl -X POST https://api.billivo.com/invoice \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "header": {
      "invoiceType": "F1",
      "paymentTermName": "30 days",
      "paymentTermDays": 30,
      "paymentType": "Bank Transfer",
      "paymentTypeDescription": "ES1234567890123456789012",
      "bankName": "Santander",
      "SWIFT": "BSCHESMMXXX",
      "dueDate": "2025-12-28",
      "currency": "EUR",
      "irpf": 1,
      "customer": {
        "customerName": "ACME Corporation",
        "nationalId": "B12345678",
        "addressStreet": "Calle Principal",
        "addressNumber": "123",
        "postalCode": "08001",
        "city": "Barcelona",
        "country": "ES",
        "mailAddress": "cliente@acme.com"
      },
      "issuer": {
        "name": "Mi Empresa SL",
        "nationalId": "B99540254",
        "addressStreet": "C/ de la Diputació",
        "addressNumber": "180",
        "postalCode": "08011",
        "city": "Barcelona",
        "country": "ES",
        "vatId": "ESB99540254",
        "mailAddress": "facturacion@miempresa.com"
      },
      "taxType": "01",
      "regimeKey": "01"
    },
    "items": [
      {
        "itemId": "1",
        "name": "Servicios Profesionales",
        "type": "Service",
        "description": "Consultoría técnica",
        "unitPrice": 100,
        "quantity": 5,
        "vatPercentage": 21,
        "discountPercentage": 0,
        "totalExcTax": 500
      }
    ]
  }'
```

---

### Ejemplo 2: Crear Factura en Modo ERP (Numeración Manual)

```bash
curl -X POST https://api.billivo.com/invoice \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "header": {
      "invoiceNumber": "FAC-2025-001234",
      "invoiceType": "F1",
      "invoiceDate": "2025-09-28T14:38:15.000Z",
      "paymentTermName": "30 days",
      "paymentTermDays": 30,
      "paymentType": "Bank Transfer",
      "paymentTypeDescription": "ES1234567890123456789012",
      "dueDate": "2025-12-28",
      "currency": "EUR",
      "irpf": 0,
      "customer": {
        "customerName": "ACME Corporation",
        "nationalId": "B12345678",
        "addressStreet": "Calle Principal",
        "addressNumber": "123",
        "postalCode": "08001",
        "city": "Barcelona",
        "country": "ES",
        "mailAddress": "cliente@acme.com"
      },
      "issuer": {
        "name": "Mi Empresa SL",
        "nationalId": "B99540254",
        "addressStreet": "C/ de la Diputació",
        "addressNumber": "180",
        "postalCode": "08011",
        "city": "Barcelona",
        "country": "ES",
        "vatId": "ESB99540254",
        "mailAddress": "facturacion@miempresa.com"
      },
      "taxType": "01",
      "regimeKey": "01"
    },
    "items": [
      {
        "itemId": "1",
        "name": "Servicios Profesionales",
        "type": "Service",
        "description": "Consultoría técnica",
        "unitPrice": 100,
        "quantity": 5,
        "vatPercentage": 21,
        "discountPercentage": 0,
        "totalExcTax": 500
      }
    ]
  }'
```

---

### Ejemplo 3: Crear Factura Rectificativa

```bash
curl -X POST https://api.billivo.com/invoice \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "header": {
      "invoiceType": "R1",
      "correctedInvoiceNumber": "2025-0000000029",
      "correctiveType": "I",
      "paymentTermName": "30 days",
      "paymentTermDays": 30,
      "paymentType": "Bank Transfer",
      "paymentTypeDescription": "ES1234567890123456789012",
      "dueDate": "2025-12-28",
      "additionalNotes": "Rectificación por error en precio unitario",
      "currency": "EUR",
      "irpf": 0,
      "customer": { ... },
      "issuer": { ... },
      "taxType": "01",
      "regimeKey": "01"
    },
    "items": [
      {
        "itemId": "1",
        "name": "Servicios Profesionales",
        "type": "Service",
        "description": "Consultoría técnica",
        "unitPrice": 80,
        "quantity": 5,
        "vatPercentage": 21,
        "discountPercentage": 0,
        "totalExcTax": 400
      }
    ]
  }'
```

**Explicación del ejemplo**:

- **Factura original** (`2025-0000000029`): `unitPrice: 100, quantity: 5` → Total: 500€
- **Factura rectificativa**: Enviamos los nuevos valores reales: `unitPrice: 80, quantity: 5` → Total: 400€
- **El sistema calcula automáticamente** la diferencia de -100€ y genera la rectificativa

**Nota sobre Facturas Rectificativas**:

- `invoiceType`: Debe ser `R1` o `R5`
- `correctedInvoiceNumber`: Número de la factura que se está corrigiendo
- `correctiveType`:
  - `I`: Rectificación por diferencia (ajuste sobre la factura original)
  - `S`: Rectificación por sustitución (reemplaza completamente la factura original)
- **Importante**: Envía los valores reales (nuevos precios/cantidades) que deben quedar después de la rectificación. El sistema calcula automáticamente la diferencia con la factura original

---

### Ejemplo 4: Consultar Facturas con Filtros

```bash
# Obtener todas las facturas de un usuario
curl -X GET "https://api.billivo.com/invoice?userEmail=usuario@empresa.com" \
  -H "Authorization: Bearer {tu_token}"

# Obtener una factura específica
curl -X GET "https://api.billivo.com/invoice?userEmail=usuario@empresa.com&invoiceNumber=2025-0000000027" \
  -H "Authorization: Bearer {tu_token}"

# Consultar con paginación
curl -X GET "https://api.billivo.com/invoice?userEmail=usuario@empresa.com&page=2&limit=50" \
  -H "Authorization: Bearer {tu_token}"

# Filtrar por estado
curl -X GET "https://api.billivo.com/invoice?userEmail=usuario@empresa.com&status=paid" \
  -H "Authorization: Bearer {tu_token}"
```

---

### Ejemplo 5: Actualizar Estado de Factura

```bash
curl -X PATCH "https://api.billivo.com/invoice?userEmail=usuario@empresa.com&invoiceNumber=2025-0000000027" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "validated"
  }'
```

---

## Estructura de Datos

### Estructura de Factura

```json
{
  "header": {
    "invoiceType": "F1",
    "invoiceNumber": "2025-0000000028",
    "previousInvoiceNumber": "2025-0000000027",
    "correctedInvoiceNumber": "",
    "correctiveType": null,
    "bankName": "Santander",
    "SWIFT": "BSCHESMMXXX",
    "paymentTermName": "30 days",
    "paymentTermDays": 30,
    "paymentType": "Bank Transfer",
    "paymentTypeDescription": "ES1234567890123456789012",
    "invoiceDate": "2025-09-28T14:38:15.000Z",
    "dueDate": "2025-12-28",
    "operationDate": "2025-10-15",
    "additionalNotes": "Notas adicionales",
    "currency": "EUR",
    "irpf": 1,
    "discount": {
      "percentage": 0,
      "amount": 0,
      "currency": "EUR"
    },
    "customer": {
      "customerName": "ACME Corporation",
      "nationalId": "B12345678",
      "vatId": "",
      "addressStreet": "Calle Principal",
      "addressNumber": "123",
      "postalCode": "08001",
      "city": "Barcelona",
      "country": "ES",
      "phoneNumber": "+34612345678",
      "mailAddress": "cliente@acme.com"
    },
    "issuer": {
      "name": "Mi Empresa SL",
      "nationalId": "B99540254",
      "vatId": "ESB99540254",
      "addressStreet": "C/ de la Diputació",
      "addressNumber": "180",
      "postalCode": "08011",
      "city": "Barcelona",
      "country": "ES",
      "mailAddress": "facturacion@miempresa.com",
      "phoneNumber": "+34987654321"
    },
    "taxType": "01",
    "regimeKey": "01",
    "pos": {
      "storeId": "MAD-01",
      "terminalId": "T-07",
      "cashierId": "EMP-045",
      "shift": "afternoon"
    },
    "currencyExchange": {
      "currency": "USD",
      "baseCurrency": "EUR",
      "exchangeRate": 0.92,
      "exchangeRateDate": "2025-08-20",
      "totalAmountInBaseCurrency": 111.32
    },
    "customFields": {
      "orderReference": "PO-2025-123",
      "salesAgent": "John Doe"
    }
  },
  "items": [
    {
      "itemId": "1",
      "name": "Servicios Profesionales",
      "type": "Service",
      "description": "Consultoría técnica",
      "unitPrice": 100,
      "quantity": 5,
      "vatPercentage": 21,
      "discountPercentage": 10,
      "discountAmount": 50,
      "totalExcTax": 500,
      "customFields": {
        "productCode": "SKU-12345",
        "warranty": "2 years",
        "supplier": "ACME Corp"
      }
    }
  ]
}
```

### Tipos de Factura (invoiceType)

**Facturas Ordinarias**:

- `F1`: Factura
- `F2`: Factura simplificada (ticket)

**Facturas Rectificativas**:

- `R1`: Factura rectificativa
- `R5`: Factura rectificativa en facturas simplificadas

### Métodos de Pago (paymentType)

- `Bank Transfer`: Transferencia bancaria
- `Credit Card`: Tarjeta de crédito
- `PayPal`: PayPal
- `Cash`: Efectivo
- `Other`: Otro método

### Estados de Factura

- `draft`: Borrador
- `sent`: Enviada
- `validated`: Validada
- `paid`: Pagada
- `overdue`: Vencida
- `validated_with_errors`: Validada con errores
- `cancelled`: Cancelada

---

## Manejo de Errores

### Formato de Error

Todos los errores siguen el siguiente formato:

```json
{
  "error": "Descripción del error"
}
```

### Errores Comunes

#### Error 400 - Bad Request

**Causa**: Datos de entrada inválidos o faltantes

**Ejemplo**:

```json
{
  "error": "Validation failed: Missing required field 'customer.mailAddress'"
}
```

**Solución**: Verificar que todos los campos obligatorios estén presentes y tengan el formato correcto.

---

#### Error 401 - Unauthorized

**Causa 1**: Token ausente o inválido

```json
{
  "error": "Unauthorized - invalid or missing authentication"
}
```

**Solución**: Verificar que el header `Authorization` esté presente y contenga un token válido.

**Causa 2**: Credenciales incorrectas (al obtener token)

```json
{
  "error": "invalid_grant",
  "error_description": "Incorrect username or password.",
  "code": "NotAuthorizedException"
}
```

**Solución**: Verificar las credenciales de usuario y contraseña.

---

#### Error 404 - Not Found

**Causa**: Factura no encontrada

**Ejemplo**:

```json
{
  "error": "Invoice not found"
}
```

**Solución**: Verificar que el número de factura y el email del usuario sean correctos.

---

#### Error 500 - Internal Server Error

**Causa**: Error interno del servidor

**Ejemplo**:

```json
{
  "error": "Internal server error"
}
```

**Solución**: Contactar con el equipo de soporte de Billivo.

---

## Códigos de Respuesta

| Código | Descripción           | Significado                               |
| ------ | --------------------- | ----------------------------------------- |
| 200    | OK                    | Petición exitosa (GET, PATCH)             |
| 201    | Created               | Factura creada exitosamente (POST)        |
| 400    | Bad Request           | Error de validación en los datos enviados |
| 401    | Unauthorized          | Autenticación fallida o token inválido    |
| 404    | Not Found             | Recurso no encontrado                     |
| 500    | Internal Server Error | Error interno del servidor                |

---

## Campos Obligatorios vs Opcionales

### Header (Obligatorios)

- `invoiceType`
- `paymentTermName`
- `paymentTermDays`
- `paymentType`
- `dueDate`
- `issuer` (con campos obligatorios: `name`, `nationalId`, `city`, `country`)
- `taxType`
- `regimeKey`
- `irpf`

### Header (Opcionales)

- `invoiceNumber` (obligatorio solo en modo ERP)
- `paymentTypeDescription`
- `invoiceDate`
- `operationDate`
- `previousInvoiceNumber`
- `correctedInvoiceNumber` (obligatorio para facturas rectificativas)
- `correctiveType` (obligatorio para facturas rectificativas)
- `bankName`
- `SWIFT` (código SWIFT/BIC del banco, 8 u 11 caracteres)
- `additionalNotes`
- `currency` (por defecto: EUR)
- `discount`
- `pos`
- `currencyExchange`
- `customFields`

### Customer (Obligatorios)

- `customerName`
- `nationalId`
- `mailAddress`

### Customer (Opcionales)

- `vatId`
- `addressStreet`
- `addressNumber`
- `postalCode`
- `city`
- `country`
- `phoneNumber`

### Items (Obligatorios)

- `name`
- `unitPrice`
- `quantity`
- `vatPercentage`

### Items (Opcionales)

- `itemId`
- `description`
- `type` (Service o Article)
- `discountPercentage`
- `discountAmount`
- `totalExcTax`
- `customFields`

---

## Validaciones Importantes

### Formatos

- **Email**: Formato válido de email
- **País**: Código ISO 3166-1 alpha-2 (2 letras mayúsculas, ej: "ES")
- **Moneda**: Código ISO 4217 (3 letras mayúsculas, ej: "EUR")
- **Fecha**: Formato ISO 8601 (YYYY-MM-DD)
- **Fecha/Hora**: Formato ISO 8601 (YYYY-MM-DDTHH:mm:ss.sssZ)
- **Teléfono**: Formato E.164 (ej: "+34612345678")

### Límites

- `name` (issuer/customer): máximo 120 caracteres
- `nationalId`: máximo 20 caracteres
- `addressStreet`: máximo 120 caracteres
- `postalCode`: máximo 10 caracteres
- `city`: máximo 50 caracteres
- `mailAddress`: máximo 120 caracteres
- `phoneNumber`: máximo 30 caracteres
- `additionalNotes`: máximo 300 caracteres
- `vatPercentage`: 0-100
- `discountPercentage`: 0-100
- `customFields`: máximo 10 propiedades, cada valor máximo 255 caracteres

---

## Respuesta de Creación de Factura

Cuando se crea una factura exitosamente, la API devuelve:

```json
{
  "invoice": {
    "header": { ... },
    "items": [ ... ]
  },
  "base64pdf": "data:application/pdf;filename=generated.pdf;base64,JVBERi0xLjQKJeLjz9MKMSAwIG9iago8PC9..."
}
```

El campo `base64pdf` contiene el PDF de la factura codificado en base64, listo para ser guardado o enviado al cliente.

---

## Mejores Prácticas

### 1. Gestión de Tokens

- Almacena el `access_token` de forma segura
- Implementa lógica para renovar el token antes de que expire
- Utiliza el `refresh_token` para obtener nuevos tokens sin re-autenticar

### 2. Manejo de Errores

- Implementa reintentos con backoff exponencial para errores 5xx
- Valida los datos antes de enviarlos a la API
- Registra todos los errores para análisis posterior

### 3. Paginación

- Utiliza siempre paginación al consultar listas de facturas
- El límite máximo es 100 elementos por página
- Implementa carga incremental para mejorar la experiencia de usuario

### 4. Numeración de Facturas

- En modo NO_ERP: No incluyas `invoiceNumber` en la petición
- En modo ERP: Asegúrate de que la numeración sea única y secuencial
- Guarda siempre el `invoiceNumber` devuelto por la API

### 5. Validación de Datos

- Verifica que todos los campos obligatorios estén presentes
- Valida los formatos (email, país, moneda, fechas)
- Respeta los límites de caracteres establecidos

### 6. Facturas Rectificativas

- Siempre incluye `correctedInvoiceNumber` y `correctiveType`
- Envía los valores reales (nuevos precios/cantidades) que deben quedar después de la rectificación
- El sistema calculará automáticamente la diferencia con la factura original
- Documenta claramente el motivo de la rectificación en `additionalNotes`

---

## Infraestructura y Seguridad

### Arquitectura Serverless

La API está desplegada utilizando servicios administrados de AWS:

**API Gateway (HTTP API)**:

- Enrutamiento automático de peticiones
- Validación de tokens JWT integrada
- CORS configurado para integraciones web
- Límites de tasa (rate limiting) configurables
- Logs de acceso y métricas en CloudWatch

**AWS Lambda**:

- Funciones serverless para cada endpoint:
  - `c10-egrow-token`: Gestión de autenticación OAuth2
  - `c10-egrow-invoices`: Operaciones CRUD de facturas
  - `c10-egrow-v1`: Endpoints versión 1
- Escalado automático según demanda
- Sin gestión de servidores
- Timeout configurado para operaciones largas

**AWS Cognito User Pool**:

- Gestión completa de usuarios
- Políticas de contraseña configurables
- MFA (Multi-Factor Authentication) disponible
- Pool de usuarios segregado por cliente

### Seguridad

**Autenticación y Autorización**:

- OAuth2 con JWT tokens
- Tokens firmados con algoritmo RS256
- Validación automática en cada petición
- Segregación de usuarios por cliente

**Encriptación**:

- TLS 1.2+ para todas las comunicaciones
- Datos en tránsito encriptados
- Secrets gestionados en AWS Secrets Manager
- Variables de entorno encriptadas en Lambda

**Auditoría y Logs**:

- Logs de acceso en CloudWatch
- Trazabilidad completa de operaciones
- Retención de logs configurable por ambiente:
  - Desarrollo: 90 días
  - Producción: 7 días
  - Demo: 1 día

---

## Soporte

Para dudas, problemas o sugerencias sobre la API, contacta con el equipo de Billivo:

- **Email**: equipo@billivo.com
- **Documentación**: Consulta el archivo `api-definition.json` para la especificación completa OpenAPI
- **Swagger UI**: Abre `index.html` en un navegador para explorar la API de forma interactiva

---

**Última actualización**: Noviembre 2025

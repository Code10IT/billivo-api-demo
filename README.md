# API de Billivo - Guía de Integración

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

La API de Billivo permite crear, consultar y gestionar facturas electrónicas, gestionar las series de numeración y leer los gastos de Billivo Gastos de forma programática. Esta API está diseñada para integrarse fácilmente en sistemas ERP, plataformas de e-commerce y aplicaciones de gestión empresarial.

### Arquitectura Tecnológica

La API está construida sobre infraestructura AWS serverless. Esta arquitectura garantiza alta disponibilidad, escalabilidad automática y seguridad empresarial.

### Características Principales

- Autenticación OAuth2
- Creación de facturas ordinarias y rectificativas
- Consulta de facturas con paginación
- Actualización de estado de facturas
- Generación automática de PDF en base64, en cinco idiomas
- Envío de la factura por email al cliente
- Series de numeración propias (modo NO_ERP)
- Recargo de equivalencia
- Exportación de gastos de Billivo Gastos y sincronización con el ERP
- Soporte para múltiples tipos de IVA y descuentos
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

Sin el header `Authorization: Basic`, la API responde `401` con `error: "invalid_request"`.

### 2. Usar el Token en las Peticiones

Una vez obtenido el `access_token`, debe incluirse en todas las peticiones posteriores:

**Headers**:

```
Authorization: Bearer {access_token}
```

**Nota importante**: Los tokens expiran después de 3600 segundos (1 hora - configurable). La API no tiene un endpoint para canjear el `refresh_token`. Para renovar el token, llama otra vez a `POST /token` con las mismas credenciales Basic.

### 3. Detalles de Autenticación OAuth2

#### Flujo de Autenticación

La API implementa el flujo **Resource Owner Password Credentials** de OAuth2:

1. El cliente envía credenciales (username/password) mediante HTTP Basic Auth
2. Billivo valida las credenciales
3. Si son válidas, retorna:
   - `access_token`: Token JWT para autorizar peticiones a la API
   - `id_token`: Token JWT con información del usuario (claims)
   - `refresh_token`: La API lo devuelve, pero no tiene un endpoint para canjearlo
   - `expires_in`: Tiempo de expiración del token en segundos

#### Tipos de Tokens

**Access Token (JWT)**:

- Se incluye en el header `Authorization: Bearer {token}` de cada petición
- Validado automáticamente por AWS API Gateway
- Contiene los scopes y permisos del usuario
- Expira en 3600 segundos (1 hora - configurable)

**Refresh Token**:

- `POST /token` lo devuelve, pero la API no tiene un endpoint para canjearlo
- Para renovar el `access_token`, llama otra vez a `POST /token` con las mismas credenciales Basic

---

## Modos de Operación

La API de Billivo soporta dos modos de operación según las necesidades del cliente:

### Modo ERP

**Características**:

- El cliente controla la numeración de las facturas
- Debe proporcionar el campo `invoiceNumber` en la petición
- Útil para empresas que ya tienen un sistema de numeración establecido
- Mayor control sobre la secuencia de facturas
- La API rechaza un `invoiceNumber` que ya está en una factura registrada de la empresa
- No se puede enviar `formatId`

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
- La API asigna el siguiente número de una serie de la empresa (ver [Series de Numeración](#5-listar-series-de-numeración))
- Con `header.formatId` eliges la serie. Sin él, Billivo usa la serie por defecto del tipo: la serie de tipo `F` para facturas y la de tipo `C` para rectificativas
- Son las mismas series y contadores que usa la app de Billivo

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

**Parámetros Query** (opcionales):

| Parámetro | Tipo    | Descripción                                                                                                                                        |
| --------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `email`   | boolean | Con `true`, Billivo envía el PDF a `customer.mailAddress`. No aplica a facturas F2 y R5 ni a clientes sin email                                    |
| `lang`    | string  | Idioma del PDF: `es`, `en`, `ca`, `gl`, `eu`. Sin él se usa el idioma de la empresa. Un cliente de fuera de España siempre recibe el PDF en inglés |

**Headers**:

```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Body**: Ver [Estructura de Factura](#estructura-de-factura)

**Respuestas**:

- `201 Created`: Factura creada exitosamente
- `400 Bad Request`: Campos obligatorios ausentes o inválidos, o `dueDate` anterior a hoy. El campo `error` reúne todos los problemas en un solo texto
- `401 Unauthorized`: Token inválido o ausente, o `issuer.mailAddress` de otra empresa
- `500 Internal Server Error`: Error del servidor. También si la factura no se puede registrar: número ERP ya registrado, serie `formatId` inexistente o de otro tipo, o empresa sin serie por defecto. El campo `error` indica el motivo

---

### 3. Consultar Facturas

```
GET /invoice?userEmail={email}&invoiceNumber={numero}&page={pagina}&limit={limite}&status={estado}&pdf={true|false}
```

Obtiene todas las facturas de un usuario o una factura específica.

**Parámetros Query**:

| Parámetro       | Tipo    | Requerido | Descripción                                                                                                        |
| --------------- | ------- | --------- | ------------------------------------------------------------------------------------------------------------------ |
| `userEmail`     | string  | Sí        | Email del usuario propietario de las facturas                                                                      |
| `invoiceNumber` | string  | No        | Número de factura específica a consultar                                                                           |
| `page`          | integer | No        | Número de página (por defecto: 1)                                                                                  |
| `limit`         | integer | No        | Elementos por página (por defecto: 20, máximo: 100)                                                                |
| `status`        | string  | No        | Filtrar por estado. Ver [Estados de Factura](#estados-de-factura). Un valor no reconocido devuelve `400`           |
| `pdf`           | boolean | No        | Con `true`, cada factura incluye su PDF en `base64pdf` (base64 sin prefijo `data:`). Sin él, `base64pdf` es `null` |

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
      "items": [ ... ],
      "base64pdf": null,
      "status": "sent"
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

Cada factura incluye su estado actual en `status`. Si ninguna factura coincide con los filtros, `body` es una lista vacía.

---

### 4. Actualizar Estado de Factura

```
PATCH /invoice?userEmail={email}&invoiceNumber={numero}
```

Marca una factura como pagada, o deshace el pago. Solo se aplica a facturas en estado `sent` o `paid`.

**Parámetros Query**:

- `userEmail` (requerido): Email del usuario propietario
- `invoiceNumber` (requerido): Número de la factura a actualizar

**Body**:

```json
{
  "status": "paid"
}
```

**Estados válidos**:

- `paid`: Marca la factura como pagada
- `sent`: Deshace el pago. La factura vuelve a `sent`

**Ejemplo de Respuesta (200)**:

```json
{
  "body": {
    "header": { ... },
    "items": [ ... ],
    "status": "paid"
  },
  "message": "Invoice status updated successfully",
  "status": "paid"
}
```

**Respuestas**:

- `200 OK`: Estado actualizado correctamente
- `400 Bad Request`: Estado no aceptado o falta `invoiceNumber`
- `401 Unauthorized`: No autorizado
- `404 Not Found`: Factura no encontrada
- `409 Conflict`: La factura no está en `sent` ni en `paid`. El campo `error` indica su estado actual
- `500 Internal Server Error`: Error del servidor

---

### 5. Listar Series de Numeración

```
GET /invoice/series
```

Devuelve las series de la empresa del token. Solo las usan las empresas en modo NO_ERP.

**Ejemplo de Respuesta (200)**:

```json
{
  "series": [
    {
      "id": 1,
      "name": "General",
      "type": "F",
      "default": true,
      "invoicePrefix": "",
      "invoiceYear": 4,
      "invoiceDigits": 10,
      "firstSeparator": "",
      "secondSeparator": "-"
    },
    {
      "id": 3,
      "name": "Tienda online",
      "type": "F",
      "default": false,
      "invoicePrefix": "WEB",
      "invoiceYear": 4,
      "invoiceDigits": 5,
      "firstSeparator": "-",
      "secondSeparator": "/"
    }
  ]
}
```

- `type`: `F` numera facturas (F1, F2) y `C` numera rectificativas (R1, R5)
- `default`: la serie que Billivo usa cuando la factura no tiene `formatId`
- Formato del número: prefijo, primer separador, año, segundo separador y contador. La serie `Tienda online` genera `WEB-2026/00001`

---

### 6. Crear Serie de Numeración

```
POST /invoice/series
```

**Body**:

```json
{
  "name": "Tienda online",
  "type": "F",
  "invoicePrefix": "WEB",
  "invoiceYear": 4,
  "invoiceDigits": 5,
  "firstSeparator": "-",
  "secondSeparator": "/",
  "firstNumber": 1
}
```

| Campo             | Obligatorio | Descripción                                                                      |
| ----------------- | ----------- | -------------------------------------------------------------------------------- |
| `name`            | Sí          | Máximo 100 caracteres                                                            |
| `type`            | Sí          | `F` (facturas) o `C` (rectificativas)                                            |
| `invoicePrefix`   | Sí          | Máximo 4 caracteres. Puede ser vacío                                             |
| `invoiceYear`     | Sí          | Dígitos del año: `0` (sin año), `2` o `4`                                        |
| `invoiceDigits`   | Sí          | Dígitos del contador, de 1 a 15                                                  |
| `firstSeparator`  | No          | Un carácter o vacío. Va entre el prefijo y el año                                |
| `secondSeparator` | No          | Un carácter o vacío. Va entre el año y el contador                               |
| `firstNumber`     | No          | Número de la primera factura (por defecto 1). Útil para continuar una numeración |

La primera serie de un tipo pasa a ser la serie por defecto de ese tipo. Una serie con año vuelve a empezar en 1 cada año.

**Respuestas**:

- `201 Created`: `{ "series": { ... } }`
- `400 Bad Request`: `{ "error": [ ... ] }`, un mensaje por campo inválido
- `401 Unauthorized`: No autorizado
- `409 Conflict`: Otra serie de la empresa ya genera números con el mismo prefijo, año y separadores

---

### 7. Billivo Gastos: Exportar Gastos

Los endpoints `/expenses/*` leen los gastos de Billivo Gastos de la empresa del token. La empresa debe tener acceso a Billivo Gastos, si no la API responde `403`. Solo se devuelven gastos confirmados y no eliminados.

```
GET /expenses/export?from={YYYY-MM-DD}&to={YYYY-MM-DD}&format=json&page={pagina}&limit={limite}
```

| Parámetro       | Tipo    | Requerido | Descripción                                                            |
| --------------- | ------- | --------- | ---------------------------------------------------------------------- |
| `from`          | date    | Sí        | Primera fecha de gasto                                                 |
| `to`            | date    | Sí        | Última fecha de gasto (incluida)                                       |
| `format`        | string  | No        | `json` devuelve una página de gastos. Sin él, devuelve un fichero ZIP  |
| `includeImages` | boolean | No        | Solo ZIP. Con `true`, el ZIP incluye los tickets y facturas escaneados |
| `page`          | integer | No        | Solo JSON. Por defecto 1                                               |
| `limit`         | integer | No        | Solo JSON. Por defecto 100, máximo 500                                 |

**Respuesta ZIP (200)**: un enlace que caduca a los 5 minutos. El ZIP contiene un Excel con los gastos.

```json
{
  "url": "https://...",
  "fileName": "expenses_2026-01-01_2026-03-31.zip"
}
```

**Respuesta JSON (200)**:

```json
{
  "page": 1,
  "limit": 100,
  "total": 2,
  "expenses": [ { ... } ]
}
```

Ver [Estructura de Gasto](#estructura-de-gasto).

---

### 8. Billivo Gastos: Sincronización con el ERP

```
GET /expenses/erp?after={expenseId}&limit={limite}
POST /expenses/erp
```

Flujo recomendado:

1. `GET /expenses/erp` devuelve los gastos confirmados que ningún ERP ha registrado todavía, ordenados por `expenseId`. `limit` es 100 por defecto y 500 como máximo
2. Para la página siguiente, envía `after` con el último `expenseId` de la página. Un array `expenses` vacío indica que no quedan más
3. Registra los gastos en el ERP
4. `POST /expenses/erp` con el id que cada gasto recibió en el ERP

**Respuesta GET (200)**:

```json
{
  "limit": 100,
  "expenses": [ { ... } ]
}
```

**Body POST** (de 1 a 500 elementos, `erpId` de 1 a 100 caracteres, texto o número):

```json
{
  "confirmed": [
    { "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77", "erpId": "FAC-88" }
  ]
}
```

**Respuesta POST (200)**:

```json
{
  "confirmed": ["8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77"],
  "rejected": []
}
```

- Un gasto confirmado sale del `GET /expenses/erp` y ya no se puede editar ni eliminar en Billivo Gastos
- `rejected` lista los gastos que no existen, no están confirmados, están eliminados o ya tienen otro `erpId`
- Enviar otra vez el mismo `erpId` es válido, así que puedes repetir una llamada que falló por timeout

---

### 9. Billivo Gastos: Categorías

```
GET /expenses/categories
```

```json
{
  "categories": [
    { "categoryId": "transport", "categoryName": "Transporte", "active": true },
    { "categoryId": "meals", "categoryName": "Comidas", "active": true }
  ]
}
```

Un `categoryId` nunca cambia y una categoría no se elimina, solo se desactiva (`active: false`). El ERP puede asociar cada `categoryId` a una cuenta contable.

---

### 10. Billivo Gastos: Métodos de Pago

```
GET /expenses/paymentMethods
```

```json
{
  "paymentMethods": [
    { "paymentMethodId": "2", "paymentMethodName": "Tarjeta de crédito", "active": true, "markAsPaid": true },
    { "paymentMethodId": "4", "paymentMethodName": "Efectivo", "active": true, "markAsPaid": false }
  ]
}
```

Si `markAsPaid` es `true`, un gasto que recibe ese método de pago queda marcado como pagado (tiene `paidAt`).

Funciona igual que las categorías: un `paymentMethodId` nunca cambia y un método de pago no se elimina, solo se desactiva (`active: false`). El ERP puede asociar cada `paymentMethodId` a su propia forma de pago. Es el valor de `paymentMethodId` en cada gasto.

---

### 11. Billivo Gastos: Cambiar el Estado de un Gasto

```
PATCH /expenses/status
```

Marca un gasto como pagado o no pagado, o revisa un gasto pendiente (lo confirma o lo rechaza). Cada llamada hace un solo cambio: envía `paid` o `reviewStatus`, no los dos.

**Body para el pago**:

```json
{
  "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77",
  "paid": true
}
```

- `paid: true` marca el gasto como pagado. `paid: false` deshace el pago
- Se permite en cualquier estado del gasto, también si ya está en el ERP o en un período cerrado

**Body para la revisión**:

```json
{
  "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77",
  "reviewStatus": "rejected",
  "rejectionReason": "Ticket duplicado"
}
```

- `reviewStatus: "confirmed"` confirma el gasto. Solo los gastos confirmados salen en `GET /expenses/erp` y en `GET /expenses/export`
- `reviewStatus: "rejected"` rechaza el gasto. `rejectionReason` es obligatorio (máximo 500 caracteres). Solo se pueden rechazar los gastos que subió un empleado
- Solo se puede revisar un gasto pendiente. Una revisión es definitiva: un gasto confirmado o rechazado no vuelve a pendiente
- No se puede revisar un gasto que ya está en el ERP o en un período cerrado

**Respuesta (200)**: el gasto actualizado. Los campos del estado son:

```json
{
  "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77",
  "reviewStatus": "rejected",
  "reviewedBy": "empresa@ejemplo.com",
  "reviewedAt": "2026-10-19T10:30:00.000Z",
  "rejectionReason": "Ticket duplicado",
  "paidAt": null
}
```

`paidAt` es la fecha del pago, o `null` si el gasto no está pagado.

**Respuestas**:

- `200 OK`: Estado actualizado correctamente
- `400 Bad Request`: Body no válido. También `REJECTION_REASON_REQUIRED` (falta el motivo) y `ONLY_EMPLOYEE_EXPENSES_CAN_BE_REJECTED` (el gasto no lo subió un empleado)
- `401 Unauthorized`: No autorizado
- `403 Forbidden`: La empresa no tiene acceso a Billivo Gastos
- `404 Not Found`: Gasto no encontrado
- `409 Conflict`: `EXPENSE_ALREADY_REVIEWED` (el gasto no está pendiente), `EXPENSE_IN_ERP` o `EXPENSE_IN_CLOSED_PERIOD`
- `502 Bad Gateway`: Falló la petición a Billivo Gastos

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
      "dueDate": "2030-12-31",
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
        "discountAmount": 0,
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
      "paymentTermName": "30 days",
      "paymentTermDays": 30,
      "paymentType": "Bank Transfer",
      "paymentTypeDescription": "ES1234567890123456789012",
      "bankName": "Santander",
      "dueDate": "2030-12-31",
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
        "discountAmount": 0,
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
      "bankName": "Santander",
      "dueDate": "2030-12-31",
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
        "discountAmount": 0,
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

# Obtener una factura específica (su estado actual está en body[0].status)
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

### Ejemplo 5: Marcar una Factura como Pagada

```bash
curl -X PATCH "https://api.billivo.com/invoice?userEmail=usuario@empresa.com&invoiceNumber=2025-0000000027" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "paid"
  }'
```

---

### Ejemplo 6: Crear Factura con una Serie y Enviarla por Email

```bash
# 1. Consultar las series para obtener el id
curl -X GET "https://api.billivo.com/invoice/series" \
  -H "Authorization: Bearer {tu_token}"

# 2. Crear la factura con formatId, enviarla al cliente y generar el PDF en catalán
curl -X POST "https://api.billivo.com/invoice?email=true&lang=ca" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "header": {
      "invoiceType": "F1",
      "formatId": 3,
      ...
    },
    "items": [ ... ]
  }'
```

---

### Ejemplo 7: Factura con Recargo de Equivalencia

```json
"items": [
  {
    "name": "Material de oficina",
    "type": "Article",
    "unitPrice": 500,
    "quantity": 1,
    "vatPercentage": 21,
    "discountPercentage": 0,
    "discountAmount": 0,
    "totalExcTax": 500,
    "equivalenceSurcharge": 5.2,
    "equivalenceSurchargeTax": 26
  }
]
```

---

### Ejemplo 8: Sincronizar Gastos con el ERP

```bash
# 1. Leer la primera página de gastos pendientes
curl -X GET "https://api.billivo.com/expenses/erp?limit=100" \
  -H "Authorization: Bearer {tu_token}"

# 2. Leer la página siguiente
curl -X GET "https://api.billivo.com/expenses/erp?limit=100&after=8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77" \
  -H "Authorization: Bearer {tu_token}"

# 3. Confirmar los gastos registrados en el ERP
curl -X POST "https://api.billivo.com/expenses/erp" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{ "confirmed": [ { "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77", "erpId": "FAC-88" } ] }'
```

---

### Ejemplo 9: Cambiar el Estado de un Gasto

```bash
# Marcar un gasto como pagado
curl -X PATCH "https://api.billivo.com/expenses/status" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{ "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77", "paid": true }'

# Confirmar un gasto pendiente
curl -X PATCH "https://api.billivo.com/expenses/status" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{ "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77", "reviewStatus": "confirmed" }'

# Rechazar un gasto pendiente
curl -X PATCH "https://api.billivo.com/expenses/status" \
  -H "Authorization: Bearer {tu_token}" \
  -H "Content-Type: application/json" \
  -d '{ "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77", "reviewStatus": "rejected", "rejectionReason": "Ticket duplicado" }'
```

---

## Estructura de Datos

### Estructura de Factura

```json
{
  "header": {
    "invoiceType": "F1",
    "invoiceNumber": "2025-0000000028",
    "formatId": null,
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
    "dueDate": "2030-12-31",
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
      "equivalenceSurcharge": 0,
      "equivalenceSurchargeTax": 0,
      "calificacionOperacion": "S1",
      "operacionExenta": null,
      "customFields": {
        "productCode": "SKU-12345",
        "warranty": "2 years",
        "supplier": "ACME Corp"
      }
    }
  ]
}
```

**Nota**: La API acepta `header.discount`, `header.pos`, `header.currencyExchange`, `header.customFields` e `items[].customFields`, pero no los guarda ni los devuelve. La excepción es `customFields.orderReference`, que sí se guarda y se devuelve. Los descuentos por línea (`discountPercentage`, `discountAmount`) sí se aplican. Ver [Factura Devuelta por la API](#factura-devuelta-por-la-api).

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
- `Other`: Otro método. `Others` se acepta como sinónimo

Cualquier otro valor devuelve `400`.

### Estados de Factura

El campo `status` de una factura y el filtro `status` de `GET /invoice` usan estos valores:

- `draft`: Borrador
- `proforma`: Proforma
- `validating`: En proceso de validación
- `sent`: Registrada en la AEAT
- `validated_with_errors`: Aceptada por la AEAT con errores
- `paid`: Pagada
- `overdue`: Vencida
- `cancelled`: Anulada
- `rectified`: Rectificada por una factura rectificativa
- `rectifying`: En proceso de rectificación
- `error`: Rechazada por la AEAT

Una factura creada con `POST /invoice` queda en `sent` o en `validated_with_errors`.

El filtro `status=overdue` se calcula con la fecha de vencimiento: devuelve las facturas sin pagar cuyo `dueDate` ya pasó. El campo `status` de esas facturas puede seguir en `sent`.

### Tipos de Impuesto (taxType)

- `01`: IVA
- `02`: IPSI (Ceuta y Melilla)
- `03`: IGIC (Canarias)
- `05`: Otro

### Estructura de Gasto

Los gastos de `GET /expenses/export?format=json` y `GET /expenses/erp` tienen esta forma. Los importes son números y las fechas tienen formato `YYYY-MM-DD`.

```json
{
  "expenseId": "8f14e45f-ceea-4a7a-9c1d-0b5e2a1f3c77",
  "title": "Taxi aeropuerto",
  "expenseDate": "2026-03-12",
  "operationDate": null,
  "totalAmount": 121,
  "currency": "EUR",
  "taxBase": 100,
  "vatAmount": 21,
  "vatPercent": 21,
  "irpfPercent": 0,
  "irpfAmount": 0,
  "invoiceNumber": "T-4411",
  "merchantName": "Taxi Barcelona SL",
  "merchantTaxId": "B12345678",
  "categoryId": "transport",
  "categoryName": "Transporte",
  "paymentMethodId": "2",
  "paymentMethodName": "Tarjeta",
  "documentTypeName": "Factura",
  "erpId": null,
  "vatLines": [{ "vatPercent": 21, "taxBase": 100, "vatAmount": 21 }],
  "receipt": {
    "fileName": "taxi.pdf",
    "mimeType": "application/pdf",
    "url": "https://..."
  }
}
```

- `vatLines`: desglose de IVA, una línea por tipo
- `irpfPercent` e `irpfAmount`: tipo (%) y cuota de la retención de IRPF, calculada sobre `taxBase`. Sin retención, los dos valen `0`. `totalAmount` es base más IVA, así que el importe a pagar es `totalAmount - irpfAmount`
- `receipt.url`: enlace al ticket o factura escaneada. Caduca a los 5 minutos
- `erpId`: el id que el ERP envió en `POST /expenses/erp`
- La respuesta incluye además otros campos del gasto (dirección del emisor y del receptor, descuento, país, fechas de alta y modificación)

---

## Manejo de Errores

### Formato de Error

La mayoría de los errores siguen el siguiente formato:

```json
{
  "error": "Descripción del error"
}
```

Excepciones: el `401` por token ausente o inválido devuelve `{ "message": "Unauthorized" }`, y el `400` de `POST /invoice/series` devuelve `error` como una lista.

### Errores Comunes

#### Error 400 - Bad Request

**Causa**: Datos de entrada inválidos o faltantes

**Ejemplo** (`POST /invoice` devuelve el mensaje en español y reúne todos los problemas en un solo texto):

```json
{
  "error": "No se encuentran los siguientes campos obligatorios o son inválidos: header.bankName | Errores en líneas: items[0].discountAmount ausente"
}
```

**Solución**: Verificar que todos los campos obligatorios estén presentes y tengan el formato correcto.

---

#### Error 401 - Unauthorized

**Causa 1**: Token ausente o inválido. El API Gateway rechaza la petición antes de que llegue a Billivo, y la respuesta no usa el formato `error`

```json
{
  "message": "Unauthorized"
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

**Causa 3**: El usuario de `userEmail` o de `issuer.mailAddress` no pertenece a la empresa del token, la empresa no tiene acceso a la API, o el usuario es de tipo Agent o Guest. El campo `error` indica el motivo

```json
{
  "error": "User cannot use the API."
}
```

**Solución**: Usar el email de un usuario de Billivo de la misma empresa que el token, con acceso a la API.

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

| Código | Descripción           | Significado                                 |
| ------ | --------------------- | ------------------------------------------- |
| 200    | OK                    | Petición exitosa (GET, PATCH)               |
| 201    | Created               | Factura creada exitosamente (POST)          |
| 400    | Bad Request           | Error de validación en los datos enviados   |
| 401    | Unauthorized          | Autenticación fallida o token inválido      |
| 403    | Forbidden             | La empresa no tiene acceso a Billivo Gastos |
| 404    | Not Found             | Recurso no encontrado                       |
| 409    | Conflict              | La serie ya existe con el mismo formato     |
| 500    | Internal Server Error | Error interno del servidor                  |
| 502    | Bad Gateway           | Falló la petición a Billivo Gastos          |

---

## Campos Obligatorios vs Opcionales

### Header (Obligatorios)

- `invoiceType`
- `paymentTermName`
- `paymentTermDays` (la API no lo devuelve en las respuestas)
- `paymentType`
- `dueDate` (no puede ser anterior a hoy, fecha de España, ni anterior a `invoiceDate`)
- `customer` (excepto en facturas `F2` y `R5`)
- `issuer` (con campos obligatorios: `name`, `nationalId`, `mailAddress`; `city` y `country` son opcionales)
- `taxType`
- `regimeKey`

`issuer.mailAddress` debe ser el email de un usuario de Billivo de la misma empresa que el token: identifica la empresa que emite la factura. Si pertenece a otra empresa, la API responde `401`.

### Header (Opcionales)

- `invoiceNumber` (obligatorio en modo ERP, no permitido en modo NO_ERP)
- `formatId` (solo modo NO_ERP, id de `GET /invoice/series`)
- `paymentTypeDescription`
- `invoiceDate`
- `operationDate`
- `previousInvoiceNumber`
- `correctedInvoiceNumber` (obligatorio para facturas rectificativas)
- `correctiveType` (obligatorio para facturas rectificativas)
- `bankName` (obligatorio si `paymentType` es `Bank Transfer`)
- `SWIFT` (código SWIFT/BIC del banco, 8 u 11 caracteres)
- `additionalNotes`
- `currency` (por defecto: EUR)
- `irpf` (valores aceptados: `0`, `1`, `7`, `15`; por defecto `0`)
- `discount` (la API lo acepta, pero no lo guarda ni lo devuelve)
- `pos` (la API lo acepta, pero no lo guarda ni lo devuelve)
- `currencyExchange` (la API lo acepta, pero no lo guarda ni lo devuelve)
- `customFields` (la API solo guarda y devuelve `orderReference`)

### Customer (Obligatorios)

- `customerName`
- `nationalId`

### Customer (Opcionales)

- `mailAddress` (necesario para `email=true`)
- `vatId`
- `addressStreet`
- `addressNumber`
- `postalCode`
- `city`
- `country`
- `phoneNumber`

### Items (Obligatorios)

- `name`
- `type` (`Service` o `Article`)
- `unitPrice`
- `quantity`
- `vatPercentage`
- `discountPercentage`
- `discountAmount`
- `totalExcTax`

### Items (Opcionales)

- `itemId`
- `description`
- `equivalenceSurcharge` (tipo del recargo de equivalencia, 0-100)
- `equivalenceSurchargeTax` (cuota del recargo; se envían los dos o ninguno)
- `calificacionOperacion` (`S1` sujeta, `S2` inversión del sujeto pasivo, `N1`/`N2` no sujeta; por defecto `S1`)
- `operacionExenta` (causa de exención de Veri*Factu, por ejemplo `E1`)
- `customFields` (la API lo acepta, pero no lo guarda ni lo devuelve)

---

## Validaciones Importantes

Al recibir la factura, la API comprueba:

- Los campos obligatorios
- Los valores permitidos de `invoiceType`, `paymentType`, `taxType`, `regimeKey`, `irpf`, `correctiveType` y `type` de cada línea
- `vatPercentage` mayor o igual que 0
- `equivalenceSurcharge` entre 0 y 100
- `dueDate` no anterior a hoy (fecha de España) ni anterior a `invoiceDate`
- Un máximo de 9999999999.99 para el precio unitario, la cantidad y el total de cada línea

Los formatos y límites siguientes son los esperados. La API no los comprueba al recibir la factura, pero un valor fuera de rango puede hacer que el registro falle o que la AEAT rechace la factura.

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

---

## Respuesta de Creación de Factura

Cuando se crea una factura exitosamente, la API devuelve:

```json
{
  "invoice": {
    "header": { ... },
    "items": [ ... ],
    "status": "sent"
  },
  "base64pdf": "data:application/pdf;filename=generated.pdf;base64,JVBERi0xLjQKJeLjz9MKMSAwIG9iago8PC9...",
  "emailSent": false
}
```

El campo `invoice.status` contiene el estado de la factura: `sent` si la AEAT la registró, o `validated_with_errors` si la aceptó con errores. Ver [Estados de Factura](#estados-de-factura).

Para conocer más adelante el estado de una factura ya enviada, usa `GET /invoice?userEmail={email}&invoiceNumber={numero}`. Es una consulta de solo lectura: no reenvía la factura ni crea duplicados.

El campo `base64pdf` contiene el PDF de la factura codificado en base64, listo para ser guardado o enviado al cliente. Billivo también guarda el PDF, así que `GET /invoice?pdf=true` lo devuelve después.

El campo `emailSent` es `true` si Billivo envió la factura al cliente (`email=true`). Un error en el email no hace fallar la petición: la factura ya está creada.

### Factura Devuelta por la API

La factura que devuelven `POST /invoice` (`invoice`), `GET /invoice` y `PATCH /invoice` (`body`) tiene menos campos que la petición:

- **header**: `invoiceType`, `invoiceNumber`, `previousInvoiceNumber`, `correctedInvoiceNumber`, `correctiveType`, `bankName`, `SWIFT`, `paymentTermName`, `paymentType`, `paymentTypeDescription`, `dueDate`, `additionalNotes` (el texto `No additional notes` si no se envió ninguna), `currency`, `irpf`, `discount` (siempre `0`), `customer`, `issuer`, `taxType`, `regimeKey`, `operationDate`, `customFields.orderReference`
- **header, no se devuelven**: `invoiceDate`, `formatId`, `paymentTermDays`, `pos`, `currencyExchange`
- **items**: `itemId`, `name`, `type`, `description`, `unitPrice`, `quantity`, `vatPercentage`, `taxId`, `discountPercentage`, `discountAmount`, `totalExcTax`, `calificacionOperacion`, `operacionExenta`
- **items, no se devuelven**: `equivalenceSurcharge`, `equivalenceSurchargeTax`, `customFields`
- Además, cada factura incluye `status` y `base64pdf` en su nivel superior

---

## Mejores Prácticas

### 1. Gestión de Tokens

- Almacena el `access_token` de forma segura
- Implementa lógica para renovar el token antes de que expire
- Para renovar, llama otra vez a `POST /token` con las mismas credenciales Basic. La API no tiene un endpoint para canjear el `refresh_token`

### 2. Manejo de Errores

- Implementa reintentos con backoff exponencial para errores 5xx
- Valida los datos antes de enviarlos a la API
- Registra todos los errores para análisis posterior

### 3. Paginación

- Utiliza siempre paginación al consultar listas de facturas
- El límite máximo es 100 elementos por página
- Implementa carga incremental para mejorar la experiencia de usuario

### 4. Numeración de Facturas

- En modo NO_ERP: No incluyas `invoiceNumber` en la petición. Usa `formatId` para elegir la serie
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

### 7. Sincronización de Gastos

- Pagina `GET /expenses/erp` con `after`, no con número de página: la lista de pendientes cambia mientras confirmas
- Confirma cada página con `POST /expenses/erp` antes de pedir la siguiente
- Descarga el ticket de `receipt.url` enseguida, el enlace caduca a los 5 minutos

---

## Infraestructura y Seguridad

### Arquitectura Serverless

La API está desplegada utilizando servicios administrados de AWS:

**API Gateway (HTTP API)**:

- Enrutamiento automático de peticiones
- Validación de tokens JWT integrada
- CORS configurado para integraciones web
- Logs de acceso y métricas en CloudWatch

**AWS Lambda**:

- Funciones serverless para cada endpoint:
  - `c10-egrow-token`: Gestión de autenticación OAuth2
  - `c10-egrow-invoices`: Facturas, series de numeración y gastos
  - `c10-egrow-v1`: Endpoints versión 1
- Escalado automático según demanda
- Sin gestión de servidores
- Timeout configurado para operaciones largas

**AWS Cognito User Pool**:

- Gestión completa de usuarios
- Políticas de contraseña configurables

### Seguridad

**Autenticación y Autorización**:

- OAuth2 con JWT tokens
- Tokens firmados con algoritmo RS256
- Validación automática en cada petición

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

**Última actualización**: Octubre 2026

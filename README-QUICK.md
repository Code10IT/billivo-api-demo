# API Billivo - Guía Rápida

## Introducción

API REST para crear y gestionar facturas electrónicas, series de numeración y gastos de Billivo Gastos.

**Características principales**:

- Autenticación OAuth2 con tokens JWT
- Creación de facturas ordinarias y rectificativas
- Numeración automática o manual
- Series de numeración propias
- Generación de PDF en base64 (es, en, ca, gl, eu) y envío por email
- Consulta y actualización de facturas
- Exportación de gastos y sincronización con el ERP

---

## Autenticación

### 1. Obtener Token

```bash
POST /token
Authorization: Basic {base64(username:password)}
```

**Respuesta**:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### 2. Usar el Token

```
Authorization: Bearer {access_token}
```

El token expira en 1 hora. Para renovarlo, llama de nuevo a `POST /token` con las mismas credenciales. La API devuelve un `refresh_token`, pero no tiene un endpoint para canjearlo.

---

## Modos de Operación

### Modo NO_ERP (Numeración Automática)

- Billivo genera el número de factura automáticamente
- **NO** incluyas el campo `invoiceNumber` en la petición
- Elige la serie con `formatId` (id de `GET /invoice/series`). Sin él, se usa la serie por defecto
- Ideal para nuevas implementaciones

### Modo ERP (Numeración Manual)

- Controlas la numeración desde tu sistema
- **DEBES** incluir el campo `invoiceNumber` en la petición
- La API rechaza un número que ya está en una factura registrada
- Útil para sistemas ERP existentes

---

## Endpoints

### Crear Factura

```bash
POST /invoice?email=true&lang=es
Authorization: Bearer {token}
Content-Type: application/json
```

`email=true` envía el PDF al cliente. `lang` fija el idioma del PDF. Los dos son opcionales.

### Consultar Facturas

```bash
GET /invoice?userEmail={email}&invoiceNumber={numero}&page={pagina}&limit={limite}&status={estado}&pdf=true
Authorization: Bearer {token}
```

`pdf=true` incluye el PDF de cada factura en `base64pdf`. Cada factura incluye su estado en `status`.

### Actualizar Estado

```bash
PATCH /invoice?userEmail={email}&invoiceNumber={numero}
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "paid"
}
```

`paid` marca la factura como pagada y `sent` deshace el pago. Solo para facturas en `sent` o `paid`: en otro estado devuelve `409`.

### Series de Numeración

```bash
GET /invoice/series
POST /invoice/series
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Tienda online",
  "type": "F",
  "invoicePrefix": "WEB",
  "invoiceYear": 4,
  "invoiceDigits": 5,
  "firstSeparator": "-",
  "secondSeparator": "/"
}
```

`type`: `F` para facturas, `C` para rectificativas. Este ejemplo genera `WEB-2026/00001`.

### Billivo Gastos

```bash
GET /expenses/export?from=2026-01-01&to=2026-03-31&format=json&page=1&limit=100
GET /expenses/erp?after={ultimoExpenseId}&limit=100
POST /expenses/erp        { "confirmed": [ { "expenseId": "...", "erpId": "FAC-88" } ] }
GET /expenses/categories
GET /expenses/paymentMethods
PATCH /expenses/status   { "expenseId": "...", "paid": true }  o  { "expenseId": "...", "reviewStatus": "confirmed" }
Authorization: Bearer {token}
```

- La empresa necesita acceso a Billivo Gastos (si no, `403`)
- Sin `format=json`, `/expenses/export` devuelve un enlace a un ZIP con el Excel
- `/expenses/erp` devuelve los gastos que el ERP aún no registró. Tras el `POST`, el gasto ya no se puede editar en Billivo Gastos
- `/expenses/status` marca un gasto como pagado (`paid`) o revisa uno pendiente (`reviewStatus`: `confirmed` o `rejected`, con `rejectionReason`)

---

## Ejemplo Rápido: Crear Factura

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
      "dueDate": "2030-12-31",
      "currency": "EUR",
      "irpf": 0,
      "taxType": "01",
      "regimeKey": "01",
      "customer": {
        "customerName": "ACME Corporation",
        "nationalId": "B12345678",
        "mailAddress": "cliente@acme.com"
      },
      "issuer": {
        "name": "Mi Empresa SL",
        "nationalId": "B99540254",
        "city": "Barcelona",
        "country": "ES",
        "mailAddress": "facturacion@miempresa.com"
      }
    },
    "items": [
      {
        "name": "Servicios Profesionales",
        "type": "Service",
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

**Respuesta** (201 Created):

```json
{
  "invoice": {
    "header": {
      "invoiceNumber": "2025-0000000028",
      ...
    },
    "items": [...],
    "status": "sent"
  },
  "base64pdf": "data:application/pdf;filename=generated.pdf;base64,JVBERi0x...",
  "emailSent": false
}
```

`invoice.status` es `sent` o `validated_with_errors`. Para consultar el estado más adelante sin reenviar la factura, usa `GET /invoice?userEmail={email}&invoiceNumber={numero}` y lee `body[0].status`.

La factura devuelta no incluye `invoiceDate`, `formatId` ni `paymentTermDays` (lista completa en README.md).

---

## Campos Esenciales

### Header (Obligatorios)

- `invoiceType`: `F1`, `F2`, `R1` o `R5` (ver Tipos de Factura)
- `paymentTermName`: Nombre del plazo de pago
- `paymentTermDays`: Días del plazo de pago
- `paymentType`: Método de pago: `Bank Transfer`, `Credit Card`, `PayPal`, `Cash` u `Other`
- `bankName`: Nombre del banco. Obligatorio solo si `paymentType` es `Bank Transfer`
- `dueDate`: Fecha de vencimiento (YYYY-MM-DD). No puede ser anterior a hoy ni a `invoiceDate`
- `taxType`: "01"
- `regimeKey`: "01"

### Header (Opcionales)

- `paymentTypeDescription`: Descripción/número de cuenta (IBAN para transferencias)
- `irpf`: Porcentaje de retención. Solo `0`, `1`, `7` o `15` (por defecto `0`)
- `SWIFT`: Código SWIFT/BIC del banco (8 u 11 caracteres)
- `formatId`: Serie de numeración (solo modo NO_ERP)

### Customer (Obligatorios)

- `customerName`: Nombre del cliente
- `nationalId`: NIF/CIF del cliente

Opcional: `mailAddress` (email del cliente), necesario para `email=true`. El cliente no es obligatorio en `F2` y `R5`.

### Issuer (Obligatorios)

- `name`: Nombre de tu empresa
- `nationalId`: NIF/CIF de tu empresa
- `mailAddress`: Email de un usuario de Billivo de la misma empresa que el token

Opcionales: `city` (ciudad) y `country` (código de país, ej: "ES").

### Items (Obligatorios)

- `name`: Nombre del producto/servicio
- `type`: `Service` o `Article`
- `unitPrice`: Precio unitario
- `quantity`: Cantidad
- `vatPercentage`: % de IVA
- `discountPercentage`: % de descuento (`0` si no hay)
- `discountAmount`: Importe del descuento (`0` si no hay)
- `totalExcTax`: Total de la línea sin impuestos

### Items (Opcionales)

- `equivalenceSurcharge` y `equivalenceSurchargeTax`: tipo y cuota del recargo de equivalencia (los dos o ninguno)

---

## Tipos de Factura

**Facturas Ordinarias**:

- `F1`: Factura normal
- `F2`: Factura simplificada (ticket)

**Facturas Rectificativas**:

- `R1`: Factura rectificativa
- `R5`: Rectificativa de factura simplificada

Para facturas rectificativas, añade:

- `correctedInvoiceNumber`: Número de factura a corregir
- `correctiveType`: "I" (diferencia) o "S" (sustitución)

---

## Estados de Factura

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

---

## Errores Comunes

| Código | Causa                                                   | Solución                                    |
| ------ | ------------------------------------------------------- | ------------------------------------------- |
| 400    | Datos inválidos o faltantes, o `dueDate` anterior a hoy | Verifica campos obligatorios y fechas       |
| 401    | Token inválido o expirado                               | Obtén un nuevo token                        |
| 403    | Sin acceso a Billivo Gastos                             | Solicita el acceso a Billivo                |
| 404    | Factura no encontrada                                   | Verifica número de factura y userEmail      |
| 409    | Serie con el mismo formato                              | Cambia el prefijo, el año o los separadores |
| 500    | Error del servidor                                      | Contacta con soporte                        |

---

## Mejores Prácticas

1. **Tokens**: Guarda el token de forma segura. Antes de que expire, pide otro con `POST /token` (no hay endpoint de `refresh_token`)
2. **Validación**: Verifica todos los campos obligatorios antes de enviar
3. **Paginación**: Usa `page` y `limit` al consultar muchas facturas (máx: 100/página)
4. **Numeración**: En modo NO_ERP, no envíes `invoiceNumber`. Guarda el número que devuelve la API
5. **PDF**: El `base64pdf` está listo para guardarse o enviarse directamente
6. **Gastos**: Pagina `/expenses/erp` con `after` y confirma cada página antes de pedir la siguiente

---

## Soporte

**Email**: equipo@billivo.com

**Documentación completa**: Ver README.md

**API Interactiva**: Abre `index.html` para explorar con Swagger UI

---

**Última actualización**: Octubre 2026

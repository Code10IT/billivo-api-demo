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

El token expira en 1 hora. Usa el `refresh_token` para renovarlo.

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

`pdf=true` incluye el PDF de cada factura en `base64pdf`.

### Actualizar Estado

```bash
PATCH /invoice?userEmail={email}&invoiceNumber={numero}
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "sent"
}
```

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
Authorization: Bearer {token}
```

- La empresa necesita acceso a Billivo Gastos (si no, `403`)
- Sin `format=json`, `/expenses/export` devuelve un enlace a un ZIP con el Excel
- `/expenses/erp` devuelve los gastos que el ERP aún no registró. Tras el `POST`, el gasto ya no se puede editar en Billivo Gastos

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
      "dueDate": "2025-12-28",
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
        "unitPrice": 100,
        "quantity": 5,
        "vatPercentage": 21
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
    "items": [...]
  },
  "base64pdf": "data:application/pdf;filename=generated.pdf;base64,JVBERi0x...",
  "emailSent": false
}
```

---

## Campos Esenciales

### Header (Obligatorios)

- `invoiceType`: "F1" (factura normal) o "R1" (rectificativa)
- `paymentTermName`: Nombre del plazo de pago
- `paymentTermDays`: Días del plazo de pago
- `paymentType`: Método de pago
- `dueDate`: Fecha de vencimiento (YYYY-MM-DD)
- `taxType`: "01"
- `regimeKey`: "01"
- `irpf`: Porcentaje de retención (0-100)

### Header (Opcionales)

- `paymentTypeDescription`: Descripción/número de cuenta (IBAN para transferencias)
- `bankName`: Nombre del banco (para transferencias bancarias)
- `SWIFT`: Código SWIFT/BIC del banco (8 u 11 caracteres)
- `formatId`: Serie de numeración (solo modo NO_ERP)

### Customer (Obligatorios)

- `customerName`: Nombre del cliente
- `nationalId`: NIF/CIF del cliente
- `mailAddress`: Email del cliente

### Issuer (Obligatorios)

- `name`: Nombre de tu empresa
- `nationalId`: NIF/CIF de tu empresa
- `city`: Ciudad
- `country`: Código país (ej: "ES")
- `mailAddress`: Email de tu empresa

### Items (Obligatorios)

- `name`: Nombre del producto/servicio
- `unitPrice`: Precio unitario
- `quantity`: Cantidad
- `vatPercentage`: % de IVA

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
- `sent`: Enviada
- `validated`: Validada
- `paid`: Pagada
- `overdue`: Vencida
- `cancelled`: Cancelada

---

## Errores Comunes

| Código | Causa                       | Solución                                    |
| ------ | --------------------------- | ------------------------------------------- |
| 400    | Datos inválidos o faltantes | Verifica campos obligatorios                |
| 401    | Token inválido o expirado   | Obtén un nuevo token                        |
| 403    | Sin acceso a Billivo Gastos | Solicita el acceso a Billivo                |
| 404    | Factura no encontrada       | Verifica número de factura y userEmail      |
| 409    | Serie con el mismo formato  | Cambia el prefijo, el año o los separadores |
| 500    | Error del servidor          | Contacta con soporte                        |

---

## Mejores Prácticas

1. **Tokens**: Guarda el token de forma segura y renuévalo antes de que expire
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

**Última actualización**: Septiembre 2026

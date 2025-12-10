# API Billivo - Guía Rápida

## Introducción

API REST para crear y gestionar facturas electrónicas de forma programática.

**Características principales**:
- Autenticación OAuth2 con tokens JWT
- Creación de facturas ordinarias y rectificativas
- Numeración automática o manual
- Generación de PDF en base64
- Consulta y actualización de facturas

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
- Ideal para nuevas implementaciones

### Modo ERP (Numeración Manual)
- Controlas la numeración desde tu sistema
- **DEBES** incluir el campo `invoiceNumber` en la petición
- Útil para sistemas ERP existentes

---

## Endpoints

### Crear Factura
```bash
POST /invoice
Authorization: Bearer {token}
Content-Type: application/json
```

### Consultar Facturas
```bash
GET /invoice?userEmail={email}&invoiceNumber={numero}&page={pagina}&limit={limite}&status={estado}
Authorization: Bearer {token}
```

### Actualizar Estado
```bash
PATCH /invoice?userEmail={email}&invoiceNumber={numero}
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "sent"
}
```

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
  "base64pdf": "data:application/pdf;filename=generated.pdf;base64,JVBERi0x..."
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

| Código | Causa                      | Solución                                  |
|--------|----------------------------|-------------------------------------------|
| 400    | Datos inválidos o faltantes| Verifica campos obligatorios              |
| 401    | Token inválido o expirado  | Obtén un nuevo token                      |
| 404    | Factura no encontrada      | Verifica número de factura y userEmail    |
| 500    | Error del servidor         | Contacta con soporte                      |

---

## Mejores Prácticas

1. **Tokens**: Guarda el token de forma segura y renuévalo antes de que expire
2. **Validación**: Verifica todos los campos obligatorios antes de enviar
3. **Paginación**: Usa `page` y `limit` al consultar muchas facturas (máx: 100/página)
4. **Numeración**: En modo NO_ERP, no envíes `invoiceNumber`. Guarda el número que devuelve la API
5. **PDF**: El `base64pdf` está listo para guardarse o enviarse directamente

---

## Soporte

**Email**: equipo@billivo.com

**Documentación completa**: Ver README.md

**API Interactiva**: Abre `index.html` para explorar con Swagger UI

---

**Última actualización**: Noviembre 2025

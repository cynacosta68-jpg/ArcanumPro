# Facturador Masivo

Panel web que se sirve sobre el motor [Arcanum](./README.md) para emitir comprobantes
electrónicos de ARCA (ex-AFIP) **en lote** y administrar el seguimiento de facturas.
Está pensado para dos realidades distintas, que se eligen con un **perfil** al ingresar:

- **Asociación médica / entidad que liquida a profesionales.** Importás un lote de
  comprobantes "globales" (uno por obra social), y dentro de cada uno, una fila por
  profesional. Para cada profesional podés **emitir vos** el comprobante (y obtener el
  CAE) o **solicitarle** que lo emita él y después **cargar** la factura que te manda.
- **PyME que factura a sus clientes.** Emitís comprobantes individuales y obtenés el
  CAE al instante, o **importás un comprobante ya emitido** (leyendo su PDF) para
  registrarlo en el sistema.

> El panel corre sobre el mismo contenedor que el motor. No necesitás nada aparte:
> al desplegar Arcanum, el Facturador queda disponible en la raíz (`/`).

---

## Acceso

El panel se sirve en la raíz del dominio (`https://tu-dominio/`). El login usa los
usuarios de Arcanum (locales o, si lo configurás, OIDC). El primer superadmin se crea
con `ARCANUM_ADMIN_USER` / `ARCANUM_ADMIN_PASS` la primera vez que arranca con la base
vacía (ver [`.env.example`](./.env.example)).

El rol **admin/superadmin** ve todos los CUIT; un usuario con scope por CUIT solo ve
los suyos (aislamiento multi-inquilino del motor).

---

## Secciones del panel

- **Panel** — resumen y accesos rápidos.
- **Clientes** — los CUIT que operás y la carga de su **certificado** de ARCA (la clave
  privada se cifra en reposo; ver [SECURITY.md](./SECURITY.md)).
- **Representados** — terceros a los que les facturás **con tu propio certificado**
  (delegación). Cada representado guarda su razón social, domicilio, condición frente al
  IVA, email y si es MiPyME. Se administra por perfil (los de Asociación no se mezclan
  con los de PyME). Se pueden importar desde un `.xlsx`.
- **Servicios** — catálogo de web services de ARCA y su estado de activación por CUIT.
- **Emitir** — el flujo principal, distinto según el perfil (ver abajo).
- **Comprobantes** — listado de lo emitido/registrado, con descarga de PDF, envío por
  email y exportación a CSV (las columnas del CSV se adaptan al perfil).
- **Seguimiento** — solo Asociación: profesionales a los que les pediste factura y
  todavía no cargaste, con semáforo por antigüedad. Permite reenviar la solicitud y
  marcar como cargada.
- **Generador / Webhooks / Usuarios** — utilidades del motor y administración.

---

## Flujo Asociación

1. **Importar lote** (`.xlsx`). Cada comprobante global trae sus filas (un profesional
   por fila): profesional, matrícula, responsabilidad fiscal, tipo, período, total.
2. Por cada profesional, en el detalle del global, elegís:
   - **Emitir** → el sistema pide el CAE a ARCA y queda el comprobante.
   - **Solicitar factura** → manda un email al profesional pidiéndole que la emita.
     Eso lo deja **pendiente** en *Seguimiento*.
3. En **Seguimiento** ves los pendientes con color por antigüedad (verde &lt; 5 días,
   amarillo 5–15, rojo &gt; 15). Podés **reenviar** la solicitud o, cuando te llega la
   factura, **cargarla** (CAE + número + PDF) y se quita de pendientes.

El emisor de cada comprobante (vos o un representado) se elige en el momento de emitir.
Un filtro por obra social ayuda a navegar lotes grandes.

## Flujo PyME

- **Emitir y obtener CAE** — formulario individual: emisor (propio o representado),
  condición y tipo de comprobante, moneda (con cotización si es USD), receptor e
  importes. Devuelve el CAE al instante.
- **Importar comprobante** — subís el **PDF de un comprobante ya emitido**. El sistema
  lee de ahí el CUIT del emisor, tipo, punto de venta y número, fecha, importe total,
  CAE y vencimiento (primero por el **QR de ARCA**, con respaldo por texto). Antes de
  registrarlo valida dos cosas:
  - **CAE duplicado** → si ese CAE ya está en la base, avisa "comprobante ya ingresado".
  - **CUIT del emisor** → si no corresponde a un emisor habilitado (tu CUIT o el de un
    representado tuyo), lo bloquea.

---

## Datos del emisor en el PDF

Como ARCA no devuelve los datos fiscales del emisor, el PDF los resuelve en este orden:

1. lo que venga en el propio comprobante (`emisor` / `meta.emisor`);
2. la variable de entorno **`ARCANUM_EMISORES`** (JSON por CUIT) — **acá van tus datos
   reales**, fuera del repo;
3. el archivo de ejemplo [`src/services/emisores.json`](./src/services/emisores.json)
   (solo placeholders);
4. un default vacío.

Para tu despliegue: poné tus datos en `ARCANUM_EMISORES` (ver `.env.example`) y dejá el
`emisores.json` del repo con el ejemplo genérico.

---

## Homologación vs. producción

Arcanum separa por entorno (`ARCANUM_ENV=homo|prod`) y nunca mezcla los datos. En
**homologación** los comprobantes **no tienen validez fiscal** y sirven para probar.
Para producción necesitás un **certificado productivo** asociado al web service de
facturación, el **punto de venta** habilitado para web services, y —si facturás por
terceros— las **delegaciones** correspondientes en ARCA. Ver [DESPLIEGUE.md](./DESPLIEGUE.md).

> Esto es software, no asesoramiento contable ni legal. Validá tus comprobantes con tu
> contador y con la normativa vigente de ARCA.

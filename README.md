
# 🤖 DeliveryBot — Menú Digital Bot n8n

> **Sistema Autónomo de Pedidos, Gestión de Sesiones FSM e Integración Telegram + Google Sheets DB**

---

## 📌 Información del Proyecto

* **Proyecto:** DeliveryBot — Menú Digital n8n (Paso 1)
* **👤 Autor / Desarrollador:** Alexi Mandeli
* **⚡ Motor de Automatización:** n8n (Workflow Engine v1.5 / v2.0)
* **📊 Base de Datos & Integraciones:** Google Sheets API v4, Telegram Bot API

---

## 1. 🌐 Visión General y Arquitectura del Sistema

**DeliveryBot** es una solución backend integral automatizada mediante **n8n**, diseñada para operar un sistema de menú digital, toma de pedidos, gestión interactiva de carritos de compra y panel de control para restaurantes o cafeterías. El flujo permite interactuar dinámicamente con usuarios finales a través de un **Bot de Telegram** y procesar datos en tiempo real utilizando **Google Sheets** como base de datos relacional orientada a documentos (`SESSIONS`, `MENU`, `PEDIDOS`).

> [!NOTE]
> 📌 **CARACTERÍSTICA CLAVE: PERSISTENCIA DE ESTADO**
> El sistema implementa un patrón de **Máquina de Estados Finitos (FSM)** descentralizada. Cada mensaje del usuario desencadena una consulta a la tabla `SESSIONS` para recuperar el contexto exacto (`pantalla_actual`, `carrito_temporal`, `datos_checkout`) y así determinar de forma precisa la siguiente ruta de ejecución.

---

## 2. 🗄️ Estructura y Esquema de Base de Datos (Google Sheets)

La persistencia de datos se gestiona en un libro de Google Sheets denominado **`DeliveryBot_DB`**

*(ID: `1APTn0Y6sI9pTRpxm2HRFNA4AyluSd_z9YcoZO-ece_c`)*.

El libro está estructurado en pestañas especializadas para garantizar la integridad de los datos y el desacoplamiento de responsabilidades:

| Hoja *(Sheet)* | Campos Clave / Esquema | Descripción y Propósito |
| --- | --- | --- |
| 🔄 **`SESSIONS`** | `telegram_id`, `pantalla_actual`, `carrito_temporal`, `datos_checkout`, `ultimo_cambio` | Mantiene la sesión activa del cliente, estado en el embudo (FSM) y estado temporal del carrito en formato JSON. |
| 🍔 **`MENU`** | `id_producto`, `nombre`, `categoria`, `precio`, `descripcion`, `stock` | Catálogo maestro de productos con imágenes, precios, categorización y control automatizado de inventario en tiempo real. |
| 📦 **`PEDIDOS`** | `id_pedido`, `id_usuario`, `detalles_pedido`, `total_pago`, `estado`, `fecha`, `hora`, `nombre_cliente`, `telefono`, `direccion`, `metodo_pago` | Historial consolidado de pedidos confirmados. Utilizado para auditoría, notificaciones a cocina y reporting de administración. |

---
<img width="975" height="404" alt="image" src="https://github.com/user-attachments/assets/19763e03-d199-4b0c-a25a-2edd7d3b24d1" />

## 3. 🔀 Diagrama de Flujo y Control de Rutas (FSM)

El flujo principal de n8n opera recibiendo eventos mediante **Webhook de Telegram**. A continuación:

1. Normaliza el *payload* (diferenciando entre mensajes de texto y clics en botones *Callback*).
2. Recupera el estado del usuario en la base de datos.
3. Evalúa las reglas de negocio a través del nodo Switch **`Route Action`**.

--- <img width="990" height="427" alt="image" src="https://github.com/user-attachments/assets/0773e90a-e826-40b1-8f08-ed39ad96a97b" />


## 4. 🛠️ Desglose Detallado de Nodos y Lógica JavaScript

El flujo cuenta con **30+ nodos interconectados**. A continuación se detallan las unidades de procesamiento más críticas del sistema:

### 4.1. ⚙️ Nodo `Normalize Input` (Código JavaScript v2)

Este nodo estandariza las entradas provenientes de Telegram. Detecta si la interacción es un `callback_query` (botón presionado) o un `message` (texto plano), extrayendo campos clave como `chatId`, `action`, `fromId` y `fromName`.

```javascript
// Normalize Input Code
const item = $input.first().json;
let action = '', chatId = '', fromId = '', fromName = '';

if (item.callback_query) {
  action = item.callback_query.data || '';
  chatId = item.callback_query.message.chat.id;
  if (item.callback_query.from) {
    fromId = item.callback_query.from.id || '';
    fromName = (item.callback_query.from.first_name || '') + 
               (item.callback_query.from.last_name ? ' ' + item.callback_query.from.last_name : '');
  }
} else if (item.message) {
  action = item.message.text || '';
  chatId = item.message.chat.id;
  // ... extrae datos del remitente
}

let category = '', route = 'categories';
if (action.startsWith('cat_')) {
  route = 'products';
  category = action.substring(4);
}

return [{ json: { action, chatId, category, route, fromId, fromName } }];

```

### 4.2. 🧠 Nodo `Determine Route` (Lógica FSM)

Es el motor algorítmico del Bot. Cruza la acción recibida con el estado almacenado en `pantalla_actual`. Soporta saludos, selección de categorías, comandos del grupo de administración/cocina, flujo de checkout secuencial y edición de datos.

```javascript
// Fragmento de Determine Route: Enrutamiento FSM y Modos de Checkout
if (action.startsWith('cat_')) { 
  route = 'products'; 
  category = action.substring(4); 
}
else if (action.startsWith('add_')) { 
  route = 'ask_quantity'; 
  id_producto = action.substring(4); 
}
else if (action.startsWith('qty_')) { 
  route = 'add_to_cart'; 
  // ... 
}
else if (action === 'order_confirm') { 
  route = 'checkout_ask'; 
  checkout_next = 'nombre'; 
}
else if (isCheckout) {
  // Manejo de flujo interactivo de recopilación de datos (Nombre, Teléfono, Dirección, Pago)
  if (field === 'telefono' && !/^\d{10}$/.test(trimmed)) {
    route = 'checkout_phone_error';
  } else { 
    // ... 
  }
}

```

### 4.3. ✅ Validaciones de Negocio: Stock y Creación de Pedidos

* **Verificación de Stock:** Antes de confirmar un pedido, el nodo `Validate Stock` verifica en tiempo real que cada artículo solicitado cuente con unidades disponibles en la hoja `MENU`.
* **Generación de ID:** Si el stock es suficiente, `Build Order` genera un código único de pedido con el formato `PED-YYYYMMDD-XXXX` *(ej. `PED-20260812-0001`)*.

---

## 5. 👨‍🍳 Módulo de Cocina, Despacho y Gestión de Estados

Una fortaleza destacada del desarrollo es la integración directa con el **Grupo de Trabajo de la Cocina** *(ID: `-1004484176309` / `-5341804536`)*.

1. **Notificación Instantánea:** Al confirmarse un pedido, se envía un mensaje a la cocina con el desglose completo y el botón interactivo **`▶️ Avanzar estado`**.
2. **Flujo de Transición:** Al presionar el botón, el pedido transiciona secuencialmente:
`Recibido` ➔ `En preparación` ➔ `En camino` ➔ `Entregado`
3. **Sincronización:** Cada avance actualiza la hoja `PEDIDOS` en Google Sheets y notifica automáticamente al cliente en su chat privado de Telegram.

---<img width="930" height="291" alt="image" src="https://github.com/user-attachments/assets/39803960-fcda-4da2-9af8-9d8312a36acd" />


## 6. 🚀 Guía de Instalación y Despliegue en n8n

Para poner en marcha este flujo en una instancia de n8n propia o Cloud, siga estos pasos:

1. 📥 **Importación de JSON:**
Descargue el código fuente del workflow (`.json`) e impórtelo en su panel de n8n mediante la opción **'Import from File'** o **'Import from URL'**.
2. 🔑 **Credenciales de Telegram:**
Cree una credencial de tipo **'Telegram API'** en n8n ingresando el Bot Token otorgado por [@BotFather](https://t.me/BotFather). Asocie esta credencial a los nodos `Telegram Trigger` y `Send Telegram`.
3. 📊 **Credenciales de Google Sheets:**
Configure una conexión OAuth2 o Service Account con permisos de lectura y escritura sobre el libro `DeliveryBot_DB`. Reemplace el Document ID (`1APTn0Y6sI9pTRpxm2HRFNA4AyluSd_z9YcoZO-ece_c`) con el ID de su propia hoja de cálculo.
4. 🌐 **Configuración de Webhook:**
Active el workflow en n8n. El webhook de Telegram registrará automáticamente la URL de producción para empezar a recibir eventos en tiempo real.

> [!TIP]
> ⚡ **COMPATIBILIDAD Y RENDIMIENTO**
> El sistema se encuentra optimizado para **n8n v1.5+** y **n8n v2.0**. Todas las llamadas a Google Sheets utilizan la API v4 de alto rendimiento con operaciones de `appendOrUpdate` automáticas.

# BgMetal Functions - Motherboard Listener

## 📋 Descripción

Firebase Cloud Function que actúa como listener de eventos Pub/Sub para procesar cambios de estado de imágenes y enviar notificaciones push a través de Firebase Cloud Messaging (FCM) a usuarios específicos.

## 🏗️ Arquitectura Funcional

Este proyecto implementa un patrón de mensajería asíncrona donde:

```
Pub/Sub Topic (motherboard-status-updates)
    ↓
Firebase Cloud Function (sub_motherboard_sendNotification)
    ↓
Firestore (deviceTokens + notificationTemplates)
    ↓
Firebase Cloud Messaging (FCM)
    ↓
Dispositivos de Usuario (WEB/ANDROID/IOS)
```

### Flujo de Ejecución

1. **Recepción de Evento Pub/Sub**: La función se activa cuando se publica un mensaje en el topic `motherboard-status-updates`
2. **Extracción de Datos**: Se extrae información del evento incluyendo `userId`, `imageId`, `oldStatus`, `newStatus`, etc.
3. **Consulta de Tokens**: Se buscan todos los tokens FCM activos del usuario en la colección `deviceTokens`
4. **Obtención de Plantillas**: Para cada dispositivo, se obtiene la plantilla de notificación correspondiente según el `deviceType`
5. **Personalización de Mensaje**: Se reemplazan variables en la plantilla con datos del evento
6. **Envío de Notificaciones**: Se envían notificaciones push a todos los dispositivos activos del usuario
7. **Limpieza de Tokens**: Si un token es inválido o expirado, se elimina automáticamente de Firestore

## 🛠️ Stack Tecnológico

- **Runtime**: Node.js 20
- **Lenguaje**: TypeScript 5.2.2
- **Framework**: Firebase Functions v2 (6.4.0)
- **SDK**: Firebase Admin SDK 11.11.1
- **Mensajería**: Google Cloud Pub/Sub 5.2.0
- **Base de Datos**: Cloud Firestore
- **Notificaciones**: Firebase Cloud Messaging (FCM)

## 📁 Estructura del Proyecto

```
BgMetal-Functions-Motherboard-listener/
├── src/
│   ├── index.ts                 # Punto de entrada, define la Cloud Function
│   ├── pubsub-subscriber.ts     # Lógica principal del subscriber
│   ├── fcm-service.ts           # Servicio de envío de notificaciones FCM
│   └── types.ts                 # Definiciones de tipos TypeScript
├── lib/                         # Código compilado (generado por tsc)
├── firebase.json                # Configuración de Firebase
├── tsconfig.json                # Configuración de TypeScript
├── package.json                 # Dependencias y scripts
└── README.md                    # Este archivo
```

### Descripción de Módulos

#### `src/index.ts`
Define la Cloud Function principal que se activa con eventos Pub/Sub:
```typescript
export const sub_motherboard_sendNotification = onMessagePublished(
  "motherboard-status-updates",
  (event) => handlePubSubEvent(event.data)
);
```

#### `src/pubsub-subscriber.ts`
Contiene la lógica de procesamiento de eventos:
- Parsea el mensaje JSON del evento Pub/Sub
- Consulta tokens activos del usuario en Firestore
- Obtiene plantillas de notificación por tipo de dispositivo
- Coordina el envío masivo de notificaciones

#### `src/fcm-service.ts`
Servicio especializado en FCM:
- Envía notificaciones push individuales
- Maneja errores de tokens inválidos
- Elimina tokens expirados de la base de datos

#### `src/types.ts`
Definiciones de tipos TypeScript para el proyecto:
- `DeviceType`: Tipos de dispositivos soportados (WEB, ANDROID, IOS)
- `DeviceToken`: Estructura de tokens en Firestore

## 🔧 Configuración y Requisitos Previos

### Requisitos del Sistema

- **Node.js**: Versión 20.x (obligatorio)
- **npm**: Versión 7 o superior
- **Firebase CLI**: `npm install -g firebase-tools`
- **Cuenta de Firebase**: Proyecto configurado con:
  - Cloud Functions habilitadas
  - Firestore habilitado
  - Pub/Sub habilitado
  - Firebase Cloud Messaging configurado

### Estructura de Firestore Requerida

#### Colección `deviceTokens`
```javascript
{
  userId: string,        // ID del usuario
  token: string,         // Token FCM del dispositivo
  deviceType: string,    // "WEB" | "ANDROID" | "IOS"
  active: boolean,       // Estado del token
  createdAt: string      // Timestamp de creación
}
```

**Índices requeridos**:
- `userId` (ASC) + `active` (ASC)

#### Colección `notificationTemplates`
Documentos con IDs: `WEB`, `ANDROID`, `IOS`

```javascript
{
  title: string,         // Ej: "Estado actualizado: {oldStatus} → {newStatus}"
  body: string          // Ej: "Tu imagen del {shortCreatedDate} ha cambiado"
}
```

**Variables disponibles para interpolación**:
- `{oldStatus}`: Estado anterior de la imagen
- `{newStatus}`: Estado nuevo de la imagen
- `{shortCreatedDate}`: Fecha formateada (DD-MM-YYYY)

### Formato de Mensaje Pub/Sub

El topic `motherboard-status-updates` debe recibir mensajes JSON con esta estructura:

```json
{
  "eventId": "uuid-v4",
  "imageId": "document-id",
  "userId": "user-id",
  "oldStatus": "pending",
  "newStatus": "processed",
  "createdAt": "2025-12-09T10:30:00Z",
  "updatedAt": "2025-12-09T10:35:00Z"
}
```

## 📦 Instalación

### 1. Clonar el repositorio
```bash
git clone <repository-url>
cd BgMetal-Functions-Motherboard-listener
```

### 2. Instalar dependencias
```bash
npm install
```

### 3. Configurar Firebase
```bash
# Autenticarse en Firebase
firebase login

# Seleccionar el proyecto de Firebase
firebase use <project-id>
```

### 4. Configurar credenciales de servicio (opcional para desarrollo local)
Si necesitas ejecutar localmente:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/serviceAccountKey.json"
```

## 🔨 Compilación del Proyecto

### Compilar TypeScript a JavaScript
```bash
npm run build
```

Este comando:
- Ejecuta el compilador TypeScript (`tsc`)
- Compila archivos de `src/` a `lib/`
- Genera archivos `.js` y `.d.ts`
- Valida tipos y sintaxis

**Output esperado**: Carpeta `lib/` con el código JavaScript compilado

### Verificar el código compilado
```bash
ls -la lib/
# Deberías ver: index.js, pubsub-subscriber.js, fcm-service.js, types.d.ts
```

## 🚀 Ejecución y Despliegue

### Desarrollo Local con Emuladores

```bash
npm run serve
# o
npm start
```

Esto inicia:
- Emulador de Cloud Functions (puerto 5001 por defecto)
- Emulador de Pub/Sub (si está configurado)
- Emulador de Firestore (si está configurado)

**Nota**: Para probar con Pub/Sub localmente, necesitas publicar mensajes manualmente al emulador.

### Despliegue a Producción

```bash
# Desplegar solo esta función
npm run deploy

# O usar el comando completo
firebase deploy --only functions:sub_motherboard_sendNotification
```

### Verificar el despliegue
```bash
firebase functions:log
```

## 🧪 Pruebas y Debugging

### Ver logs en tiempo real
```bash
firebase functions:log --only sub_motherboard_sendNotification
```

### Probar la función manualmente
Publica un mensaje en el topic Pub/Sub:

```bash
gcloud pubsub topics publish motherboard-status-updates \
  --message '{
    "eventId": "test-123",
    "imageId": "img-456",
    "userId": "user-789",
    "oldStatus": "pending",
    "newStatus": "approved",
    "createdAt": "2025-12-09T10:00:00Z",
    "updatedAt": "2025-12-09T10:05:00Z"
  }'
```

### Logs esperados
```
Handling status update for userId=user-789, docId=img-456
Evento recibido: {...}
Sending notification: {title: "...", body: "..."}
Push sent to <token>: projects/.../messages/...
Notifications sent successfully.
```

## 📊 Monitoreo y Métricas

### Métricas clave a monitorear:
- **Invocaciones**: Número de veces que se activa la función
- **Duración de ejecución**: Tiempo de procesamiento por evento
- **Errores**: Tasa de fallos y tipos de error
- **Memoria utilizada**: Consumo de recursos
- **Latencia de Pub/Sub**: Tiempo entre publicación y ejecución

### Dashboard de Firebase Console
1. Ir a Firebase Console → Functions
2. Seleccionar `sub_motherboard_sendNotification`
3. Ver métricas de:
   - Invocaciones por minuto
   - Tasa de error
   - Tiempo de ejecución
   - Logs en tiempo real

## 🔍 Linting

```bash
npm run lint
```

Ejecuta ESLint con configuración de Google para validar:
- Estilo de código
- Mejores prácticas
- Posibles errores

## ⚙️ Variables de Entorno y Configuración

### Variables implícitas de Firebase Functions:
- `FIREBASE_CONFIG`: Configuración automática del proyecto
- `GCLOUD_PROJECT`: ID del proyecto de GCP
- `FUNCTION_REGION`: Región de despliegue (default: us-central1)

### Configurar región de despliegue (opcional)
En `src/index.ts`:
```typescript
export const sub_motherboard_sendNotification = onMessagePublished(
  {
    topic: "motherboard-status-updates",
    region: "southamerica-east1" // Ejemplo: São Paulo
  },
  (event) => handlePubSubEvent(event.data)
);
```

## 🐛 Solución de Problemas Comunes

### Error: "Cannot find module 'firebase-admin'"
```bash
rm -rf node_modules package-lock.json
npm install
```

### Error: "PERMISSION_DENIED" en Firestore
- Verificar que Firebase Admin SDK tenga permisos
- Revisar reglas de seguridad de Firestore
- Confirmar que la función corre con credenciales de servicio

### No se envían notificaciones
1. Verificar que existan tokens activos en `deviceTokens`
2. Confirmar que existan plantillas en `notificationTemplates`
3. Validar que los tokens FCM sean válidos y no hayan expirado
4. Revisar logs de la función: `firebase functions:log`

### Error de compilación TypeScript
```bash
# Limpiar y recompilar
rm -rf lib/
npm run build
```

### Función no se activa con eventos Pub/Sub
- Verificar que el topic `motherboard-status-updates` exista
- Confirmar que el proyecto tenga Pub/Sub API habilitado
- Revisar que el formato del mensaje sea JSON válido

## 📝 Scripts Disponibles

| Script | Comando | Descripción |
|--------|---------|-------------|
| `build` | `npm run build` | Compila TypeScript a JavaScript |
| `lint` | `npm run lint` | Ejecuta ESLint en archivos .ts |
| `serve` | `npm run serve` | Inicia emuladores locales |
| `start` | `npm start` | Alias de `serve` |
| `deploy` | `npm run deploy` | Despliega la función a producción |

## 🔐 Seguridad

### Recomendaciones:
1. **Tokens FCM**: Rotar tokens periódicamente y eliminar inactivos
2. **Reglas Firestore**: Asegurar que solo Cloud Functions puedan escribir en `deviceTokens`
3. **IAM**: Usar principio de privilegio mínimo para service accounts
4. **Secrets**: No hardcodear credenciales en el código
5. **HTTPS**: Todas las comunicaciones usan TLS por defecto

## 📈 Escalabilidad

- **Concurrencia**: Firebase Functions v2 soporta hasta 1000 instancias concurrentes por defecto
- **Throttling**: Pub/Sub maneja hasta 10,000 mensajes/segundo por topic
- **Firestore**: Soporta hasta 10,000 lecturas/escrituras por segundo por colección
- **FCM**: Sin límite de mensajes por proyecto (con fair usage policy)

### Optimizaciones recomendadas:
- Implementar batching de notificaciones si se esperan > 100 tokens por usuario
- Usar cache para plantillas de notificación
- Considerar Dead Letter Topic para mensajes fallidos

## 📚 Recursos Adicionales

- [Firebase Functions Documentation](https://firebase.google.com/docs/functions)
- [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)

## 👤 Autor

BgMetal Development Team

## 📄 Licencia

Private - Todos los derechos reservados


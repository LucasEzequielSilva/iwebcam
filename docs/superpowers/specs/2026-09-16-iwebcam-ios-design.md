# iwebcam — App iOS (diseño)

Fecha: 2026-09-16
Estado: aprobado por Lucas, listo para plan de implementación

## Contexto

Clon propio de Camo. Lucas hoy usa Camo con Windows: la app de Camo en el
iPhone muestra un QR, lo escanea/conecta desde la PC, y usa la cámara del
iPhone como webcam en Windows.

Este documento cubre **solo el sub-proyecto de la app iOS**: captura de
cámara, pairing por QR, y servir el streaming por WiFi hacia quien se
conecte. El companion Windows (que recibe el stream y lo expone como cámara
virtual del sistema, vía Media Foundation Virtual Camera) es un subsistema
independiente, con su propia spec, fuera de este documento.

## Decisiones de scope

- **Transporte**: solo WiFi en este MVP (no USB).
- **Video**: H.264 encoded por hardware (VideoToolbox), sobre un socket TCP
  propio con framing simple — no MJPEG, no WebRTC. Elegido por baja latencia,
  buena calidad a bitrate bajo, y porque Windows Media Foundation decodifica
  H.264 nativamente (no repinta el trabajo cuando se arme el companion).
- **Sin dependencias de terceros**: todo con frameworks nativos de Apple
  (AVFoundation, VideoToolbox, Network.framework, CoreImage).
- **Un solo cliente conectado a la vez** (igual que Camo).

## Arquitectura y componentes

- **CameraCaptureService** — wrappea `AVCaptureSession` +
  `AVCaptureVideoDataOutput`. Entrega frames (`CMSampleBuffer`) vía
  delegate/callback. Maneja permisos, selección de cámara (front/back) y
  resolución.
- **H264Encoder** — wrappea `VTCompressionSession`. Recibe `CMSampleBuffer`,
  devuelve NAL units (SPS/PPS al arrancar, después frames). Config: bitrate
  objetivo, keyframe interval, resolución (720p/1080p).
- **PairingService** — genera un token de sesión random al iniciar, resuelve
  la IP local WiFi del dispositivo, arma el payload del QR (`{ip, port,
  token}` como JSON). Expone ese payload a la UI para render.
- **StreamServer** — socket TCP (`NWListener`) escuchando en un puerto fijo.
  Acepta una sola conexión activa. Handshake: el cliente manda el token
  recibido por QR; si coincide, el server arranca a mandar SPS/PPS + stream
  de NALs framed (4 bytes de longitud + payload). Si el cliente se
  desconecta, vuelve a esperar.
- **ConnectionState** — estado observable simple (`disconnected` /
  `waiting` / `connected`) que consume la UI.
- **ContentView (SwiftUI)** — preview de cámara en vivo
  (`AVCaptureVideoPreviewLayer` vía `UIViewRepresentable`) + overlay con QR
  cuando no hay cliente conectado + indicador de estado cuando sí.

Todo vive en un solo target de app iOS.

## Protocolo de wire (StreamServer)

1. Cliente abre conexión TCP al puerto publicado en el QR.
2. Cliente manda el token como primer mensaje: `[4 bytes length][token UTF-8]`.
3. Server valida el token:
   - Si no matchea → cierra la conexión, sigue esperando.
   - Si matchea → arranca `H264Encoder`, manda `[4 bytes length][SPS+PPS]`,
     después por cada frame `[4 bytes length][NAL unit]`.
4. Si el socket se cae (cliente cierra o error de red), el server detiene el
   encoder y vuelve a `waiting`.

## Data flow

1. App arranca → pide permiso de cámara → `CameraCaptureService` arranca la
   captura → `PairingService` genera token + resuelve IP local →
   `StreamServer` empieza a escuchar → UI muestra preview local + QR.
2. El encoder **no corre** mientras no hay cliente conectado (ahorra batería
   — no tiene sentido comprimir frames sin destino).
3. Cliente se conecta y pasa el handshake → `StreamServer` arranca el
   encoder → manda SPS/PPS y después el stream de frames.
4. UI pasa a estado `connected`, oculta el QR.
5. Si el socket se cae, vuelve a `waiting` con el mismo token (no hace falta
   regenerar QR salvo que cambie la IP).

## Error handling

- Permiso de cámara denegado → pantalla explicando que hace falta habilitarlo
  en Settings, sin QR.
- Sin WiFi (o solo celular, sin IP local válida) → mensaje pidiendo conectar
  a la misma red WiFi, en vez de un QR roto.
- Cambio de red mientras la app está abierta (IP cambia) → detectado vía
  `NWPathMonitor`; regenera token + QR automáticamente y corta la conexión
  activa si había una.
- Token inválido en el handshake → el server cierra esa conexión puntual sin
  arrancar el stream; sigue esperando otra.
- Falla del encoder (`VTCompressionSession` devuelve error) → loguear, cortar
  la conexión activa, volver a `waiting`.

## Testing

- Unit tests para lo aislable de hardware: armado del payload del QR,
  validación de token en el handshake, framing de NAL units
  (length-prefix), y la máquina de estados `ConnectionState`.
- `CameraCaptureService`, `H264Encoder` y `StreamServer` se prueban
  manualmente en dispositivo real — el simulador de iOS no tiene cámara.
- Cliente de prueba aparte, fuera del target de la app, en
  `tools/test-client/`: script Python (`socket` + `PyAV`/`ffmpeg`) que hace
  el handshake, recibe los NALs, y los pipea a `ffplay` para validar
  visualmente que el stream decodifica bien. Es una herramienta de
  desarrollo, no parte del producto.

## Fuera de alcance (próximos sub-proyectos)

- Companion Windows: recibe el stream TCP y lo expone como cámara virtual
  del sistema vía Media Foundation Virtual Camera. Spec independiente.
- Transporte USB.
- Controles avanzados de cámara (zoom, exposición manual, filtros).
- Multi-cliente simultáneo.

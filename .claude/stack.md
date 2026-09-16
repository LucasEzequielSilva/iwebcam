# Stack — iwebcam

Clon propio de Camo: app iOS que convierte el iPhone en webcam. Este repo cubre
solo el sub-proyecto de la app iOS (captura + pairing QR + streaming WiFi). El
companion Windows (cámara virtual del sistema) es un sub-proyecto aparte, no
vive acá.

Este stack.md reemplaza por completo el stack default de `D:\Dev\CLAUDE.md`
(que asume Next.js/TS) — este proyecto es 100% Swift/iOS nativo.

## Plataforma

- iOS nativo, Swift + SwiftUI
- Target mínimo: iOS 17
- Sin macOS target, sin companion en este repo

## Frameworks (todos nativos de Apple, sin dependencias de terceros)

- AVFoundation — captura de cámara
- VideoToolbox — encoding H.264 por hardware
- Network.framework (NWListener/NWConnection) — servidor TCP del streaming
- CoreImage (CIFilter) — generación del QR de pairing

## Exclusions

- Nada de WebRTC, MJPEG, ni paquetes externos (SPM/CocoaPods de terceros) en
  el MVP — decisión explícita: menor superficie, todo con frameworks nativos.
- Nada de lógica del companion Windows en este repo.

## Convenciones de archivos

- Swift estándar: `PascalCase.swift` por tipo (no kebab-case — eso es la
  convención default de `D:\Dev\CLAUDE.md` para proyectos TS/Next.js, no
  aplica acá).

## Testing

- Unit tests para lo aislable de hardware: payload de QR, validación de
  token, framing de NAL units, máquina de estados de conexión.
- Captura de cámara / encoder / socket real: prueba manual en dispositivo
  físico (el simulador de iOS no tiene cámara real).

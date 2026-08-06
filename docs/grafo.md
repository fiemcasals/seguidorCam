# Grafo de Dependencias -- seguidorCam

_Generado automaticamente el 2026-08-06T00:07:14.503Z -- no editar a mano, se sobreescribe en cada publicacion._

```mermaid
graph TD
  subgraph US_1785973986502["HU-01: Visualización de la cámara en vivo"]
    REQ_1785974832467["RF-01: Conexión RTSP"]
    REQ_1785974833072["RF-02: Mostrar flujo de video"]
    REQ_1785974833983["RNF-01: Latencia < 100ms"]
  end
  subgraph US_1785973986827["HU-02: Generación automática de fotos para mi perfil"]
    REQ_1785974834220["RF-01: Detección genérica de rostros"]
    REQ_1785974834478["RF-02: Recortar y guardar rostro"]
  end
```
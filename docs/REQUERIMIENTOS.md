# Requerimientos -- seguidorCam

_Generado automaticamente el 2026-08-06T00:07:20.909Z -- no editar a mano, se sobreescribe en cada publicacion._

## HU-01: Visualización de la cámara en vivo

### RF-01: Conexión RTSP (Funcional)

El sistema debe establecer una conexión con el flujo de video de la cámara IP a través del protocolo RTSP.

### RF-02: Mostrar flujo de video (Funcional)

El sistema debe mostrar el flujo de video en una ventana gráfica en tiempo real.

### RNF-01: Latencia < 100ms (No funcional)

La latencia en la recepción y visualización del video no debe superar los 100 milisegundos.

## HU-02: Generación automática de fotos para mi perfil

### RF-01: Detección genérica de rostros (Funcional)

El sistema debe poseer un Modo Captura que utilice un algoritmo de detección genérico para detectar rostros.

### RF-02: Recortar y guardar rostro (Funcional)

El sistema debe recortar el área del rostro detectado y guardarla como una imagen.

### RF-03: Configurar cantidad de captura (Funcional)

El sistema debe permitir al usuario configurar la cantidad de imágenes a capturar y el intervalo.

### RNF-01: Exportación normalizada (No funcional)

Las imágenes recortadas deben exportarse en formato estándar (JPG/PNG) y con dimensiones uniformes.

## HU-03: Entrenamiento del sistema para mi rostro

### RF-01: Notebook Google Colab preparado (Funcional)

Se debe proveer un entorno de ejecución en la nube (Jupyter Notebook) preparado para el entrenamiento.

### RF-02: Aceptar dataset de imágenes (Funcional)

El entorno de entrenamiento debe aceptar el dataset de imágenes generado localmente.

### RF-03: Reentrenamiento YOLO (Funcional)

El script debe ejecutar un proceso de reentrenamiento (Fine-Tuning) sobre un modelo base definiendo una única clase.

### RF-04: Exportar modelo final (Funcional)

El sistema debe permitir la exportación y descarga del modelo entrenado.

### RNF-01: Ejecutable en capa gratuita T4 (No funcional)

El proceso de entrenamiento debe estar optimizado para ejecutarse dentro de los límites de la capa gratuita.

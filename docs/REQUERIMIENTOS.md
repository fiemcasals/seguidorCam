# Requerimientos -- seguidorCam

_Generado automaticamente el 2026-08-06T23:59:14.754Z -- no editar a mano, se sobreescribe en cada publicacion._

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

## HU-04: Identificación exclusiva frente a la cámara

### RF-01: Cargar modelo e inferir (Funcional)

El software local principal debe cargar el modelo reentrenado y realizar inferencias sobre el flujo.

### RF-02: Dibujar recuadro solo en clase objetivo (Funcional)

El sistema local debe dibujar un rectángulo identificatorio únicamente si confirma la presencia de la clase objetivo.

### RNF-01: Mínimo 15 FPS (No funcional)

La inferencia en tiempo real debe procesarse a un mínimo de 15 FPS.

## HU-05: Localización de mi rostro en la pantalla

### RF-01: Extraer coordenadas (x,y,w,h) (Funcional)

El sistema debe extraer las coordenadas del bounding box generado por la inferencia.

### RF-02: Calcular centroide (Cx, Cy) (Funcional)

El sistema debe calcular el centroide geométrico del rostro detectado.

### RF-03: Calcular error vs centro (Funcional)

El sistema debe comparar constantemente la posición del centroide contra el centro absoluto del fotograma.

## HU-06: Movimiento físico y seguimiento de la cámara

### RF-01: Conexión servicio PTZ ONVIF (Funcional)

El sistema debe inicializar una conexión con el servicio PTZ de la cámara utilizando ONVIF.

### RF-02: Enviar comandos direccionales (Funcional)

El sistema debe enviar comandos PTZ direccionales a través de ONVIF si el error supera una zona muerta.

### RNF-01: Uso de librería onvif-zeep (No funcional)

El código de conexión ONVIF debe implementarse utilizando la librería onvif-zeep en Python.

### RNF-02: Control PID para movimientos suaves (No funcional)

El algoritmo de seguimiento debe implementar una lógica de control (Proporcional o PID) para asegurar movimientos suaves.

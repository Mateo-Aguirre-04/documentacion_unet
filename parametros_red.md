# Parametros red

## Parámetros Obligatorios

Técnicamente, el código ya tiene valores por defecto para no romperse de inmediato, estos son los datos que tienes que se debe colocar obligatoriamente para su funcionamiento:

- **`image_dir`:** En este espacio se debe especificar la ruta de la carpeta donde se encuentran guardadas las fotografías originales (las ortofotos capturadas por drones o satélites).
- **`csv_path`:** Corresponde a la ruta exacta donde está alojado el archivo con las mediciones de campo (el documento CSV o Excel que contiene la latitud, longitud y la profundidad real).
- **`lat_column`, `lon_column`, `depth_column`:** Son los identificadores de las columnas del archivo CSV. Es necesario que coincidan con exactitud; por ejemplo, si en el archivo el encabezado dice "Latitud" (con la primera letra en mayúscula), se debe ingresar exactamente igual para que el programa ubique las coordenadas y la profundidad correctamente.

## Parámetros Opcionales

Estos parámetros funcionan como herramientas adicionales. No son indispensables para que el sistema opere, pero pueden enriquecer el análisis si se dispone de dicha información. Si se dejan en blanco o desactivados, el modelo continuará funcionando únicamente con las fotografías estándar.

- **Mapas de apoyo (`water_mask_dir` y `dsm_dir`):** Si el proyecto cuenta con imágenes binarias (blanco y negro) que delimitan la presencia de agua (máscaras) o modelos digitales de superficie (DSM), se debe incluir la ruta en este apartado. En caso de no contar con ellos, el espacio simplemente se deja vacío ingresando `""`.
- **Interruptores de la arquitectura (`use_dsm`, `use_cbam`, `use_transformer`):** El código posee módulos avanzados que se pueden activar (`True`) o desactivar (`False`). Por lo general, se recomienda mantenerlos activos, aunque pueden desactivarse si el equipo de cómputo tiene limitaciones de procesamiento.
- **Aumento de datos (`augment`):** Cuando este parámetro está activo (`True`), el programa toma las fotografías originales y les aplica rotaciones, efecto espejo o alteraciones de brillo. Es una técnica muy útil para que el modelo entrene con mayor variedad visual sin necesidad de recolectar nuevas imágenes.
- **Sistema de reanudación (`resume_training`):** Si el entrenamiento se interrumpe (por ejemplo, por un corte de energía), este parámetro se configura en `True` y se proporciona la ruta del último archivo guardado (`resume_path`). De esta forma, el sistema retoma el proceso desde el último punto de control en lugar de reiniciar desde cero.

## Recomendaciones de Configuración

Para lograr que el modelo aprenda con mayor rapidez, consuma menos recursos y genere resultados precisos, se sugiere establecer los siguientes parámetros de esta manera:

- **`pretrained_encoder = True`:**
    - *¿Qué función cumple?* Permite que la inteligencia artificial no inicie su aprendizaje desde cero, sino que utilice un modelo base con conocimientos previos de otras imágenes.
    - *¿Por qué utilizarlo?* Ahorra una gran cantidad de horas de entrenamiento, ya que la red neuronal cuenta con un reconocimiento básico de formas y colores, permitiéndole enfocarse directamente en las características del agua.
- **`use_amp = True`:**
    - *¿Qué función cumple?* Activa la "precisión mixta", una técnica de cálculo que optimiza el uso de la memoria de la tarjeta gráfica.
    - *¿Por qué utilizarlo?* Evita cuellos de botella en el hardware y previene que el equipo se quede sin memoria o se congele al procesar imágenes de alta resolución.
- **`use_uncertainty = True`:**
    - *¿Qué función cumple?* Permite que el sistema decida automáticamente cómo priorizar sus tareas.
    - *¿Por qué utilizarlo?* Dado que el modelo debe identificar el agua y, simultáneamente, estimar su profundidad, esta función equilibra ambas tareas de forma autónoma, sin necesidad de calibración manual.
- **Dimensiones de recorte (`patch_size = 512` y `stride = 256`):**
    - *¿Qué función cumplen?* Como las ortofotos son demasiado grandes, el programa las divide en cuadrículas de 512x512 píxeles. El parámetro "stride" en 256 asegura que estos recortes se superpongan exactamente a la mitad.
    - *¿Por qué utilizarlos?* Al generar una superposición entre las secciones, se evita la aparición de bordes duros o líneas divisorias cuando el sistema reconstruye la imagen final.
- **`use_cbam = True` y `use_transformer = True`:**
    - *¿Qué función cumplen?* Son mecanismos que otorgan al modelo una mejor capacidad para analizar contextos y focalizar su atención.
    - *¿Por qué utilizarlos?* Debido a que el agua puede presentar texturas visualmente idénticas en diferentes zonas, estas herramientas ayudan a la red a inferir detalles lógicos; por ejemplo, deducir que un píxel ubicado en el centro de un cuerpo de agua suele ser más profundo que uno cercano a la orilla.
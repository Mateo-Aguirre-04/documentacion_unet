# Red RESNET-50 y UNet

---

# Documentación Técnica: Red Multitarea para Estimación de Profundidad en Humedales

## 1. Descripción y Objetivo del Sistema

El sistema implementa una arquitectura híbrida de *Deep Learning* diseñada para generar una representación espacial densa de la profundidad de humedales a partir de fotografías aéreas RGB georreferenciadas. A diferencia de los modelos de regresión tradicionales que devuelven un único valor global, esta red trabaja a nivel de píxel para producir mapas continuos.

El modelo aborda el problema mediante un enfoque multitarea, aprendiendo simultáneamente a:

1. **Segmentar el humedal:** Estima la probabilidad $M(x,y)$ de que un píxel pertenezca al cuerpo de agua.
2. **Estimar la profundidad:** Calcula la profundidad $D(x,y)$ dentro de la zona segmentada.
3. **Cuantificar la incertidumbre (opcional):** Genera un mapa que refleja la confiabilidad espacial de las predicciones.

El sistema destaca por su capacidad para procesar datos geoespaciales, conservando sistemas de referencia de coordenadas (CRS), resoluciones y transformaciones espaciales durante todo el flujo de trabajo.

---

## 2. Procesamiento de Datos y Supervisión Espacial

El flujo de información comienza integrando imágenes aéreas (ortofotos) y mediciones de campo (puntos GPS de profundidad).

- **Ingesta de Imágenes (RGB):** La red recibe imágenes de tres canales, normalizadas al rango $[0,1]$. A través del entrenamiento, el modelo aprende a identificar características visuales no explícitas, como el color del agua, texturas, bordes, sombras y patrones de vegetación circundante.
- **Alineación Espacial GPS-Píxel:** Las mediciones reales de profundidad (latitud, longitud) provistas en formato CSV pasan por un proceso de validación geográfica (filtrando errores y coordenadas fuera de rango). Posteriormente, utilizando `pyproj` y transformaciones afines de `rasterio`, las coordenadas EPSG:4326 se proyectan al CRS del GeoTIFF original, mapeando la relación conceptual $GPS \rightarrow CRS \rightarrow \text{Píxel}$.
- **Supervisión Dispersa:** Dado que no existe una superficie de profundidad continua, las mediciones actúan como puntos dispersos. El sistema rasteriza estos puntos transformándolos en discos de 4 píxeles de radio y normaliza la profundidad respecto a un máximo configurado de 10 metros. Esto genera objetivos de entrenamiento (`depth_target`) y máscaras de validez (`depth_valid_mask`) que indican a la función de pérdida qué píxeles contienen información real.
- **Manejo de Imágenes a Gran Escala:** Para evitar la saturación de memoria de la GPU, las ortofotos se dividen en parches de 512×512 píxeles con un paso (*stride*) de 256 píxeles. Esta superposición previene errores en los bordes y permite promediar las predicciones durante la inferencia.
- **División Estratégica del Dataset:** Los datos se segmentan en entrenamiento (70%), validación (15%) y prueba (15%). Para evitar el sobreajuste geográfico (*data leakage*), la división agrupa las imágenes por el identificador del vuelo (`flight_id`), garantizando que regiones visualmente idénticas no se filtren entre los subconjuntos.

---

## 3. Arquitectura del Modelo

La red sigue un flujo de extracción de características jerárquicas y recuperación de resolución, definido por la secuencia: **RGB → ResNet50 → FPN → Fusión multimodal (Cross-Attention) → CBAM → Transformer → U-Net Decoder → Cabezas de Salida**.

### 3.1. Extracción y Refinamiento de Características

- **Encoder ResNet50:** Actúa como el extractor principal de atributos (desde bordes simples hasta representaciones semánticas complejas). Utiliza *Transfer Learning* (pesos preentrenados) y una estrategia de *progressive unfreezing* en tres fases (descongelando secuencialmente desde las capas nuevas hasta las más profundas, como `layer4` y `layer3`) para estabilizar el entrenamiento.
- **Red Piramidal de Características (FPN):** Recibe los mapas de características a diferentes resoluciones espaciales y aplica un enfoque *top-down* para unificarlos. Esto permite al modelo detectar tanto grandes cuerpos de agua como estructuras estrechas o bordes finos.
- **Preparación Multimodal y Cross-Attention:** El sistema está codificado para admitir un Modelo Digital de Superficie (DSM) mediante un codificador secundario ResNet34. Cuando se activa (`use_dsm=True`), aplica atención cruzada bidireccional donde el RGB actúa como *Query* y el DSM como *Key/Value*. Sin embargo, en la configuración actual activa, procesa únicamente RGB.
- **Módulo CBAM:** Aplica atención secuencial para ponderar qué canales de características son más críticos ($F' = F \times A_c$) y qué regiones espaciales contienen la información más relevante ($F'' = F' \times A_s$), optimizando la identificación visual del humedal.
- **Transformer Bottleneck:** Procesa el tensor colapsado espacialmente aplicando *Multi-Head Self-Attention* (4 cabezas, 2 capas) para capturar relaciones de largo alcance que las redes convolucionales locales suelen ignorar.

### 3.2. Decodificación y Cabezas de Salida (Heads)

El decodificador basado en **U-Net** recupera progresivamente la resolución original utilizando conexiones residuales (*skip connections* con *gating*) que fusionan las características semánticas de la FPN con la información espacial de alta resolución. Finalmente, el tensor pasa a tres cabezas independientes formadas por bloques de convolución, normalización y activación:

- **Cabeza de Segmentación:** Produce probabilidades mediante una activación sigmoide.
- **Cabeza de Profundidad:** Utiliza activación *Softplus* para garantizar predicciones positivas. La profundidad final se calcula modulando la predicción directa por la probabilidad del agua ($D_{final}=D_{raw}\times P_{agua}$), asegurando que la red asocie directamente la presencia de agua con el cálculo de volumen.
- **Cabeza de Incertidumbre:** Genera un mapa espacial que cuantifica la confiabilidad predictiva del modelo usando *Softplus*.

---

## 4. Estrategias de Entrenamiento y Funciones de Pérdida

Para evitar que una tarea domine sobre la otra, el modelo emplea una **Pérdida Multitarea** que pondera dinámicamente los objetivos utilizando la incertidumbre (parámetros aprendibles `log_vars`):

$$
L_{total} = w_{seg}L_{seg} + w_{depth}L_{depth} + w_{unc}L_{unc}
$$

- **Pérdida de Segmentación ($L_{seg}$):** Suma la entropía cruzada binaria (para precisión individual del píxel) y *Dice Loss* (para coherencia estructural global frente a desequilibrios de clase): $L_{seg} = L_{BCE} + L_{Dice}$.
- **Pérdida de Profundidad ($L_{depth}$):** Evalúa el error absoluto ($L_{L1}$), penaliza la pérdida de bordes topográficos comparando gradientes espaciales ($L_{grad}$) y fomenta la suavidad estructural guiada por cambios visuales en el RGB ($L_{smooth}$). La configuración asigna $\lambda_{grad}=0.1$.
- **Pérdida de Incertidumbre ($L_{unc}$):** Aplica una formulación heterocedástica que equilibra el error de profundidad contra la varianza espacial estimada: $L_{unc} = \frac{1}{\sigma^2}\vert{}D_{pred}-D_{real}\vert{} + \log(\sigma^2)$.

### 4.1. Técnicas de Optimización

El entrenamiento es estabilizado mediante **AdamW** (tasa de aprendizaje y decaimiento de pesos en $10^{-4}$), regulado por el planificador **CosineAnnealingWarmRestarts** ($T_0=10, T_{mult}=2$). Adicionalmente, el pipeline aplica *gradient clipping* (1.0), precisión mixta (GradScaler/autocast para aceleración en CUDA) y un robusto *data augmentation* sincronizado (rotaciones, *flips*, ruido gaussiano y alteraciones de brillo/contraste).

---

## 5. Evaluación e Inferencia de Ortofotos

El rendimiento se monitoriza de manera independiente por tarea. La segmentación se valida mediante métricas de área (IoU, Dice), mientras que la profundidad se analiza mediante MAE, RMSE y R².

Durante la etapa de inferencia sobre nuevas ortofotos, el sistema ejecuta el *pipeline* completo:

1. Divide el GeoTIFF en parches.
2. Predice profundidad, segmentación e incertidumbre.
3. Promedia las zonas superpuestas para eliminar artefactos.
4. Desnormaliza los tensores de profundidad.
5. Ensambla y exporta resultados como nuevos archivos (`predicted_depth.tif`, `predicted_water_mask.tif`, `uncertainty.tif`), preservando la integridad del CRS original.

---

## 6. Limitaciones del Sistema y Mejoras Futuras

A pesar de su avanzada estructura geométrica y multiescala, el análisis técnico revela ciertas limitaciones que trazan el camino para su evolución.

### Limitaciones Actuales

- **Desaprovechamiento Multimodal:** Al operar con `use_dsm=False`, se ignora información geométrica crucial.
- **Ruido por Rasterización Dispersa:** Extender mediciones puntuales a discos de 4 píxeles introduce imprecisiones si el radio no coincide con la huella real de la toma. Adicionalmente, limitar la profundidad máxima a 10 metros podría saturar modelos en humedales más profundos.
- **Riesgo de Sobreajuste:** La elevada complejidad arquitectónica de los módulos combinados requiere conjuntos de datos masivos; de lo contrario, el modelo es susceptible a memorizar en lugar de generalizar.
- **Acoplamiento Directo:** La modulación directa de la profundidad mediante la máscara de segmentación restringe el aprendizaje puramente independiente de la topografía submarina.

### Mejoras Futuras

- **Evolución de Entradas:** Activar el DSM (geometría de superficie) y añadir índices multiespectrales (NDVI) o termales para diferenciar mejor suelos, turbidez y vegetación acuática.
- **Expansión de Métricas y Validación Estricta:** Implementar evaluaciones del error absoluto porcentual y errores estratificados por rango de profundidad. Además, separar humedales completamente inéditos para el set de prueba validaría si la red abstrae conceptos universales o simplemente memoriza características locales.
- **Ajuste Arquitectónico:** Ejecutar estudios de ablación (evaluar cada bloque iterativamente: ResNet → +FPN → +CBAM, etc.) para justificar el costo computacional de cada módulo agregado.
- **Diversificación del *Dataset*:** Incorporar variabilidad climática, diferentes alturas de vuelo, multiplicidad de cámaras y variaciones de iluminación es la medida más crítica para robustecer el ecosistema predictivo.
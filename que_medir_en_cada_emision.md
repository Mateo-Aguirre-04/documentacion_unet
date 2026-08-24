# Que medir en cada emision?

# Identificación de la emisión

Cada vuelo debe tener un identificador único para poder diferenciarlo de las demás emisiones.

Se recomienda utilizar una nomenclatura como:

```
EMM-2026-001
EMM-2026-002
EMM-2026-003
```

Cada emisión debe registrar como mínimo:

| Dato | Descripción |
| --- | --- |
| ID de emisión | Identificador único |
| Fecha | Día del vuelo |
| Hora de inicio | Inicio de la adquisición |
| Hora de finalización | Finalización de la adquisición |
| Zona | Área monitoreada |
| Operador | Persona encargada |
| Drone | Equipo utilizado |
| Cámara | Cámara utilizada |

### Ejemplo

```
ID: EMM-2026-001
Fecha: 11/08/2026
Hora: 09:30 – 10:15
Zona: Zona Norte
Drone: DJI XXX
Cámara: RGB XXX
```

## Altura de vuelo

Se debe establecer una altura de vuelo constante para las diferentes emisiones.

La altura debe ser suficiente para cubrir el área de interés y obtener una resolución adecuada.

Se debe registrar:

```
Altura de vuelo: XX metros
```

# Información que debe conservarse de las fotografías

Las fotografías originales deben conservarse sin modificaciones.

Se recomienda almacenar:

```
01_RAW/
└── RGB/
    ├── IMG_0001.JPG
    ├── IMG_0002.JPG
    ├── IMG_0003.JPG
    └── ...
```

También se deben conservar sus metadatos cuando estén disponibles.

Entre ellos:

- Fecha.
- Hora.
- Coordenadas GPS.
- Altura.
- Orientación.
- Modelo de cámara.
- Resolución.
- Distancia focal.

No se recomienda eliminar los archivos originales después de generar la ortofoto.

# Georreferenciación

Para poder comparar diferentes emisiones, los datos deben estar correctamente ubicados geográficamente.

Se puede utilizar:

- GPS del drone.
- RTK.

| Imagen | Latitud | Longitud | Altura |  |
| --- | --- | --- | --- | --- |
| IMG_0001 | ... | ... | ... |  |
| IMG_0002 | ... | ... | ... |  |
| IMG_0003 | ... | ... | ... |  |
|  |  |  |  |  |

---

# Puntos de control terrestre (GCP) (2da opcion)

Si se utilizan GCP, estos deben ubicarse en posiciones conocidas y visibles dentro del área.

Ejemplo:

```
        GCP01 ●────────────● GCP02

              HUMEDAL

        GCP03 ●────────────● GCP04
```

Para cada punto se debe registrar:

| Campo | Información |
| --- | --- |
| ID | Identificador del punto |
| X | Coordenada |
| Y | Coordenada |
| Z | Elevación |
| CRS | Sistema de coordenadas |

Los GCP permiten mejorar la precisión y facilitar la correcta alineación de los productos generados en diferentes emisiones.

# Registro de condiciones ambientales

Durante cada emisión se debe registrar información sobre las condiciones existentes durante la adquisición.

Como mínimo:

- Temperatura ambiente.
- Humedad.
- Viento.
- Nubosidad.
- Precipitación.
- Hora de adquisición.

# Datos que se deben registrar para cada punto de profundidad

Cada medición debe tener un identificador y estar asociada a una ubicación geográfica.

Se recomienda utilizar una tabla como:

| ID | X | Y | Z | Profundidad |
| --- | --- | --- | --- | --- |
| P001 | ... | ... | ... | 0.43 m |
| P002 | ... | ... | ... | 0.71 m |
| P003 | ... | ... | ... | 0.92 m |
| P004 | ... | ... | ... | 0.65 m |

También se recomienda registrar:

- Fecha.
- Hora.
- ID del humedal.
- Método de medición.
- Observaciones.

Por ejemplo:

```
Humedal: H001
Punto: P003
Profundidad: 0.92 m
Fecha: 11/08/2026
Hora: 10:32
```

Esto servira para que se pueda saber la informacion del propio dataset obtenido, recolectando caracteristicas que sirvan para que la red pueda predecir la profundidad en diferentes emisiones

# Obtención del DSM (A futuro)

El DSM representa la elevación de la superficie observada.

ODM genera un archivo como:

```
dsm.tif
```

El DSM debe conservar:

- Coordenadas.
- Sistema de referencia.
- Resolución espacial.
- Valores de elevación.
- Información de NoData.

Debe verificarse que el DSM esté correctamente alineado con la ortofoto RGB.

# Digitalización de la máscara

Se recomienda utilizar un programa GIS, por ejemplo QGIS.

Se carga:

```
orthophoto.tif
```

Después se crea una capa de polígonos.

Se dibuja el límite del humedal:

```
┌───────────────────────────────┐
│                               │
│       ┌───────────────┐       │
│       │               │       │
│       │    HUMEDAL    │       │
│       │               │       │
│       └───────────────┘       │
│                               │
└───────────────────────────────┘
```

Posteriormente se convierte el polígono a raster.

El resultado será:

```
water_mask.tif
```

# Estructura final de los datos

La información de una emisión puede organizarse de la siguiente manera:

```
EMM-2026-001/
│
├── 01_RAW/
│   ├── RGB/
│   │   ├── IMG_0001.JPG
│   │   ├── IMG_0002.JPG
│   │   └── ...
│   │
│   ├── GPS/
│   ├── RTK/
│   └── flight_log/
│
├── 02_FIELD/
│   ├── depth_points.csv
│   ├── GCP.csv
│   └── environmental.csv
│
├── 03_ODM/
│   ├── orthophoto.tif
│   ├── dsm.tif
│   ├── dem.tif
│   └── pointcloud.laz
│
└── 04_DATASET/
    ├── RGB/
    │   ├── tile_001.tif
    │   ├── tile_002.tif
    │   └── ...
    │
    ├── DSM/
    │   ├── tile_001.tif
    │   ├── tile_002.tif
    │   └── ...
    │
    ├── WATER_MASK/
    │   ├── tile_001.tif
    │   ├── tile_002.tif
    │   └── ...
    │
    └── DEPTH_GT/
        ├── tile_001.tif
        ├── tile_002.tif
        └── ...
```
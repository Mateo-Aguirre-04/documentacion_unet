# Posibles parametros futuros

## Archivos principales (Parámetros que usará la red con ODM)

- `orthophoto.tif`

Foto procesada por el ODM para poder obtener los valores de RGB.

- `dsm.tif`

Procesamiento del ODM que contendra la profundidad de los humedales (NO MEDIDOS MANUALMENTE, SE DEBE MODIFICAR EL ODM YA QUE NO GENERA PROFUNDIDAD)

- `water_mask.tif (aun no incluido en la red, podria evitar que el modelo alucine)`

Archivo en el que se puede ver si lo que detecta la red es suelo o humedal, se lo obtiene mediante el programa QGIS, primeramente se debe tener el procesado del ODM.

- `depth_gt.tif`

Primero puede ser un csv que contenga todos los datos (Nombre de la imagen, Coordenadas X Y, profundidad real), obtenidas de las mediciones reales, para que luego estos datos puedan ser llevados al QGIS y ser procesados nuevamente.
# Etiquetado

Como parte de la Iniciativa de Conservación Basada en Datos VLIR-TEAM para la Reserva Natural Antisana, se desarrolló un marco automatizado de procesamiento de imágenes diseñado para caracterizar y monitorear los ecosistemas de humedales altoandinos a partir de fotografías tomadas por vehículos aéreos no tripulados. Este marco de trabajo se validó en el humedal de Pugllohuma, el cual se seleccionó como el sitio de prueba inicial del proyecto previo a su despliegue a gran escala en los sistemas de humedales de la Reserva Ecológica Antisana durante la segunda fase de la investigación.

El flujo de trabajo diseñado integró cuatro etapas fundamentales que abarcaron la anotación automática de los humedales en las imágenes, la segmentación de las áreas de interés, el filtrado para la reducción de ruido y la posterior clasificación y detección. En esta última fase, se probó y evaluó un modelo de red neuronal de reconocimiento de imágenes basado en la arquitectura UNet, el cual fue entrenado para identificar los límites del humedal, las características de la superficie y la estimación del volumen de líquido. Esta arquitectura por etapas permitió verificar de manera independiente cada componente frente a datos reales de campo antes de escalar el sistema a conjuntos de datos más extensos y heterogéneos.

Para llevar a cabo la evaluación del sistema, se estructuró un conjunto de datos compuesto por 150 imágenes, cuyas regiones de humedal fueron etiquetadas mediante un proceso híbrido que combinó el pre-etiquetado automático asistido por el modelo con la posterior corrección manual, logrando optimizar tanto el rendimiento del etiquetado como la precisión en la definición de los bordes. Toda la imaginería fue adquirida a lo largo de múltiples campañas de vuelos de prueba utilizando plataformas aéreas de transición VTOL de despegue y aterrizaje vertical, vinculando de forma directa el procesamiento de datos con las operaciones de vuelo ejecutadas en Pugllohuma y Antisana. El objetivo principal de esta etapa consistió en validar plenamente la precisión y robustez del marco de trabajo sobre el conjunto de datos recopilado, garantizando que el proceso desde la segmentación hasta la detección estuviera listo para su uso operativo antes de iniciar la adquisición masiva de imágenes sobre la reserva natural.

# Prerequisitos

- WSL (Obligatorio)

- Visual Studio Code (Opcional)

# Requisitos

- Docker
- Xlaunch
- Python

## Instalación

### Docker

Con el docker instalado se procede a hacer lo siguiente:

**1. Iniciar un contenedor interactivo con la base de NVIDIA APROX (30-40GB)**

Ejecuta en PowerShell:

PowerShell

```
docker run -it --gpus all --name temporal nvidia/cuda:12.1.1-cudnn8-devel-ubuntu22.04 /bin/bash
```

Docker descargará la imagen oficial y te dejará directamente dentro del contenedor (`root@<id>:/#`).

**2. Instalar las dependencias y Miniconda dentro del contenedor**

Pega estos comandos dentro de la consola de Linux del contenedor:

Bash

```
# Actualizar repositorios e instalar paquetes básicos
apt-get update && apt-get install -y wget git bzip2 libgl1 libglib2.0-0 ca-certificates

# Descargar e instalar Miniconda silenciosamente
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh
bash /tmp/miniconda.sh -b -p /root/miniconda3
rm /tmp/miniconda.sh

# Cargar conda al path del entorno actual y activarlo
export PATH="/root/miniconda3/bin:$PATH"
/root/miniconda3/bin/conda init bash
source ~/.bashrc
```

Si quieres instalar PyTorch, OpenCV o tus librerías aquí mismo, puedes hacerlo ahora con `conda install` o `pip install`. Cuando termines, sal del contenedor escribiendo:

Bash

```
exit
```

**3. Guardar el contenedor como tu nueva imagen (`docker commit`)**

En tu ventana habitual de PowerShell (fuera del contenedor), ejecuta:

PowerShell

```
docker commit temporal yolo_cuda_conda
```

Para verificar que ya existe, ejecuta `docker images` y verás `yolo_cuda_conda` en la lista.

**4. Limpiar el temporal y correr tu entorno definitivo**

Elimina el contenedor temporal:

PowerShell

```
docker rm temporal
```

Ahora ya puedes ejecutar tu comando original de montaje de volumen sin errores:

PowerShell

```
docker run -it --gpus all --name unet `
  -v "C:/Users/"TU-USUARIO"/Documents/Unet_humedales:/app" `
  -w /app `
  yolo_cuda_conda /bin/bash
```

## Librerias

Una vez se haya ejecutado lo ultimo y se este dentro del contenedor instalar las siguientes librerias

```jsx
apt update && apt install -y \
    libegl1 \
    libgl1 \
    libfontconfig1 \
    libdbus-1-3 \
    libglib2.0-0 \
    libx11-6 \
    libx11-xcb1 \
    libxcb1 \
    libxcb-cursor0 \
    libxcb-icccm4 \
    libxcb-image0 \
    libxcb-keysyms1 \
    libxcb-render-util0 \
    libxcb-shape0 \
    libxcb-xinerama0 \
    libxcb-xkb1 \
    libxkbcommon0 \
    libxkbcommon-x11-0\
    rasterio
    
pip install "numpy<2" --force-reinstall
pip install tensorflow
apt-get install -y libgl1-mesa-glx libglib2.0-0
pip install opencv-python-headless
pip install -r requirements.txt
pip install seaborn 
```

## XLaunch

Sin esto no se podra hacer la parte visual del etiquetado ya que al trabajar con wsl no lo permite

### Instala XLaunch en Windows

[VcXsrv / XLaunch](https://sourceforge.net/projects/vcxsrv/?utm_source=chatgpt.com)

Al iniciar XLaunch, selecciona:

1. **Multiple windows**
2. **Start no client**
3. En opciones adicionales marca **Disable access control**
4. Finaliza.

(El XLaunch no aparecera en barra de tareas sin embargo se esta ejecutando en segundo plano, se debe hacer este proceso de abrirlo cada vez que se apague el equipo o se lo reinicie ya que se cerrará)

Finalmente para la ultima configuracion se debe verificar el DISPLAY dentro del contenedor.

En el contenedor correr:
`echo "DISPLAY=$DISPLAY"`

Si `echo $DISPLAY` devuelve vacío, por ejemplo:

```
DISPLAY=
```

Hacer lo siguiente con XLaunch abierto:

```
exportDISPLAY=host.docker.internal:0.0
echo$DISPLAY
python etiquetado_unet.py
```
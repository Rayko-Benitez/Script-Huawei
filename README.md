
---

## Proyecto: Automatización de Reportes Diarios en FusionSolar

**Descripción general**
Este proyecto consiste en un notebook de Python que automatiza la extracción de datos diarios de producción solar desde la plataforma FusionSolar de Huawei. El proceso simula la navegación en el sitio web, selecciona la planta deseada, configura parámetros de informe, recorre un rango de fechas y genera un archivo CSV consolidado con todas las lecturas horarias de cada día.

El objetivo de este README es ofrecer una guía paso a paso, de fácil comprensión para usuarios no técnicos, sobre cómo funciona el código y cómo ejecutar el flujo completo.

---

## Contenido del repositorio

1. **Notebook principal (`Huawei.ipynb`)**
   Contiene todo el código necesario para:

   * Iniciar sesión en FusionSolar
   * Seleccionar la planta
   * Navegar a la sección de “Gestión de informes”
   * Configurar el informe en modo diario
   * Recorrer un intervalo de fechas
   * Recopilar las tablas de datos (24 registros diarios)
   * Guardar el resultado en CSV

2. **Archivo de salida (CSV)**
   Al finalizar la ejecución, se genera un archivo con nombre del estilo

   ```
   {NombrePlanta}_{MesInicio-AñoInicio}_{MesFin-AñoFin}.csv
   ```

   que agrupa, en un solo documento, todas las lecturas horas/día para el rango seleccionado.

---

## Requisitos previos

Para ejecutar el notebook correctamente, se necesita:

* **Python 3.8 o superior**
* **Google Chrome** o **Chromium**, instalado en el sistema
* **Chromedriver** compatible con la versión de Chrome/Chromium
* Conexión a internet para acceder a FusionSolar
* Paquetes de Python (instalables con `pip`):

  * `selenium`
  * `pandas`

> **Nota**: El uso de Selenium implica que el navegador se abrirá (en modo “headless” o visible) para realizar la navegación automatizada.

---

## Configuración inicial

1. **Instalar Python y dependencias**

   * Asegúrese de tener Python instalado (versión 3.8 o superior).
   * Desde línea de comandos (Windows, macOS o Linux), ejecute:

     ```
     pip install selenium pandas
     ```
   * Descargue el archivo de Chromedriver que coincida con su versión de Chrome:
     [https://chromedriver.chromium.org/downloads](https://chromedriver.chromium.org/downloads)
   * Coloque el ejecutable de Chromedriver en alguna ruta incluida en la variable de entorno `PATH` o en la misma carpeta del notebook (para que Selenium lo detecte automáticamente).

2. **Configurar credenciales y datos iniciales**
   En la primera celda del notebook (Variables iniciales), se definen:

   ```python
   url_login = "https://eu5.fusionsolar.huawei.com/unisso/login.action"
   email      = "tu_email@ejemplo.com"     # Reemplazar con su usuario FusionSolar
   password   = "tu_contraseña_segura"      # Reemplazar con su contraseña FusionSolar
   nombre_planta = "NombreDeTuPlanta"       # Nombre exacto tal como aparece en FusionSolar
   fecha_inicio = datetime.strptime("2025-05-01", "%Y-%m-%d")
   fecha_fin    = datetime.strptime("2025-05-03", "%Y-%m-%d")
   ```

   * **`url_login`**: URL de la página de acceso a FusionSolar.
   * **`email` / `password`**: Credenciales de acceso a su cuenta.
   * **`nombre_planta`**: Cadena de texto que coincide exactamente con el nombre de la planta en su dashboard de FusionSolar.
   * **`fecha_inicio` / `fecha_fin`**: Intervalo de fechas (inclusive) que se recorrerá.

3. **Opcional: modo headless**
   Si no desea ver la ventana de Chrome abierta, en la segunda celda de configuración del driver puede descomentar la línea:

   ```python
   options.add_argument("--headless")
   ```

   De este modo, Chrome funcionará en segundo plano.

---

## Flujo general del notebook

A continuación se describen las secciones principales del notebook, explicando qué hace cada bloque. El enfoque es didáctico para usuarios no técnicos.

---

### 1. Importación de librerías y configuración del controlador

* **Objetivo**: Cargar las bibliotecas esenciales (`selenium`, `pandas`, etc.) y abrir una instancia de Chrome controlada por Selenium.
* **Puntos clave**:

  1. Se define el **timout** para las esperas (por ejemplo, 10 segundos).
  2. Se preparan las **opciones de Chrome** (para headless si se desea).
  3. Se inicializa el objeto `driver` que servirá para “conducir” el navegador.

---

### 2. Acceso a FusionSolar y login

* **Paso a paso**:

  1. El script abre la URL de login:

     ```python
     driver.get(url_login)
     ```
  2. Espera a que aparezcan los campos de **usuario (ID=`username`)** y **contraseña (ID=`value`)**.
  3. Introduce los valores de `email` y `password`.
  4. Hace clic en el botón **Iniciar sesión** (ID=`loginBtn`).
  5. Espera la redirección hasta el **dashboard** principal, confirmando acceso correcto.

* **Explicación para no técnicos**:
  El notebook “maneja” Chrome igual que si usted mismo hiciera clic a cada paso: primero abre la página, luego escribe su correo, su contraseña y pulsa en “login”. Gracias a Selenium, estos pasos se automatizan sin intervención manual.

---

### 3. Selección de la planta

* **Objetivo**: En el dashboard, aparecerá un campo de búsqueda (ID=`searchName`).

* **Proceso**:

  1. Espera a que el campo de búsqueda sea visible.
  2. Escribe el texto de la variable `nombre_planta`.
  3. Envía “Enter” para filtrar los resultados.
  4. Hace clic en el enlace que coincide con el nombre de la planta para entrar a su módulo específico.

* **Aclaración**:
  Si su cuenta tiene varias plantas, este bloque se asegura de ir directamente a la que usted haya indicado por nombre.

---

### 4. Navegación a “Gestión de informes”

* **Funcionalidad**: Dentro de la vista de su planta, hay una pestaña o enlace llamado **“Gestión de informes”**.

* **Pasos**:

  1. El script espera a que el enlace **“Gestión de informes”** esté disponible.
  2. Hace clic y, en algunos casos, cambia el **contexto a un iframe** si los elementos del informe están dentro de un contenedor incrustado.

* **Qué significa iframe**:
  Algunos componentes web se “insertan” dentro de un marco interno. Es similar a una “ventanita” anidada: Selenium debe “saber” que tiene que cambiar su foco de atención dentro de ese marco antes de buscar controles de formulario.

---

### 5. Configuración del informe en modo “Diario” y paginación

1. **Cambiar la granularidad a “Diario”**

   * Aparece un desplegable que permite elegir entre opciones como “Mensual”, “Diario”, etc.
   * El script hace clic sobre el selector y, cuando salen las opciones, elige **“Diario”**.
   * Esto significa que el reporte mostrará datos día por día (en lugar de mes o año).

2. **Ajustar la cantidad de filas por página** (solo la primera vez)

   * Por defecto, la tabla muestra “10 / página”. Dado que cada día tiene 24 lecturas (una por hora), esas 10 filas no alcanzan.
   * En la **primera iteración**, se identifica si el selector dice “10 / página”:

     * Si es así, se despliega el menú y se selecciona **“50 / página”**, de modo que la tabla muestre todas las 24 filas.
     * Este paso solo ocurre una vez. En días subsiguientes, cuando el selector ya esté en “50 / página”, el script omite esta parte.

* **Explicación para no técnicos**:
  Para que aparezcan las 24 filas de un día (una fila por cada hora), necesitamos decirle al sistema que muestre 50 registros por página. Hacemos esto solo la primera vez; después, el formulario ya recordará esa preferencia.

---

### 6. Recorrido de fechas y extracción de datos

* **Idea principal**: Por cada día del rango definido (`fecha_inicio` a `fecha_fin`):

  1. Cambiar el campo de **“Periodo estadístico”** al día en cuestión.

     * Este campo permite escribir la fecha directamente (formato `YYYY-MM-DD`).
     * Tras escribir la fecha, se envía la tecla **Enter** para que el formulario comience la búsqueda.
  2. Hacer clic en el botón **“Buscar”**.
  3. Esperar a que la tabla aparezca con todas las filas (≥1).
  4. Recorrer cada fila de la tabla (cada fila = datos de una hora) y leer sus celdas.

     * Se captura la información columna por columna (por ejemplo: “Período estadístico”, “Rendimiento FV”, “Rendimiento inversor”, etc.).
     * Además, se añade un campo adicional `"FechaConsulta"` para saber a qué día pertenece ese grupo de 24 registros.
  5. Agregar todos estos datos a una lista de Python.
  6. Insertar una espera **aleatoria** entre 5 y 20 segundos antes de procesar el siguiente día (para evitar restricciones o bloqueos por parte del sitio).
  7. Avanzar al siguiente día y repetir el proceso.

* **Manejo de “StaleElementReference”**

  * Cada vez que la tabla se recarga, las referencias a los elementos antiguos quedan “obsoletas”.
  * Para evitar errores, el notebook localiza las filas justo antes de leerlas y, si detecta que alguna fila ha quedado irregular, la vuelve a obtener por índice.
  * De esta forma, el script se asegura de capturar siempre las 24 filas sin interrupciones.

---

### 7. Consolidación y guardado en CSV

* Al terminar de recorrer todo el rango de fechas, el resultado es una lista larga con cada fila horaria de cada día.
* Se convierte esa lista en un **DataFrame de pandas** (tabla en memoria con columnas bien definidas).
* Finalmente, se guarda en un archivo CSV con el siguiente formato de nombre:

  ```
  {NombrePlanta}_{MM-YYYY_inicio}_{MM-YYYY_fin}.csv
  ```

  Por ejemplo, si `nombre_planta` es “MiParqueSolar” y el rango es de enero a marzo de 2025, el archivo quedaría:

  ```
  MiParqueSolar_01-2025_03-2025.csv
  ```
* El CSV resultante contiene las columnas originales de la tabla (encabezados extraídos) y, adicionalmente, una columna “FechaConsulta” para distinguir cada bloque de 24 registros.

---

## Cómo ejecutar el notebook

1. **Abrir el notebook**

   * En la carpeta del proyecto, ejecute:

     ```
     jupyter notebook Huawei.ipynb
     ```
   * Se abrirá el navegador con la interfaz de Jupyter.

2. **Configurar las variables iniciales**

   * Edite la primera celda para introducir su `email`, `password`, `nombre_planta` y el rango de fechas.
   * Guarde los cambios de la celda (Shift+Enter para ejecutarla).

3. **Instalar dependencias (si no se han instalado antes)**

   * En una celda diferente:

     ```bash
     !pip install selenium pandas
     ```
   * Asegúrese de que Chromedriver esté accesible desde su sistema.

4. **Ejecutar celdas en orden**

   * Ejecute cada celda una a una (Shift+Enter) en el orden establecido.
   * Observe que el navegador se abrirá y, paso a paso, navegará en FusionSolar.
   * Cuando llegue a la sección de informes, no intervenga manualmente: el script se encargará de seleccionar fechas y leer la tabla.

5. **Esperar a que termine**

   * Al recorrer varios días, se incluirán esperas aleatorias de 5–20 segundos entre cada iteración. Esto es normal.
   * Cuando finalice, verá reflejado el DataFrame final al final del notebook y un archivo CSV creado en la misma carpeta.

---

## Estructura resumida del notebook

1. **Variables iniciales**

   * Definición de URL, credenciales, planta y rango de fechas.

2. **Configuración de Selenium y apertura de Chrome**

   * Tiempo de espera (timeout)
   * Opciones (headless opcional)

3. **Login en FusionSolar**

   * Navegar a la página
   * Llenar usuario/contraseña
   * Validar acceso al dashboard

4. **Selección de planta**

   * Búsqueda por nombre
   * Clic en el enlace de la planta

5. **Acceso a “Gestión de informes”**

   * Clic en la pestaña
   * Cambio a iframe si aplica

6. **Configuración inicial del informe**

   * Cambiar a modo “Diario”
   * Ajustar paginación a “50 / página” (solo la primera vez)

7. **Bucle de extracción por fecha**

   * Para cada día:

     1. Escribir fecha en “Periodo estadístico”
     2. Clic en “Buscar”
     3. Esperar tabla
     4. Leer cada fila y almacenar datos
     5. Espera aleatoria (5–20 segundos)

8. **Consolidación y guardado**

   * Convertir la lista de datos a DataFrame
   * Eliminar filas o columnas vacías
   * Guardar en CSV con nombre basado en planta y rango

---

## Consideraciones finales

* **Entorno controlado**: Para evitar bloqueos, se incluyeron esperas aleatorias y se garantiza que solo haya una instancia de Chrome abierta por vez.
* **Escalabilidad**: Si desea automatizar a largo plazo en un servidor, se sugiere aislar este notebook en un contenedor Docker (con Python, Chrome y Chromedriver instalados) y/o convertirlo en un servicio Python independiente que reciba parámetros de fecha por API.
* **Mantenimiento**:

  * Si Huawei cambia los selectores (ID, clases, texto visible), habrá que actualizar el notebook para apuntar a los nuevos atributos.
  * Antes de ejecutar, verifique que los elementos clave (ID de campos, texto de menús) sigan coincidiendo con la versión actual de FusionSolar.

---

**Autores y contacto**

* Nombre: Rayko Benitez Turuceta
* Proyecto: Sunglass-Energy (automatización de informes solares)
* Contacto: info@sunglass-energy.com

¡Gracias por utilizar este notebook! Si surge cualquier duda o falla, consulte a su equipo de TI o a la documentación de Selenium y FusionSolar para actualizar selectores y tiempos de espera.

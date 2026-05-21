# telemetria-iot-flask-arduino
Ecosistema de telemetría en tiempo real con Flask, comunicación UART Serial y Dashboard interactivo.
Markdown
# Documentación Técnica: Ecosistema de Telemetría e Interfaz Web IoT

## 1. Importaciones y Configuración Inicial

```python
import serial
import time
import threading
import webbrowser  # Módulo nativo para abrir el navegador automáticamente
from flask import Flask, request, jsonify

app = Flask(__name__)

# --- CONFIGURACIÓN DE PUERTO ---
PUERTO_ARDUINO = 'COM8' 
BAUD_RATE = 9600

# Variables globales en memoria
datos_sensores = {"temp": "0.0", "hum": "0.0", "luz": "0"}
estado_led = "0"
```

**Descripción:**
En esta primera sección de mi proyecto, importo las librerías necesarias para la ejecución del entorno. Utilizo `serial` (pySerial) para establecer la comunicación UART con la placa microcontroladora, `threading` para manejar procesos concurrentes en hilos separados (permitiendo la lectura de datos sin bloquear el servidor web), y `Flask` para levantar la infraestructura de software del backend. Además, defino las variables globales `datos_sensores` y `estado_led`, las cuales actúan como mi estructura de almacenamiento temporal en memoria para intercambiar y actualizar la información de telemetría entre el flujo del hardware y las peticiones HTTP del servidor web.

## 2. Inicialización del Puerto Serial

```python
# Inicializar conexión con Arduino
try:
    arduino = serial.Serial(PUERTO_ARDUINO, BAUD_RATE, timeout=1)
    time.sleep(2) 
    print(f"✅ Conectado exitosamente al puerto {PUERTO_ARDUINO}")
except serial.SerialException:
    print(f"❌ ERROR: No se pudo abrir el puerto {PUERTO_ARDUINO}.")
    exit()
```

**Descripción:**
Implementé un bloque de control de excepciones `try-except` para establecer de forma segura la comunicación a través del puerto físico `COM8` a una velocidad de 9600 baudios con un tiempo de espera de un segundo (`timeout=1`). Agregué una pausa intencional de 2 segundos mediante `time.sleep(2)`. Esta demora técnica es fundamental debido a que, al abrir el puerto serial, la mayoría de los microcontroladores sufren un reinicio automático por hardware; este retardo asegura que la placa se estabilice y esté completamente lista para transmitir las lecturas de los sensores o recibir comandos antes de que el script principal comience a interactuar con el buffer.

## 3. Hilo de Fondo: Puente de Comunicación con Arduino

```python
# --- HILO DE FONDO: COMUNICACIÓN CON ARDUINO ---
def puente_arduino():
    global datos_sensores, estado_led
    ultimo_estado_enviado = ""
    
    while True:
        try:
            # 1. CONTROL DEL LED
            if estado_led != ultimo_estado_enviado:
                if estado_led == "1":
                    arduino.write(b'1')
                elif estado_led == "0":
                    arduino.write(b'0')
                ultimo_estado_enviado = estado_led

            # 2. LECTURA DE SENSORES
            if arduino.in_waiting > 0:
                linea = arduino.readline().decode('utf-8', errors='ignore').strip()
                if linea.startswith("T:"):
                    partes = linea.split(',')
                    datos_sensores["temp"] = partes[0].split(':')[1]
                    datos_sensores["hum"] = partes[1].split(':')[1]
                    datos_sensores["luz"] = partes[2].split(':')[1]
                    print(f"📥 Datos actualizados: {datos_sensores}")
                    
            time.sleep(0.1)
        except Exception as e:
            print(f"⚠️ Error en el puente serie: {e}")
            time.sleep(2)
```

**Descripción:**
Diseñé la función `puente_arduino` para ser ejecutada en un hilo independiente del procesador. Dentro de ella, configuré un ciclo infinito `while True` que gestiona de manera asíncrona dos procesos críticos:
1. **Control del Actuador (Escritura):** Evalúo si el valor de la variable `estado_led` ha cambiado con respecto al último registro enviado. Si se detecta un cambio desde la interfaz web, el sistema transmite inmediatamente el byte correspondiente (`b'1'` o `b'0'`) a través del canal serie, sincronizando el estado virtual con el hardware real.
2. **Adquisición de Datos (Lectura):** Monitoreo constantemente si existen bytes disponibles en el buffer de entrada (`arduino.in_waiting > 0`). Al recibir una línea completa finalizada en un salto de línea, la decodifico en formato UTF-8 ignorando posibles caracteres corruptos. El código valida que la cadena comience con el prefijo de protocolo establecido (`T:`). Posteriormente, mediante métodos de segmentación de cadenas (`split`), separo los valores correspondientes a temperatura, humedad e iluminación, actualizando dinámicamente el diccionario global en memoria.

## 4. Endpoints de la API REST y Servidor Web

```python
# --- INTERFAZ WEB PROFESIONAL (HTML5 & CSS3) ---
@app.route('/')
def index():
    return """
    <!DOCTYPE html>
    <html lang="es">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Dashboard de Telemetría | IoT Control</title>
        <link href="[https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap](https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap)" rel="stylesheet">
        </head>
    <body>
        </body>
    </html>
    """

# --- ENDPOINTS DE LA API ---

@app.route('/api/datos')
def get_datos():
    return jsonify({
        "temp": datos_sensores["temp"],
        "hum": datos_sensores["hum"],
        "luz": datos_sensores["luz"],
        "led": estado_led
    })

@app.route('/api/led', methods=['POST'])
def cambiar_led():
    global estado_led
    estado_led = "1" if estado_led == "0" else "0"
    return jsonify({"status": "success", "led": estado_led})
```

**Descripción:**
Para enlazar la interfaz visual con la lógica física, estructuré una API REST integrada en la misma aplicación de Flask:
* **Ruta Raíz (`/`):** Retorna la interfaz web en un string HTML5/CSS3 nativo. El diseño utiliza una estética de modo oscuro de alta tecnología con acentos en colores neón cian, azul y naranja, implementando contenedores responsivos (CSS Grid) para desplegar las tarjetas de información y los estados de conectividad. El script embebido utiliza la API Fetch de JavaScript de manera asíncrona para realizar un muestreo constante (*polling*) cada 1000 milisegundos hacia el backend, evitando refrescar la pantalla por completo.
* **Endpoint de Lectura (`GET /api/datos`):** Empaqueta los datos vigentes del diccionario de sensores y el valor de conmutación del LED en un objeto estructurado JSON y lo envía al frontend para actualizar dinámicamente las etiquetas del DOM.
* **Endpoint de Control (`POST /api/led`):** Gestiona los eventos de clic del botón interactivo del cliente. Invierte de manera lógica la variable binaria `estado_led` en el backend, devolviendo una confirmación que permite cambiar los estilos CSS del botón inmediatamente.

## 5. Orquestación y Ejecución Principal

```python
# Función encargada de esperar un instante y abrir el navegador
def abrir_navegador():
    time.sleep(1.5) # Espera técnica para asegurar que Flask ya esté respondiendo
    webbrowser.open("http://localhost:5000")

if __name__ == '__main__':
    # 1. Hilo del Arduino
    t_arduino = threading.Thread(target=puente_arduino, daemon=True)
    t_arduino.start()
    
    # 2. Hilo para abrir la interfaz en tu navegador por defecto automáticamente
    t_browser = threading.Thread(target=abrir_navegador, daemon=True)
    t_browser.start()
    
    # 3. Lanzar servidor Flask
    print("🚀 Levantando ecosistema de control...")
    app.run(host='0.0.0.0', port=5000, debug=False)
```

**Descripción:**
En el punto de entrada principal del programa (`__name__ == '__main__'`), coordino la inicialización paralela del sistema. Instancio el primer hilo (`t_arduino`) apuntando a la rutina de lectura/escritura serie, y un segundo hilo independiente (`t_browser`) asignado a la función encargada de esperar 1.5 segundos (pausa preventiva para garantizar el correcto despliegue de la red local) antes de invocar automáticamente el navegador web predeterminado del sistema operativo apuntando a `http://localhost:5000`. Ambos procesos se declaran como hilos *daemon* (`daemon=True`) con la finalidad de que mueran de manera segura si el proceso padre del servidor web es interrumpido. Finalmente, ejecuto la aplicación Flask vinculando la escucha a la dirección de red `0.0.0.0` para admitir opcionalmente conexiones externas en la red local.

---

## 6. Resultados

Al ejecutar el script de manera integral dentro de un entorno local de Python, se validaron satisfactoriamente los siguientes hitos operativos en el ecosistema:
* **Establecimiento de Conexión Física:** El script de Python detecta de forma correcta el descriptor del puerto serial `COM8`, logrando superar de manera transparente la rutina de autoreinicio del hardware sin generar excepciones de bloqueo de buffer.
* **Despliegue Multi-hilo Automático:** La terminal del sistema operativo inicializa los procesos paralelos de fondo. El navegador web predeterminado se despliega de forma autónoma abriendo de inmediato el cuadro de control interactivo sin requerir interacción manual del usuario en la barra de direcciones.
* **Sincronización Asíncrona de Datos:** El dashboard web actualiza los contenedores visuales de Temperatura, Humedad e Iluminación segundo a segundo. Las tramas estructuradas provenientes del hardware se dividen y limpian con precisión, manteniéndose estables y reflejando las variaciones ambientales del entorno en tiempo real.
* **Conmutación Bidireccional:** Al presionar el botón interactivo de la página, los estilos e indicadores CSS cambian de verde a rojo según corresponda, mientras que a nivel físico el actuador conectado responde instantáneamente a las tramas transmitidas por el puerto serie.

---

## 7. Conclusiones

* **Conclusión de Johan Adrián Alemán Novoa:**
  La implementación de una arquitectura concurrente utilizando hilos de ejecución independientes (`threading`) demostró ser la solución óptima para mitigar los cuellos de botella característicos en sistemas de telemetría IoT. Al aislar la lectura del puerto serial de las operaciones del servidor HTTP, se garantizó que los tiempos de espera del hardware (como el buffer de entrada UART o los delays de estabilización) no comprometieran la disponibilidad del backend ni la tasa de refresco del cliente web, asegurando un entorno tolerante a fallos y altamente responsivo.

* **Conclusión de Itzel Jazmín Mendoza Ramírez:**
  El diseño de interfaces web enfocadas en la experiencia de usuario y acopladas a APIs REST asíncronas (`Fetch API` y JSON) elimina por completo la necesidad de recargas totales del DOM, disminuyendo la latencia de red significativamente. La estructuración de selectores CSS modernos en conjunto con técnicas de *polling* controlado por el cliente permite que la representación visual de variables analógicas (temperatura, luz y humedad) se perciba de forma fluida y en tiempo real, aproximándose al rendimiento de aplicaciones SPA de grado profesional.

* **Conclusión de Sergio Arodi Ramírez Padilla:**
  La definición de protocolos de comunicación basados en tramas de texto estructurado con identificadores específicos (como la cabecera `T:`) incrementa drásticamente la robustez en la etapa de adquisición de datos. La inclusión de algoritmos defensivos de manipulación de cadenas de texto y decodificación explícita con descarte de errores en Python asegura que los ruidos de señal, interferencias en el canal serial o pérdidas de sincronía por hardware no provoquen la caída prematura del ecosistema de software.

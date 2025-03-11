# PYTHON CON ARDUINO - PAUTAS A SEGUIR

  # Importamos las bibliotecas necesarias
  import pyfirmata
  import time
  from IPython.display import clear_output
  import ipywidgets as widgets
  from IPython.display import display
  
  # Configuración del puerto - ajusta esto a tu puerto Arduino
  # En Windows suele ser 'COM3', 'COM4', etc.
  # En Mac/Linux suele ser '/dev/ttyACM0', '/dev/ttyUSB0', etc.
  PUERTO_ARDUINO = '/dev/ttyACM0'  # Cambia esto según tu sistema
  
  # Función para detectar puertos disponibles
  def listar_puertos():
      """Función para listar puertos seriales disponibles"""
      import serial.tools.list_ports
      puertos = list(serial.tools.list_ports.comports())
      if not puertos:
          print("No se encontraron puertos seriales disponibles")
          return []
      else:
          print("Puertos disponibles:")
          for puerto in puertos:
              print(f"- {puerto.device}")
          return [puerto.device for puerto in puertos]
  
  # Crear clase para manejar el Arduino
  class ControlArduino:
      def __init__(self, puerto=None):
          self.puerto = puerto
          self.board = None
          self.led_pin = None
          self.sensor_pin = None
          self.connected = False
      
      def conectar(self, puerto=None):
          """Conecta con el Arduino en el puerto especificado"""
          if puerto:
              self.puerto = puerto
          
          try:
              # Inicializar la conexión con el Arduino
              self.board = pyfirmata.Arduino(self.puerto)
              print(f"Conectado a Arduino en {self.puerto}")
              
              # Configurar el pin del LED (pin 13 según la imagen)
              self.led_pin = self.board.digital[13]
              self.led_pin.mode = pyfirmata.OUTPUT
              
              # Configurar un pin para leer el sensor/botón (ajustar según tu conexión)
              # Asumiendo que está en el pin 2 por la imagen
              self.sensor_pin = self.board.digital[2]
              self.sensor_pin.mode = pyfirmata.INPUT
              
              # Iniciar el iterador para leer entradas
              self.it = pyfirmata.util.Iterator(self.board)
              self.it.start()
              
              self.connected = True
              return True
          except Exception as e:
              print(f"Error al conectar con Arduino: {e}")
              self.connected = False
              return False
      
      def encender_led(self):
          """Enciende el LED"""
          if self.connected:
              self.led_pin.write(1)
              print("LED encendido")
          else:
              print("Arduino no conectado")
      
      def apagar_led(self):
          """Apaga el LED"""
          if self.connected:
              self.led_pin.write(0)
              print("LED apagado")
          else:
              print("Arduino no conectado")
      
      def parpadear_led(self, veces=5, intervalo=0.5):
          """Hace parpadear el LED el número de veces especificado"""
          if self.connected:
              for i in range(veces):
                  self.led_pin.write(1)
                  print(f"LED encendido ({i+1}/{veces})")
                  time.sleep(intervalo)
                  self.led_pin.write(0)
                  print(f"LED apagado ({i+1}/{veces})")
                  time.sleep(intervalo)
          else:
              print("Arduino no conectado")
      
      def leer_sensor(self):
          """Lee el estado del sensor/botón"""
          if self.connected:
              # Necesitamos un pequeño retraso para asegurar una lectura correcta
              time.sleep(0.1)
              valor = self.sensor_pin.read()
              return valor
          else:
              print("Arduino no conectado")
              return None
      
      def cerrar(self):
          """Cierra la conexión con Arduino"""
          if self.connected:
              self.apagar_led()  # Apagamos el LED antes de desconectar
              self.board.exit()
              print("Conexión con Arduino cerrada")
              self.connected = False
  
  # Función para crear una interfaz de usuario
  def crear_interfaz():
      """Crea widgets para controlar el Arduino"""
      
      # Instanciar la clase de control
      arduino = ControlArduino()
      
      # Lista de puertos disponibles
      puertos = listar_puertos()
      
      # Widget para seleccionar puerto
      puerto_dropdown = widgets.Dropdown(
          options=puertos if puertos else ['No hay puertos disponibles'],
          description='Puerto:',
          disabled=False,
      )
      
      # Botón de conexión
      conectar_btn = widgets.Button(
          description='Conectar',
          button_style='success',
          tooltip='Conectar al Arduino'
      )
      
      # Estado de conexión
      estado_lbl = widgets.Label(value='Desconectado')
      
      # Botones para controlar el LED
      encender_btn = widgets.Button(
          description='Encender LED',
          button_style='info',
          tooltip='Encender el LED',
          disabled=True
      )
      
      apagar_btn = widgets.Button(
          description='Apagar LED',
          button_style='warning',
          tooltip='Apagar el LED',
          disabled=True
      )
      
      parpadear_btn = widgets.Button(
          description='Parpadear LED',
          button_style='info',
          tooltip='Hacer parpadear el LED',
          disabled=True
      )
      
      leer_btn = widgets.Button(
          description='Leer Sensor',
          button_style='info',
          tooltip='Leer el valor del sensor',
          disabled=True
      )
      
      valor_sensor_lbl = widgets.Label(value='Valor del sensor: -')
      
      # Funciones de callback
      def on_conectar_clicked(b):
          if puerto_dropdown.value and puerto_dropdown.value != 'No hay puertos disponibles':
              if arduino.conectar(puerto_dropdown.value):
                  estado_lbl.value = f'Conectado a {puerto_dropdown.value}'
                  encender_btn.disabled = False
                  apagar_btn.disabled = False
                  parpadear_btn.disabled = False
                  leer_btn.disabled = False
                  conectar_btn.description = 'Reconectar'
              else:
                  estado_lbl.value = 'Error al conectar'
          else:
              estado_lbl.value = 'Selecciona un puerto válido'
      
      def on_encender_clicked(b):
          arduino.encender_led()
      
      def on_apagar_clicked(b):
          arduino.apagar_led()
      
      def on_parpadear_clicked(b):
          arduino.parpadear_led()
      
      def on_leer_clicked(b):
          valor = arduino.leer_sensor()
          if valor is not None:
              valor_sensor_lbl.value = f'Valor del sensor: {valor}'
          else:
              valor_sensor_lbl.value = 'Error al leer el sensor'
      
      # Asignar callbacks
      conectar_btn.on_click(on_conectar_clicked)
      encender_btn.on_click(on_encender_clicked)
      apagar_btn.on_click(on_apagar_clicked)
      parpadear_btn.on_click(on_parpadear_clicked)
      leer_btn.on_click(on_leer_clicked)
      
      # Mostrar widgets
      display(widgets.HBox([puerto_dropdown, conectar_btn, estado_lbl]))
      display(widgets.HBox([encender_btn, apagar_btn, parpadear_btn]))
      display(widgets.HBox([leer_btn, valor_sensor_lbl]))
      
      return arduino
  
  # Ejemplo de monitoreo continuo (ejecutar en celda separada para detener con interrupción)
  def monitor_continuo(arduino, intervalo=1.0, max_lecturas=100):
      """Monitorea continuamente el sensor y controla el LED"""
      try:
          for i in range(max_lecturas):
              clear_output(wait=True)
              valor = arduino.leer_sensor()
              print(f"Lectura #{i+1}: Valor del sensor = {valor}")
              
              # Si el sensor está activado (HIGH), encendemos el LED
              if valor == 1:
                  arduino.encender_led()
                  print("Sensor activado -> LED encendido")
              else:
                  arduino.apagar_led()
                  print("Sensor desactivado -> LED apagado")
              
              time.sleep(intervalo)
      except KeyboardInterrupt:
          print("\nMonitoreo detenido por el usuario")
      finally:
          arduino.apagar_led()
  
  # Para usar el sistema, ejecuta la siguiente función que creará la interfaz
  # arduino_control = crear_interfaz()

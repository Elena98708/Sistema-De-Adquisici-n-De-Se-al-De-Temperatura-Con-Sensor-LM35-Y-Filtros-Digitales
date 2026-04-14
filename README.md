# Sistema-De-Adquisici-n-De-Se-al-De-Temperatura-Con-Sensor-LM35-Y-Filtros-Digitales
from machine import Pin, ADC, Timer
import time
import uos

# CONFIGURACIÓN ADC (LM35)
adc = ADC(Pin(34))
adc.atten(ADC.ATTN_11DB)
adc.width(ADC.WIDTH_12BIT)

led = Pin(2, Pin.OUT)

# ==========================
# FRECUENCIA (USUARIO)
# ==========================
try:
    FS = int(input("Ingrese la frecuencia de muestreo (Hz): "))
except:
    FS = 1

if FS <= 0:
    FS = 1
    print("Frecuencia inválida, se usa 1 Hz")

TS_MS = int(1000 / FS)

# ==========================
# BUFFER
# ==========================
BUFFER_SIZE = 10
buffer = [0]*BUFFER_SIZE
index = 0

alpha = 0.2
valor_exp = 0

ultima_lectura = 0
nueva_muestra = False

# ==========================
# SELECCIÓN DE FILTROS
# ==========================
print("\nFiltros disponibles:")
print("1: Promedio")
print("2: Mediana")
print("3: Exponencial")

orden_filtros = []

try:
    cantidad = int(input("¿Cuántos filtros desea usar? (1, 2 o 3): "))
except:
    cantidad = 1

if cantidad < 1:
    cantidad = 1
elif cantidad > 3:
    cantidad = 3

for i in range(cantidad):
    opcion = input("Seleccione filtro #" + str(i+1) + ": ")
    
    if opcion not in ["1", "2", "3"]:
        opcion = "1"  
    
    orden_filtros.append(opcion)

# ==========================
# CONTROL DE IMPRESIÓN
# ==========================
TIEMPO_IMPRESION = 1000
ultimo_print = 0

# FUNCIONES
def promedio():
    return sum(buffer)/BUFFER_SIZE

def mediana():
    orden = sorted(buffer)
    return orden[BUFFER_SIZE//2]

def exponencial(valor):
    global valor_exp
    valor_exp = alpha*valor + (1-alpha)*valor_exp
    return valor_exp

def adc_a_temp(valor):
    volt = valor * (3.3 / 4095)
    temp = volt * 100
    return temp

# TIMER
timer = Timer(0)

def muestrear(t):
    global ultima_lectura, nueva_muestra
    ultima_lectura = adc.read()
    nueva_muestra = True
    led.value(not led.value())

timer.init(period=TS_MS, mode=Timer.PERIODIC, callback=muestrear)

# ==========================
# ARCHIVO
# ==========================
archivo = open("temp_lm35.txt", "w")
archivo.write("Sin_filtrar,Filtrada\n")

print("\nSistema LM35 iniciado...\n")

# ==========================
# LOOP
# ==========================
try:
    while True:

        if nueva_muestra:
            nueva_muestra = False

            buffer[index] = ultima_lectura
            index = (index + 1) % BUFFER_SIZE

            # SIN FILTRO
            temp_raw = adc_a_temp(ultima_lectura)

            # FILTROS EN ORDEN
            valor = ultima_lectura

            for f in orden_filtros:
                if f == "1":
                    valor = promedio()
                elif f == "2":
                    valor = mediana()
                elif f == "3":
                    valor = exponencial(valor)

            temp_filtrada = adc_a_temp(valor)

            # CONTROL DE VELOCIDAD


            print("Sin filtro:", round(temp_raw,2),
                  "| Filtrada:", round(temp_filtrada,2), "°C")

            archivo.write("{:.2f},{:.2f}\n".format(temp_raw, temp_filtrada))
            archivo.flush()  

        

except KeyboardInterrupt:
    print("\nDetenido")

finally:
    timer.deinit()
    archivo.close()
    print("Datos guardados")

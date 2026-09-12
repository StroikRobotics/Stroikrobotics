# CODIGO ABIERTA

#!/usr/bin/env pybricks-micropython
from pybricks.hubs import EV3Brick
from pybricks.ev3devices import Motor, UltrasonicSensor, GyroSensor
from pybricks.parameters import Port, Color
from pybricks.tools import wait
import gc

# =============================================================================
# 1. PANEL DE CONTROL SÚPER OPTIMIZADO
# =============================================================================

VELOCIDAD_BASE = -1000  # Fluidez total

# -- PID de Ultrasonidos --
KP_US =  -1.5
KI_US =   0.001
KD_US =  15.0

# -- Único umbral de paredes --
UMBRAL_PASILLO_MM = 2000  

# -- Zona ciega post-giro --
TIEMPO_SALIDA_ESQUINA_MS = 400

# -- Avance final de vuelta --
TIEMPO_AVANCE_FINAL_MS = 1000

SIGNO_DIRECCION = 1
TOTAL_GIROS     = 12

RELACION_ENGRANAJE = 5
ANGULO_MAX_RUEDAS  = 18
ANGULO_MOTOR_GIRO  = ANGULO_MAX_RUEDAS * RELACION_ENGRANAJE


# =============================================================================
# 2. HARDWARE
# =============================================================================
ev3             = EV3Brick()
motor_traccion  = Motor(Port.D)
motor_direccion = Motor(Port.B)
gyro            = GyroSensor(Port.S1)
sensor_izq      = UltrasonicSensor(Port.S2)
sensor_der      = UltrasonicSensor(Port.S3)


# =============================================================================
# 3. FUNCIONES
# =============================================================================

def calibrar_gyro():
    gyro.reset_angle(0)
    wait(150)
    for _ in range(8):
        if gyro.angle() == 0 and gyro.speed() == 0:
            break
        gyro.reset_angle(0)
        wait(100)

def ejecutar_giro(sentido, giro_en_vuelta):
    ev3.light.on(Color.RED)
    ev3.speaker.beep(frequency=800, duration=60)

    # Volante a tope sobre la marcha
    angulo_volante = ANGULO_MOTOR_GIRO * sentido * SIGNO_DIRECCION
    motor_direccion.run_target(1500, angulo_volante, wait=False)

    # Meta ABSOLUTA: 90, 180, 270 o 360 (sin importar positivo o negativo)
    angulo_meta_abs = 90 * giro_en_vuelta

    while True:
        # Usamos abs() para que ignore el signo de dirección
        angulo_ahora_abs = abs(gyro.angle())
        grados_faltantes = angulo_meta_abs - angulo_ahora_abs
        
        # 1. Llegó al ángulo recto deseado
        if angulo_ahora_abs >= (angulo_meta_abs - 2):
            break
            
        # 2. SALIDA TEMPRANA
        # Si le faltan 35° o menos para terminar el giro, y ya ve el pasillo, aborta el giro
        if grados_faltantes <= 35:
            d_i = sensor_izq.distance()
            d_d = sensor_der.distance()
            if (d_i + d_d) < UMBRAL_PASILLO_MM:
                print("Salida temprana en angulo:", gyro.angle())
                break

        wait(10)

    # Enderezar volante rápidamente
    motor_direccion.run_target(1500, 0, wait=False)
    ev3.light.on(Color.ORANGE)
    
    # === ZONA CIEGA ===
    wait(TIEMPO_SALIDA_ESQUINA_MS)


# =============================================================================
# 4. ARRANQUE
# =============================================================================
ev3.light.on(Color.ORANGE)
ev3.speaker.beep(frequency=500, duration=200)
motor_direccion.reset_angle(0)
motor_direccion.run_target(800, 0, wait=True)
calibrar_gyro()
ev3.light.on(Color.GREEN)
ev3.speaker.beep(frequency=1000, duration=300)
wait(500)


# =============================================================================
# 5. ESTADO INICIAL
# =============================================================================
giros_completados    = 0
integral_us          = 0
error_previo_us      = 0
en_fase_confirmacion = False
sentido_pista        = 0

d0_izq = sensor_izq.distance()
d0_der = sensor_der.distance()
offset_pasillo = d0_izq - d0_der
print("Offset inicial:", offset_pasillo, "mm")


# =============================================================================
# 6. BUCLE PRINCIPAL
# =============================================================================
motor_traccion.run(VELOCIDAD_BASE)

while giros_completados < TOTAL_GIROS:

    dist_izq        = sensor_izq.distance()
    dist_der        = sensor_der.distance()
    suma_distancias = dist_izq + dist_der

    # =========================================================================
    # FASE A: SALIENDO DE LA CURVA (Volviendo al modo recta)
    # =========================================================================
    if en_fase_confirmacion:
        if suma_distancias < UMBRAL_PASILLO_MM:
            # === FILTRO ANTI-GLITCH ===
            wait(30)
            if (sensor_izq.distance() + sensor_der.distance()) >= UMBRAL_PASILLO_MM:
                wait(10)
                continue # Falsa alarma, seguimos en intersección
            # ==========================

            en_fase_confirmacion = False
            ev3.light.on(Color.GREEN)
            
            integral_us          = 0
            error_previo_us      = 0
            offset_pasillo       = sensor_izq.distance() - sensor_der.distance()
            print("Nuevo tramo PID, Offset:", offset_pasillo)
        
        wait(10)
        continue

    # =========================================================================
    # FASE B: DETECCIÓN DE ESQUINA (Iniciando el giro)
    # =========================================================================
    if suma_distancias >= UMBRAL_PASILLO_MM:
        
        # === FILTRO ANTI-GLITCH ===
        wait(30)
        d_i_verif = sensor_izq.distance()
        d_d_verif = sensor_der.distance()
        if (d_i_verif + d_d_verif) < UMBRAL_PASILLO_MM:
            wait(10)
            continue # Falsa alarma, fue un error del sensor en la recta
        # ==========================
        
        if sentido_pista == 0:
            sentido_pista = 1 if d_d_verif > d_i_verif else -1

        giros_completados += 1
        
        giro_en_vuelta = giros_completados % 4
        if giro_en_vuelta == 0:
            giro_en_vuelta = 4

        print("Giro #", giros_completados, " Meta Absoluta:", 90 * giro_en_vuelta, "grados")
        ejecutar_giro(sentido_pista, giro_en_vuelta)
        en_fase_confirmacion = True

        # === Lógica de fin de vuelta (Al llegar a 360°) ===
        if giro_en_vuelta == 4:
            print("Vuelta completada! Avance de 1 segundo...")
            wait(TIEMPO_AVANCE_FINAL_MS)
            
            if giros_completados < TOTAL_GIROS:
                motor_traccion.stop()
                gc.collect()
                calibrar_gyro()
                
                offset_pasillo = sensor_izq.distance() - sensor_der.distance()
                ev3.light.on(Color.GREEN)
                motor_traccion.run(VELOCIDAD_BASE)

        wait(10)
        continue

    # =========================================================================
    # FASE C: MODO RECTA (PID Activo)
    # =========================================================================
    error_pos = (dist_izq - dist_der) - offset_pasillo

    integral_us += error_pos
    integral_us  = max(-300, min(integral_us, 300))
    derivada_us  = error_pos - error_previo_us

    correccion       = (KP_US * error_pos) + (KI_US * integral_us) + (KD_US * derivada_us)
    correccion_motor = correccion * SIGNO_DIRECCION
    
    limite_motor     = ANGULO_MAX_RUEDAS * RELACION_ENGRANAJE
    correccion_motor = max(-limite_motor, min(correccion_motor, limite_motor))

    motor_direccion.track_target(correccion_motor)
    error_previo_us = error_pos
    wait(10)


# =============================================================================
# 7. PARADA FINAL DE CARRERA
# =============================================================================
motor_traccion.hold()
motor_direccion.run_target(1000, 0, wait=True)
ev3.speaker.play_notes(['C4/4', 'E4/4', 'G4/4', 'C5/2'])
print("CARRERA COMPLETADA -", giros_completados, "giros ejecutados.")



# CODIGO CERRADA 
#!/usr/bin/env pybricks-micropython
from pybricks.hubs import EV3Brick
from pybricks.ev3devices import Motor, UltrasonicSensor, GyroSensor
from pybricks.parameters import Port, Direction, Color
from pybricks.tools import wait
from pybricks.iodevices import I2CDevice
from math import cos, radians
import gc

# =============================================================================
#  HARDWARE
# =============================================================================
ev3 = EV3Brick()
motor_direccion = Motor(Port.B)
motor_traccion  = Motor(Port.D, Direction.COUNTERCLOCKWISE)
sensor_izq = UltrasonicSensor(Port.S3)
sensor_der = UltrasonicSensor(Port.S2)
gyro       = GyroSensor(Port.S1)

try:
    pixy = I2CDevice(Port.S4, 0x54)
    pixy_conectada = True
except Exception:
    pixy_conectada = False
    ev3.speaker.beep(frequency=400, duration=500)

# =============================================================================
#  CONFIGURACIÓN
# =============================================================================
VELOCIDAD_CRUCERO  = 600
RELACION_ENGRANAJE = 5
SIGNO_DIRECCION    = -1

OFFSET_IZQ_MM = 0
OFFSET_DER_MM = 0

UMBRAL_PARED_MM           = 1000
LECTURAS_CONFIRMACION     = 10
DISTANCIA_MINIMA_PARED_MM = 220

ANGULO_ESQUIVE                 = 23
DISTANCIA_BLOQUEO_POST_GIRO_CM = 40

# =============================================================================
#  PID — MISMO PARA RECTA Y AVANCE PRE-GIRO
# =============================================================================
KP      = -0.4
KI      =  0.0
KD      =  2.0
KP_GYRO =  0.0
LIMITE_TIMON = 20

# ── Conversión heading → mm equivalente para avanzar_recto_pid ────────────
# El mismo KP/KI/KD actúa sobre grados * ESCALA en lugar de mm.
# Con KP=-0.4: a 5° de error → timon = 5*5*(-0.4) = -10° (moderado).
# Si el avance se tuerce MÁS en vez de corregir: negar ESCALA_GIRO_A_MM.
ESCALA_GIRO_A_MM = 5.0

# =============================================================================
#  GIROS
# =============================================================================
# GIRO_DERECHA: valor de dir_esquina que hace girar FÍSICAMENTE a la derecha.
# Si el robot gira al lado contrario del que tiene más espacio → cambiar a -1.
GIRO_DERECHA = 1

ANGULO_TIMON_MAX_GIRO         = 30
DIAMETRO_RUEDA_MM             = 56
DISTANCIA_CRITICA_MANIOBRA_MM = 400
DISTANCIA_AVANCE_NORMAL_CM    = 40
DISTANCIA_AVANCE_REVERSA_CM   = 90
ANGULO_OBJETIVO_GIRO          = 90
TIMEOUT_ESQUIVE_CARRIL_MS     = 3000
TIMEOUT_GIRO_PRINCIPAL_MS     = 4000

# =============================================================================
#  CARRILES
# =============================================================================
CARRIL_CENTRO  =  0
CARRIL_IZQ     = -1
CARRIL_DER     =  1
carril_actual          = CARRIL_CENTRO
cambios_carril_recta   = 0
MAX_CAMBIOS_CARRIL     = 2
mantener_carril_actual = False

# =============================================================================
#  ESTADO
# =============================================================================
integral_pos       = 0.0
error_pos_prev     = 0.0
pasillo_confirmado = False
contador_dos_paredes  = 0
lecturas_esquina      = 0
ancho_pasillo_memoria = 500
contador_ciclos_pixy  = 0

# =============================================================================
#  FUNCIONES
# =============================================================================
def leer_sensores():
    return (sensor_izq.distance() + OFFSET_IZQ_MM,
            sensor_der.distance() + OFFSET_DER_MM)


def proyectar_perpendicular(dist, angulo_deg):
    return dist * max(cos(radians(angulo_deg)), 0.70)


def calcular_error_posicion(raw_izq, raw_der, dist_izq, dist_der):
    """
    + → robot cerca pared IZQ → girar derecha
    - → robot cerca pared DER → girar izquierda
    """
    global ancho_pasillo_memoria
    tiene_izq = raw_izq < UMBRAL_PARED_MM
    tiene_der = raw_der < UMBRAL_PARED_MM
    if tiene_izq and tiene_der:
        ancho = max(400, min(dist_izq + dist_der, 750))
        ancho_pasillo_memoria = ancho
        return (ancho_pasillo_memoria * 0.50) - dist_izq
    mitad = ancho_pasillo_memoria * 0.50
    if tiene_izq:
        return mitad - dist_izq
    if tiene_der:
        return dist_der - mitad
    return 0.0


def paso_pid(error, integral, prev):
    """Un ciclo PID. Retorna (timon, integral_nuevo, error_actual)."""
    integral += error
    integral  = max(-500.0, min(integral, 500.0))
    deriv     = error - prev
    raw       = error * KP + integral * KI + deriv * KD
    timon     = max(-LIMITE_TIMON, min(raw, LIMITE_TIMON))
    return timon, integral, error


def avanzar_recto_pid(angulo_bloqueo, distancia_cm, velocidad):
    """
    Avanza usando EXACTAMENTE el mismo PID (KP/KI/KD) que la recta normal.
    Error de heading se convierte a mm-equivalente con ESCALA_GIRO_A_MM.
    Con dos paredes: mezcla 70 % ultrasónico + 30 % heading.
    Sin paredes: 100 % heading.
    Si el robot se tuerce más en vez de corregir → negar ESCALA_GIRO_A_MM.
    """
    motor_traccion.reset_angle(0)
    grados_obj  = (abs(distancia_cm) * 10 / (3.14159 * DIAMETRO_RUEDA_MM)) * 360
    int_av      = 0.0
    err_av_prev = 0.0

    while abs(motor_traccion.angle()) < grados_obj:
        raw_i, raw_d = leer_sensores()
        ang    = gyro.angle()
        dist_i = proyectar_perpendicular(raw_i, ang)
        dist_d = proyectar_perpendicular(raw_d, ang)

        # Error de posición por ultrasónicos
        err_us = calcular_error_posicion(raw_i, raw_d, dist_i, dist_d)

        # Error de heading convertido a mm equivalente
        # (gyro.angle() - bloqueo) > 0 cuando el robot se desvía en un sentido
        err_h = (gyro.angle() - angulo_bloqueo) * ESCALA_GIRO_A_MM

        hay_paredes = (raw_i < UMBRAL_PARED_MM) or (raw_d < UMBRAL_PARED_MM)
        if hay_paredes:
            error_av = err_us * 0.7 + err_h * 0.3   # paredes dan más info
        else:
            error_av = err_h                          # solo gyro

        timon, int_av, err_av_prev = paso_pid(error_av, int_av, err_av_prev)
        motor_direccion.track_target(timon * RELACION_ENGRANAJE * SIGNO_DIRECCION)
        motor_traccion.run(abs(velocidad))
        wait(10)

    motor_traccion.stop()


def detectar_direccion_giro(raw_izq, raw_der):
    """
    Determina la dirección del giro mirando qué sensor detecta más espacio.
    No importa la orientación actual: si la derecha está abierta → gira derecha.
    Si cambia a -1 → intercambiar GIRO_DERECHA en la configuración.
    """
    der_abierta = raw_der >= UMBRAL_PARED_MM
    izq_abierta = raw_izq >= UMBRAL_PARED_MM

    if der_abierta and not izq_abierta:
        return GIRO_DERECHA           # solo derecha libre → girar derecha
    if izq_abierta and not der_abierta:
        return -GIRO_DERECHA          # solo izquierda libre → girar izquierda
    # Ambas abiertas → girar hacia donde hay MÁS espacio
    return GIRO_DERECHA if raw_der >= raw_izq else -GIRO_DERECHA


def obtener_firma_pixy():
    if not pixy_conectada:
        return 0
    try:
        pixy.write(0, bytes([0xae, 0xaf, 32, 2, 3, 2]))
        wait(1)
        resp = pixy.read(0, 20)
        for i in range(len(resp) - 7):
            if (resp[i] == 85 and resp[i+1] == 170 and
                    resp[i+2] == 85 and resp[i+3] == 170):
                return resp[i + 6]
    except Exception:
        return 0
    return 0


def calibrar_gyro():
    gyro.reset_angle(0)
    wait(150)
    for _ in range(8):
        if gyro.angle() == 0 and gyro.speed() == 0:
            break
        gyro.reset_angle(0)
        wait(100)


def reset_pid():
    global integral_pos, error_pos_prev
    integral_pos   = 0.0
    error_pos_prev = 0.0


# =============================================================================
#  ENCENDIDO
# =============================================================================
motor_direccion.run_target(1200, 0)
motor_direccion.reset_angle(0)
calibrar_gyro()
ev3.light.on(Color.ORANGE)
ev3.speaker.beep(frequency=1500, duration=150)
motor_traccion.reset_angle(0)
motor_traccion.run(VELOCIDAD_CRUCERO)

grados_bloqueo_giro = (DISTANCIA_BLOQUEO_POST_GIRO_CM * 10 /
                       (3.14159 * DIAMETRO_RUEDA_MM)) * 360

# =============================================================================
#  LOOP PRINCIPAL
# =============================================================================
while True:
    raw_izq, raw_der = leer_sensores()
    angulo_actual    = gyro.angle()
    dist_post_giro   = abs(motor_traccion.angle())

    dist_izq = proyectar_perpendicular(raw_izq, angulo_actual)
    dist_der = proyectar_perpendicular(raw_der, angulo_actual)

    tiene_izq      = raw_izq < UMBRAL_PARED_MM
    tiene_der      = raw_der < UMBRAL_PARED_MM
    post_giro_libre = dist_post_giro > grados_bloqueo_giro

    # =========================================================================
    # 1. CONFIRMACIÓN DE PASILLO
    # =========================================================================
    if tiene_izq and tiene_der and post_giro_libre:
        contador_dos_paredes += 1
        if contador_dos_paredes >= LECTURAS_CONFIRMACION:
            pasillo_confirmado = True
    elif not (tiene_izq and tiene_der):
        if not pasillo_confirmado or not post_giro_libre:
            contador_dos_paredes = 0

    error_pos_din = calcular_error_posicion(raw_izq, raw_der, dist_izq, dist_der)

    # =========================================================================
    # 2. GIRO EN ESQUINA — detección en 1 lectura, dirección automática
    # =========================================================================
    abierto = not tiene_izq or not tiene_der
    if pasillo_confirmado and abierto and post_giro_libre:
        lecturas_esquina += 1
        if lecturas_esquina >= 1:           # ← 1 lectura = reacción inmediata
            motor_traccion.stop()
            wait(300)
            gc.collect()
            ev3.speaker.beep(frequency=800, duration=100)

            lecturas_esquina     = 0
            pasillo_confirmado   = False
            contador_dos_paredes = 0

            # ── Dirección automática por sensor ──────────────────────────────
            dir_esquina = detectar_direccion_giro(raw_izq, raw_der)

            es_normal = min(raw_izq, raw_der) < DISTANCIA_CRITICA_MANIOBRA_MM
            dist_av   = DISTANCIA_AVANCE_NORMAL_CM if es_normal else DISTANCIA_AVANCE_REVERSA_CM
            vel_pos   = VELOCIDAD_CRUCERO if es_normal else -VELOCIDAD_CRUCERO

            # ── Avance con PID de heading (mismo KP/KI/KD) ───────────────────
            angulo_bloqueo_av = gyro.angle()   # bloquear ángulo actual (~0 tras calibración)
            avanzar_recto_pid(angulo_bloqueo_av, dist_av, vel_pos)

            # ── Timón de giro ────────────────────────────────────────────────
            inv        = 1 if es_normal else -1
            timon_giro = (ANGULO_TIMON_MAX_GIRO * RELACION_ENGRANAJE *
                          SIGNO_DIRECCION * dir_esquina * inv)
            motor_direccion.run_target(1200, timon_giro, wait=True)

            # ── Giro hasta ángulo objetivo ────────────────────────────────────
            ang_ini  = gyro.angle()
            ang_meta = ang_ini + ANGULO_OBJETIVO_GIRO * dir_esquina

            t = 0
            while t < TIMEOUT_GIRO_PRINCIPAL_MS:
                ahora = gyro.angle()
                if dir_esquina > 0 and ahora >= ang_meta:
                    break
                if dir_esquina < 0 and ahora <= ang_meta:
                    break
                dif   = abs(ang_meta - ahora)
                vel_g = (VELOCIDAD_CRUCERO if dif > 25
                         else max(int(VELOCIDAD_CRUCERO * 0.3),
                                  int(VELOCIDAD_CRUCERO * dif / 25)))
                motor_traccion.run(vel_g if es_normal else -vel_g)
                wait(10)
                t += 10

            motor_traccion.stop()
            wait(250)
            motor_direccion.run_target(1400, 0, wait=True)
            calibrar_gyro()

            reset_pid()
            motor_traccion.reset_angle(0)
            carril_actual          = CARRIL_CENTRO
            cambios_carril_recta   = 0
            mantener_carril_actual = False

            motor_traccion.run(VELOCIDAD_CRUCERO)
            continue
    else:
        lecturas_esquina = 0

    # =========================================================================
    # 3. EVASIÓN POR PIXY
    # =========================================================================
    contador_ciclos_pixy += 1
    if contador_ciclos_pixy >= 2:
        contador_ciclos_pixy = 0
        firma = obtener_firma_pixy()

        if firma in (1, 2):
            carril_obj = CARRIL_IZQ if firma == 1 else CARRIL_DER

            if carril_actual == carril_obj:
                ev3.light.on(Color.BLUE)
            elif cambios_carril_recta < MAX_CAMBIOS_CARRIL:
                motor_traccion.stop()
                wait(500)
                gc.collect()
                ev3.light.on(Color.RED if firma == 1 else Color.GREEN)

                timon_esq = 45 * RELACION_ENGRANAJE * SIGNO_DIRECCION * carril_obj
                motor_direccion.run_target(1400, timon_esq, wait=True)
                ang_ini_esq = gyro.angle()
                motor_traccion.run(VELOCIDAD_CRUCERO)

                t = 0
                while t < TIMEOUT_ESQUIVE_CARRIL_MS:
                    if abs(gyro.angle() - ang_ini_esq) >= ANGULO_ESQUIVE:
                        break
                    wait(10)
                    t += 10
                motor_traccion.stop()

                motor_direccion.run_target(1400, 0, wait=True)
                motor_traccion.reset_angle(0)
                motor_traccion.run(VELOCIDAD_CRUCERO)

                t = 0
                while t < TIMEOUT_ESQUIVE_CARRIL_MS:
                    factor_t = max(0.5, cos(radians(gyro.angle())))
                    lim_raw  = DISTANCIA_MINIMA_PARED_MM / factor_t
                    lim_desc = 250 / factor_t
                    lect = (sensor_izq.distance() + OFFSET_IZQ_MM
                            if carril_obj == CARRIL_IZQ
                            else sensor_der.distance() + OFFSET_DER_MM)
                    if lect <= lim_desc:
                        motor_traccion.run(int(VELOCIDAD_CRUCERO * 0.5))
                    if lect <= lim_raw or abs(motor_traccion.angle()) > 1500:
                        break
                    wait(10)
                    t += 10
                motor_traccion.stop()

                timon_ret = -45 * RELACION_ENGRANAJE * SIGNO_DIRECCION * carril_obj
                motor_direccion.run_target(1400, timon_ret, wait=True)
                motor_traccion.run(VELOCIDAD_CRUCERO)

                signo_ref = 1 if gyro.angle() > 0 else -1
                t = 0
                while t < TIMEOUT_ESQUIVE_CARRIL_MS:
                    ang = gyro.angle()
                    if signo_ref == 1 and ang <= 0:
                        break
                    if signo_ref == -1 and ang >= 0:
                        break
                    wait(10)
                    t += 10

                motor_traccion.stop()
                motor_direccion.run_target(1400, 0, wait=True)

                carril_actual          = carril_obj
                cambios_carril_recta  += 1
                mantener_carril_actual = True
                ev3.light.on(Color.ORANGE)
                reset_pid()
                motor_traccion.run(VELOCIDAD_CRUCERO)
                continue

    # =========================================================================
    # 4. PID DIRECTO — recta normal (sin cambios)
    # =========================================================================
    if mantener_carril_actual:
        error_pos = 0.0
    else:
        error_pos = error_pos_din

    if dist_izq < DISTANCIA_MINIMA_PARED_MM:
        fuerza = (DISTANCIA_MINIMA_PARED_MM - dist_izq) * 1.5
        if fuerza > abs(error_pos):
            error_pos = fuerza
    elif dist_der < DISTANCIA_MINIMA_PARED_MM:
        fuerza = -(DISTANCIA_MINIMA_PARED_MM - dist_der) * 1.5
        if abs(fuerza) > abs(error_pos):
            error_pos = fuerza

    timon, integral_pos, error_pos_prev = paso_pid(
        error_pos + angulo_actual * KP_GYRO, integral_pos, error_pos_prev)

    motor_direccion.track_target(timon * RELACION_ENGRANAJE * SIGNO_DIRECCION)
    motor_traccion.run(VELOCIDAD_CRUCERO)

 ## Código de la sesión cerrada (cada día tratando de perfeccionar este código)

 # Calibración para el Giroscopio y Antidrif

 #!/usr/bin/env pybricks-micropython
from pybricks.hubs import EV3Brick
from pybricks.ev3devices import GyroSensor
from pybricks.parameters import Port
from pybricks.tools import wait

ev3 = EV3Brick()

# --- CONFIGURACIÓN DEL SENSOR ---
# Nota: Estoy asumiendo el Puerto S1, cámbialo si lo tienes en otro lado
gyro = GyroSensor(Port.S1)

print("=============================================")
print("   INICIANDO CALIBRACIÓN DE GIROSCOPIO       ")
print("=============================================")
print("-> ¡NO MUEVAS EL ROBOT PARA NADA! <-")

# Pitido de advertencia inicial
ev3.speaker.beep(frequency=600, duration=300)

# Forzamos un reset inicial del hardware
gyro.reset_angle(0)
wait(200)

# Parámetros del contador (5 segundos = 50 ciclos de 100ms)
contador_estabilidad = 0
CICLOS_TOTALES = 50  
INTERVALO_MS = 100

while contador_estabilidad < CICLOS_TOTALES:
    
    angulo_actual = gyro.angle()
    velocidad_actual = gyro.speed() # Mide los grados por segundo que registra
    
    # Calculamos el progreso en segundos para mostrarlo en consola
    segundos_estables = (contador_estabilidad * INTERVALO_MS) / 1000.0
    
    print("Ángulo:", angulo_actual, "° | Velocidad:", velocidad_actual, "°/s | Tiempo Estable:", segundos_estables, "s")
    
    # CONDICIÓN DE REINICIO: Si el ángulo se desvía de 0 o detecta velocidad angular
    if angulo_actual != 0 or velocidad_actual != 0:
        print("ALERT: ¡Deriva o movimiento detectado! Reseteando contador...")
        
        # Ejecutamos el reset físico del sensor
        gyro.reset_angle(0)
        contador_estabilidad = 0
        
        # Espera de 200ms para dejar que el hardware del EV3 asiente el nuevo cero
        wait(200) 
    else:
        # Si se mantuvo perfectamente inmóvil en este ciclo, sumamos progreso
        contador_estabilidad += 1
        
    wait(INTERVALO_MS)

# =============================================================================
# CALIBRACIÓN COMPLETADA CON ÉXITO
# =============================================================================
print("\n=============================================")
print("¡CALIBRACIÓN EXITOSA! Giroscopio listo.")
print("Se mantuvo perfectamente inmóvil por 5 segundos.")
print("=============================================")

# Pitido largo de éxito
ev3.speaker.beep(frequency=880, duration=600)

 



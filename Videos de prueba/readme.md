## 🎥 Bitácora de Pruebas y Evolución de Prototipos (WRO 2026)

En esta sección se documenta el proceso experimental y de validación del robot. Las pruebas están organizadas de forma cronológica del proyecto, partiendo desde los subsistemas mecánicos aislados hasta las pruebas de navegación autónoma y evasión de obstáculos.

---

### ⚙️ Fase 1: Ingeniería Mecánica y Pruebas de Actuadores
*Pruebas iniciales de ensamblaje para validar la transmisión de torque, la sincronización de engranajes y la respuesta de los motores de los brazos manipuladores.*

| Prueba / Mecanismo | Descripción Técnica | Video de Referencia |
| :--- | :--- | :--- |
| **01. Banco de Pruebas: Mecanismo de Actuación** | Validación mecánica de la estructura de engranajes y acoples de Lego. Se comprueba el correcto funcionamiento de los motores independientes y los fines de carrera/topes mecánicos de los brazos antes del montaje final en el chasis. | 🔗 [Ver Prueba de Actuadores](https://drive.google.com/file/d/1OKj1bes9MLKAHoJ9wT5PTUmDwdpVN1Ws/view?usp=sharing) |

---

### 🎨 Fase 2: Calibración de Sensores y Visión por Computación
*Desarrollo de software y pruebas de reconocimiento de entorno utilizando sensores de color artificiales y retroalimentación por voz del sistema.*

| Prueba / Sensor | Descripción Técnica | Video de Referencia |
| :--- | :--- | :--- |
| **02. Reconocimiento y Clasificación de Objetos** | Pruebas de lectura en tiempo real frente a bloques de la competencia (Verde y Rojo). El software procesa la información del sensor de color y la interfaz emite alertas auditivas según el elemento detectado, logrando calibrar la tolerancia a la luz ambiental. | 🔗 [Ver Prueba de Sensores y Voz](https://drive.google.com/file/d/1pD1GDVeIaPtUQN4qPY9XguheEe8amylh/view?usp=sharing) |

---

### 🤖 Fase 3: Integración de Chasis y Algoritmos de Navegación Autónoma
*Pruebas de campo con el prototipo completamente armado. Validación de la cinemática del robot, detección perimetral y comportamiento frente a elementos externos.*

| Prueba / Comportamiento | Descripción Técnica | Videos de Referencia |
| :--- | :--- | :--- |
| **03. Evasión y Seguimiento Dinámico (Toma 1)** | El robot ejecuta una rutina de búsqueda en área abierta con el chasis integrado. El algoritmo calcula la proximidad de un objeto esférico (pelota roja) mediante sensores, variando su trayectoria y frenado en función del movimiento dinámico del objeto. | 🔗 [Ángulo General - Parte A](https://drive.google.com/file/d/1zOGOW2E_rQv4tVMwY_KoGFJ2vsckEusN/view?usp=sharing) <br> 🔗 [Ángulo General - Parte B](https://drive.google.com/file/d/1ZB3037ZF_t2KISXfgnCxyET07MERwG2r/view?usp=sharing) <br> 🔗 [Plano Detalle](https://drive.google.com/file/d/1jMoymnt7a0cwyraZeCqr-XZUOC6n-sQb/view?usp=sharing) |
| **04. Evasión y Seguimiento Dinámico (Toma 2)** | Pruebas complementarias de desplazamiento autónomo en pasillo, ajustando los parámetros de velocidad de los motores traseros y la sensibilidad ante la aproximación de la pelota. | 🔗 [Seguimiento en Pasillo](https://drive.google.com/file/d/1Mvlmt1I7TcK7SGeQt7Q1NI9PFN_wf9K_/view?usp=sharing) <br> 🔗 [Retorno y Giro](https://drive.google.com/file/d/1J8J4eSuPvD_glqS64slc2tfECZUNSEvm/view?usp=sharing) |

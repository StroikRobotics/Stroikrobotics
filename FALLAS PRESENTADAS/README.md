## 🛠️ Evolución del Diseño y Optimización Hardware

Durante el desarrollo del robot, identificamos diversos desafíos mecánicos y de ubicación de sensores que afectaban su desempeño en pista. A continuación se detalla la evolución del prototipo y las soluciones aplicadas:

---

### 🟢 Prototipo Inicial (Modelo 1) — Problemas Identificados

* **Ubicación de Sensores Ultrasónicos:** Situados en el centro del robot, lo que dificultaba la navegación en giros cerrados y abiertos.
* **Ajuste de la Pixy Cam:** Montada a una altura excesiva y con una inclinación inadecuada para la detección.
* **Baja Velocidad de Desplazamiento:** La relación de engranajes inicial estaba configurada para priorizar la fuerza sobre la velocidad.

---

### 🟡 Prototipo Intermedio (Modelo 2) — Primeras Mejoras

* **Relación de Transmisión:** Se cambió la relación de engranajes para favorecer la velocidad sobre la fuerza.
* **Ubicación de Ultrasónicos:** Se desplazaron hacia la parte frontal del chasis para mejorar el tiempo de respuesta en esquinas.
* **Soporte de Pixy Cam:** Se rediseñó el soporte agregando firmeza y ajustando el ángulo hacia una posición de visión más favorable.

---

### 🔴 Rediseño del Sistema de Dirección y Tracción (Modelo Final)

A pesar de las mejoras del Modelo 2, el mecanismo de dirección presentaba un amplio juego (punto muerto) debido al deslizamiento en el sistema de corredera, lo que impedía alinear las ruedas de forma precisa.

#### **Solución Aplicada:**
1. **Reconfiguración de Motores:**
   * **Motor L:** Se reasignó a la **tracción**. Esto permitió obtener una aceleración más precisa y mayor fuerza de empuje en la pista.
   * **Motor M:** Se reasignó al **sistema de dirección**.
2. **Mecanismo Directo sin Holgura:** Se eliminó la corredera previa por una transmisión directa que transfiere toda la fuerza del motor sin pérdidas de movimiento.
3. **Control de Precisión:** Se implementó una relación de engranajes **5:1** en la dirección, logrando máxima precisión y estabilidad en el centrado de las ruedas.


# DIAGRAMA DEL CIRCUITO 
## 📊 Arquitectura del Sistema y Conectividad

Para lograr un rendimiento óptimo en los desafíos de la WRO, hemos diseñado un ecosistema de hardware centrado en el bloque inteligente **LEGO Mindstorms EV3**, potenciado con sensores avanzados de visión y proximidad. 

A continuación, se detalla la distribución de componentes y periféricos conectados al bloque central:
### 🔌 Distribución de Puertos (Mapeo de Hardware)

| Puerto | Tipo de Componente | Dispositivo | Función Principal |
| :---: | :--- | :--- | :--- |
| **A** | Entrada / Salida | Motor Auxiliar (Superior Izquierdo) | Actuador secundario / Mecanismo de rotación (0% init) |
| **B** | Salida (Motor) | Motor EV3 (Izquierdo) | Control de tracción / Dirección izquierda |
| **C** | Salida (Motor) | Motor EV3 (Derecho) | Control de tracción / Dirección derecha |
| **D** | Entrada / Salida | Motor Auxiliar (Superior Derecho) | Mecanismo activo / Manipulador de objetos (0% init) |
| **1** | Entrada (Sensor) | Cámara Pixy (PixyCam) | Visión artificial: detección y rastreo de objetos por color |
| **2** | Entrada (Sensor) | Sensor Ultrasónico (Izquierdo) | Lectura de distancia lateral/frontal izquierda y evasión |
| **3** | Entrada (Sensor) | Sensor Ultrasónico (Derecho) | Lectura de distancia lateral/frontal derecha y mapeo |
| **4** | Entrada (Sensor) | Sensor de Tacto / Pulsador | Inicio de rutinas, calibración manual o fin de carrera |


[![Whats-App-Image-2026-05-19-at-7-47-18-AM.jpg](https://i.postimg.cc/wvwX7cmy/Whats-App-Image-2026-05-19-at-7-47-18-AM.jpg)](https://postimg.cc/JDBDFXKM)

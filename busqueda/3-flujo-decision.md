# Flujo de Decisión de la IA en Halo: Percepción, Evaluación, Selección, Ejecución y Cambio de Estado

El comportamiento de la inteligencia artificial en *Halo* no ocurre de forma mágica; responde a un **ciclo de actualización constante** (generalmente evaluado en cada *tick* del motor gráfico o de físicas).

El flujo de decisión de la arquitectura de Árboles de Comportamiento (Behavior Trees) para cualquier personaje —ya sea un Grunt, un Elite o un Marine— se divide en las siguientes **cinco fases**. A continuación, se detalla este flujo con dos ejemplos prácticos: el clásico **"Grunts en pánico"** (comportamiento reactivo) y el táctico **"Elite buscando cobertura"** (comportamiento defensivo).

---

## 1. Percepción (Captura de datos del entorno)

**Explicación técnica:**  
El NPC no "ve" como un humano, sino que el motor del juego alimenta al nodo raíz del árbol con datos crudos mediante sensores virtuales. Estos sensores incluyen:

- **Raycasts (Visión)**: Conos de visión con detección de obstáculos.
- **Triggers de audio (Oído)**: Esferas de detección que reaccionan a disparos o explosiones.
- **Variables globales y de estado (Tacto / Conocimiento)**: Datos compartidos en el *Blackboard* (ej: salud del líder, estado del escudo, posición del jugador).

### Ejemplo práctico: El Grunt (Infantería básica)

El sistema de percepción del Grunt actualiza constantemente sus variables internas. En este escenario específico, el sensor de *conocimiento de escuadrón* detecta un cambio crítico:

```
Variable monitorizada: isEliteAlive = true → isEliteAlive = false
```

Además, el sistema de audición confirma el origen del disparo que mató al líder, actualizando la variable `lastKnownPlayerPosition` con la ubicación exacta del jugador.

### Ejemplo alternativo: El Elite (Rango superior)

El sistema de percepción del Elite monitoriza constantemente su estado de escudo a través de la HUD interna del motor:

```
Variable monitorizada: shieldHealth = 100% → shieldHealth = 15%
```

Al mismo tiempo, el `ThreatAnalyzer` (Statechart de percepción) escanea el entorno en busca de objetos con etiqueta "Cover" (cobertura) dentro de un radio de 20 metros.

---

## 2. Evaluación de condiciones (Procesamiento lógico)

**Explicación técnica:**  
La información percibida fluye por la jerarquía del árbol de comportamiento. Los nodos de control (como los Selectores y las Secuencias) evalúan estas variables de arriba hacia abajo y de izquierda a derecha, buscando la primera rama que devuelva un valor de `True` (Éxito). Esta fase sigue el principio de cortocircuito: en un Selector, si la primera condición es verdadera, se ignoran el resto de condiciones.

### Ejemplo práctico: El Grunt (Pánico)

El árbol de comportamiento del Grunt llega al nodo Selector de mayor prioridad, que gobierna la supervivencia. La evaluación se ejecuta así:

- **Condición A (Alta prioridad):** ¿Está vivo mi líder? → **Falso** (el Elite murió).

Resultado: Al ser falsa la primera condición, el árbol detiene el análisis de las otras ramas menos prioritarias (como "Atacar" o "Buscar granadas") y activa inmediatamente la rama de "Pánico".

### Ejemplo alternativo: El Elite (Cobertura)

El árbol del Elite evalúa sus prioridades en orden estricto:

- **Condición A (Supervivencia):** ¿Mi escudo está por debajo del 20%? → **Verdadero**.
- **Condición B (Táctica):** ¿Hay cobertura disponible en mi radio de acción? → **Verdadero**.

Resultado: El árbol selecciona la rama de "Búsqueda de Cobertura", descartando acciones ofensivas como "Cargar contra el jugador" o "Lanzar granada".

---

## 3. Selección de acción (Determinación del comportamiento)

**Explicación técnica:**  
Una vez que una condición se evalúa como verdadera, el árbol desciende hacia los nodos "Hoja" (Leaf nodes). Estos nodos contienen las acciones concretas (tareas) a ejecutar. Si la rama es una Secuencia, las acciones deben ejecutarse en un orden estricto para que la reacción sea creíble.

### Ejemplo práctico: El Grunt (Secuencia de Pánico)

El nodo selector activa la rama de "Pánico", que está estructurada como una Secuencia de micro-acciones:

```
Rama: PÁNICO (Secuencia)
├── 1. Soltar arma principal
├── 2. Activar animación de "huida torpe" (brazos arriba)
├── 3. Reproducir audio de advertencia
└── 4. Calcular vector de huida (aleatorio, opuesto al jugador)
```

### Ejemplo alternativo: El Elite (Secuencia de Cobertura)

La rama seleccionada es "Buscar Cobertura", también una Secuencia:

```
Rama: BUSCAR COBERTURA (Secuencia)
├── 1. Señalar la posición de cobertura al escuadrón (comunicación tácita)
├── 2. Ejecutar animación de "rodar lateralmente"
├── 3. Moverse al punto de cobertura seleccionado
└── 4. Activar modo "Espera y Recarga" (hasta que escudo llegue al 100%)
```

---

## 4. Ejecución (Llamada a los subsistemas del motor)

**Explicación técnica:**  
El árbol de comportamiento no ejecuta las acciones por sí mismo; envía comandos a los diferentes subsistemas del motor gráfico, de audio y de navegación para materializar la decisión en la pantalla. La ejecución es el momento en que la IA "toca" el mundo real del juego.

### Ejemplo práctico: El Grunt (Ejecución en tres frentes)

La Secuencia de "Pánico" ejecuta simultáneamente (o en cascada rápida) tres comandos hacia los subsistemas:

| Subsistema | Comando | Acción concreta en pantalla |
|---|---|---|
| Animación (Animation Blueprint) | `PlayMontage("Panic_Run")` | El Grunt suelta el arma, levanta los brazos y reproduce un ciclo de carrera descoordinada. |
| Audio (FMOD / Wwise) | `PlayVoiceLine("Grunt_LeaderDown_01")` | Se reproduce una línea de voz aguda: "¡El jefe está muerto, corran!". |
| Navegación (NavMesh / Pathfinding) | `SetDestination(RandomPointBehindSquad)` | El sistema A* del motor traza una ruta de huida hacia un punto aleatorio detrás de la línea de escuadrón. |

### Ejemplo alternativo: El Elite (Ejecución táctica)

La Secuencia de "Cobertura" ejecuta comandos más sofisticados:

| Subsistema | Comando | Acción concreta en pantalla |
|---|---|---|
| Animación | `PlayMontage("Dodge_Roll_Right")` | El Elite realiza un rodaje cinematográfico para esquivar el fuego entrante. |
| Navegación | `SetCoverDestination(NearestCoverObject)` | El Elite se mueve al objeto etiquetado como "Cobertura" más cercano. |
| Estado (Escudo) | `BeginShieldRegeneration()` | Se inicia la recarga activa del escudo (el jugador ve cómo el indicador púrpura se recupera). |
| IA de Escuadrón | `BroadcastSignal("SuppressingFire!")` | Comunica a los Grunts cercanos que disparen fuego de supresión para cubrir su retirada. |

---

## 5. Cambio de estado (Actualización de variables internas)

**Explicación técnica:**  
La ejecución finaliza alterando el estado interno del NPC y su Blackboard (memoria compartida). Este nuevo estado determinará cómo se comportará en el siguiente ciclo de evaluación, alterando sus prioridades futuras y permitiendo que la IA "recuerde" lo que acaba de hacer.

### Ejemplo práctico: El Grunt (De combatiente a fugitivo)

Tras ejecutar la huida, el sistema actualiza sus variables internas:

```
Estado interno (Enum): COMBATE_TÁCTICO → HUIDA
Blackboard:
  - isPanicActive = true
  - panicTimer = 5.0 segundos
  - ignoreCombatOrders = true (ignora órdenes de ataque)
  - fleeDirection = Vector3 (aleatorio)
```

**Efecto en el siguiente ciclo:**  
En los siguientes ticks del juego, el Grunt ignorará por completo eventos como "jugador a la vista" o "arma disponible en el suelo". Su árbol de comportamiento, al leer que `isPanicActive = true`, evitará todas las ramas de ataque y solo reevaluará la rama de huida hasta que el temporizador (`panicTimer`) llegue a cero o el Grunt esté muy lejos del peligro. Una vez que el temporizador expira, el estado interno volverá a `ALERTA` y el Grunt comenzará a reagruparse con otros aliados.

### Ejemplo alternativo: El Elite (De ofensivo a defensivo)

Tras ejecutar la cobertura, el Elite actualiza su estado:

```
Estado interno (Enum): ATAQUE_ACTIVO → COBERTURA_DEFENSIVA
Blackboard:
  - shieldRegenerating = true
  - targetCoverPosition = Vector3 (coordenadas del muro)
  - holdPositionUntilShieldFull = true
```

**Efecto en el siguiente ciclo:**  
Mientras el Elite esté en `COBERTURA_DEFENSIVA`, su árbol de comportamiento no evaluará las ramas de "Carga Berserker" ni "Flanqueo". Solo escuchará al nodo de "Espera" que monitoriza su escudo. Cuando `shieldHealth` llegue nuevamente al 100%, el estado interno volverá a `ATAQUE_ACTIVO` y el árbol seleccionará la siguiente acción ofensiva (como lanzar una granada o reanudar el avance).

---

## Conclusión: El ciclo continuo

La IA de Halo ejecuta este flujo (Percepción → Evaluación → Selección → Ejecución → Cambio de Estado) en un bucle continuo, normalmente varias veces por segundo (30-60 veces en consolas de la época).

Este diseño en capas asegura que:

- Los **Grunts** reaccionen al caos y al miedo de forma instantánea.
- Los **Elites** tomen decisiones tácticas basadas en su supervivencia.
- El sistema global (**Encounter Logic**) pueda inyectar variables que alteren el flujo de todos los NPCs al mismo tiempo (ej: si el jugador tiene la Escopeta y está en rango cercano, se incrementa la prioridad de la rama "Retirada" en todos los árboles).

El resultado no es una IA que simplemente "gana", sino una que mantiene la ilusión de vida y la diversión táctica en cada combate, haciendo que cada encuentro en Halo se sienta único y reactivo al jugador.

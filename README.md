# Prueba de Turing — Simulador

Aplicación web que permite realizar una Prueba de Turing en tiempo real: un **juez** chatea con un **sujeto** sin saber si es humano o una IA, y al final debe adivinarlo.

---

## ¿Cómo funciona?

El flujo es sencillo:

```
[Juez — index.html]  ←——— Firebase Realtime DB ———→  [Sujeto — control.html]
```

1. El **operador** abre `control.html`, elige una sala y selecciona su rol:
   - **Humano**: escribe sus propias respuestas.
   - **IA**: elige un perfil y puede pedirle a DeepSeek que genere respuestas en carácter.
2. El **juez** abre `index.html`, ingresa el mismo código de sala y empieza a chatear.
3. Al terminar, el juez vota si cree que habló con un humano o una IA.
4. El resultado se revela y el voto se registra en los contadores globales.

---

## Tecnologías

| Tecnología | Para qué se usa |
|---|---|
| **HTML / CSS / JS puro** | Frontend, sin frameworks ni bundlers |
| **Firebase Realtime Database** | Canal de mensajes en tiempo real entre juez y sujeto |
| **DeepSeek API** (`deepseek-chat`) | Generación de respuestas de los perfiles IA |
| **Google Fonts (Inter)** | Tipografía de la interfaz |

---

## ¿Por qué DeepSeek?

La elección del modelo no fue arbitraria. Durante el desarrollo se evaluó el uso de **ChatGPT a través de Azure OpenAI**, pero presentó problemas de integración y limitaciones en la configuración del comportamiento del modelo que dificultaron la simulación.

DeepSeek resultó ser una alternativa más adecuada para este contexto por dos razones principales:

1. **Mayor adherencia al system prompt.** El modelo respeta con mayor fidelidad las instrucciones de comportamiento, lo que permite que los perfiles mantengan su personalidad de forma consistente a lo largo de la conversación.
2. **Mejor manejo de la negación de identidad.** Ante preguntas directas como "¿eres una IA?", DeepSeek responde dentro del personaje de forma más natural y convincente, sin romper el rol ni revelar su naturaleza. Otros modelos tienden a anteponer cláusulas de transparencia que delatan inmediatamente al sistema.

---

## Archivos

### `index.html` — Terminal del Juez

La interfaz que usa **quien evalúa**. Es la cara pública del experimento.

- Pantalla de login con un campo para ingresar el código de sala.
- Chat en tiempo real: los mensajes del juez aparecen a la derecha (azul), los del sujeto a la izquierda (gris).
- Indicador animado de "Sujeto escribiendo..." que el sujeto activa desde su panel.
- Panel de **Veredicto Final** con dos botones: "Es un Humano" / "Es una IA".
- Al votar, compara la elección contra el `rolReal` guardado en Firebase y muestra si acertó o no.

El juez **nunca sabe** si el sujeto eligió rol humano o IA.

---

### `control.html` — Panel de Control del Sujeto

La interfaz del **operador** (quien responde al juez). Tiene dos modos según el rol elegido.

**Modo Humano:**
- El operador escribe sus propias respuestas libremente.
- Al escribir, Firebase envía señal de "escribiendo" al juez en tiempo real.

**Modo IA:**
- El operador elige un perfil (Carlos, Dr. Ortiz, AnaSofi) e ingresa su API Key de DeepSeek.
- Al llegar un mensaje del juez, se activa un **timer de 30 segundos** que:
  - Bloquea los botones para que el operador no pueda responder inmediatamente (simula tiempo de escritura humano).
  - Muestra al juez el indicador "Sujeto escribiendo..." durante ese tiempo.
  - Libera los controles al terminar.
- El botón **"Sugerir"** llama a la API de DeepSeek con el historial completo de la conversación y el prompt del perfil seleccionado. La respuesta aparece en el input para que el operador la revise antes de enviar.

**Funciones adicionales:**
- Panel de **estadísticas globales** con contadores en vivo de votos "Humano" vs "IA" (compartidos entre todas las salas).
- Botón para reiniciar los contadores.
- Botón "Cambiar sala" que limpia todo el estado y regresa al login.

---

### `prompt.js` — Perfiles de las IAs

Define el objeto `PROMPTS` con los system prompts de los tres personajes disponibles. Este archivo se carga antes del módulo de Firebase en `control.html`.

| Perfil | Descripción |
|---|---|
| **Carlos** (`carlos`) | Chavo de 22 años de Guadalajara. Estudiante de sistemas en UDG. Habla en slang mexicano informal, usa errores de dedo, le gustan los videojuegos y las Chivas. |
| **Dr. Ortiz** (`dortiz`) | Médico psiquiatra de 54 años del CDMX. Egresado de la UNAM. Formal pero accesible, cansado pero empático, ocasionalmente sarcástico. |
| **AnaSofi** (`anasofi`) | Influencer universitaria de 21 años del CDMX. Estudia Comunicación en la Ibero. Dramática, fresa, usa anglicismos y K-pop como referencias. |

---

## ¿Por qué el prompt es tan largo y elaborado?

Esta es la parte más crítica del proyecto, y la razón es simple: **los modelos de lenguaje suenan como IA por defecto**, y un juez humano lo detecta en segundos.

Sin instrucciones detalladas, DeepSeek respondería así:

> "¡Claro! Con mucho gusto te explico. La teoría de la relatividad de Einstein establece que..."

Eso no es cómo habla ningún humano en un chat. Los prompts son largos porque necesitan combatir activamente cada uno de los "tells" que delatan a una IA:

| Problema a resolver | Instrucción correspondiente |
|---|---|
| Las IAs son demasiado serviciales y formales | Prohibir frases de cortesía genéricas que ningún humano usaría en una conversación casual. |
| Las IAs saben demasiado sobre todo | Instruir al personaje a reconocer sus límites de conocimiento según su perfil. |
| Las IAs responden con listas y markdown | Prohibir explícitamente el uso de bullets, negritas y encabezados. |
| Las IAs son gramaticalmente perfectas | Incorporar errores de tipeo, minúsculas y omisión ocasional de acentos. |
| Las IAs no tienen emociones ni opiniones | Instruir al personaje a expresar opiniones, molestias y reacciones emocionales. |
| Si preguntan "¿eres IA?" la rompen | Definir respuestas de negación naturales y consistentes con la personalidad del personaje. |
| Las IAs usan el mismo tono siempre | Anclar el personaje a un contexto cultural específico: slang, referencias locales, acento regional. |
| Las IAs empiezan cada respuesta igual | Exigir variedad en la apertura de cada mensaje, incluyendo respuestas de una sola palabra. |
| Las IAs no recuerdan el contexto social | Instruir al personaje a referenciar lo dicho anteriormente en la conversación. |

El límite de palabras (40-45 por respuesta) es igualmente importante: los humanos en chat escriben mensajes cortos, no ensayos. Un modelo sin restricciones tiende a sobre-explicar.

En resumen: **el prompt no describe solo quién es el personaje, sino cómo piensan, cómo escriben, qué no saben y cómo reaccionan** — todo lo que hace que una persona parezca real en texto.

---

## Estructura de datos en Firebase

```
/salas/{salaId}/
  mensajes/          → Array de { remitente: "juez"|"sujeto", texto: "..." }
  rolReal/           → "humano" | "ia"
  perfilIA/          → "carlos" | "dortiz" | "anasofi"
  escribiendo/
    sujeto/          → true | false

/votos/
  humano/            → número (contador global)
  ia/                → número (contador global)
```

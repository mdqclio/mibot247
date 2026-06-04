# Wisdom — aprendizajes

Aprendizajes durables del proyecto. Agregar arriba; no borrar lo viejo.

## n8n

- **Los nodos HTTP Request PISAN `$json`** con su respuesta → las referencias `$json.x` de nodos anteriores se rompen aguas abajo. Para valores que tienen que sobrevivir varios nodos, usar un nodo dedicado referenciado por nombre (ej. `Reply Final`) en vez de `$json`. Misma familia que "calcular el timestamp una sola vez en un Set node y referenciarlo".

- **System prompts grandes dentro del expression `{{ }}`** del nodo LLM rompen el parser de n8n con concatenación/llaves pesadas (`invalid syntax`). Armar el system en un Code node (template literal JS) y que el LLM solo referencie el string.

- **Verificar los shapes reales de Firebase ANTES de codear.** `precios.habitaciones` es un **ARRAY** (no objeto keyed por hab) → normalizar a mapa. `beds` usa claves `${hab}-1` (ej `"1-1"`), no `"hab-1"`. La doc/los supuestos estaban mal; el GET de prueba lo reveló.

## LLM

- **Los LLM son poco confiables con matemática de días de la semana/fechas** (resolvió "el finde" a lunes-martes). Hacer la cuenta de calendario en n8n (luxon, `setZone`) e inyectar los anclajes ya resueltos; el LLM usa los anclajes, nunca calcula el día de la semana.

- **Al ganar una capacidad nueva, auditar el system prompt por instrucciones viejas que la contradigan.** Quedó viva la deflección "derivá disponibilidad a un humano" después de que el bot ya calculaba disponibilidad. Cuidado extra: el LLM escribe su `reply` ANTES de que n8n calcule — en el caso completo lo pisamos con el template, pero el caso incompleto usa el reply del LLM → ese reply tiene que estar alineado con la capacidad real.

- **Intents multi-turno necesitan memoria conversacional.** Sin ella, cada turno extrae solo del mensaje actual y los parámetros nunca se combinan (fechas en un turno, personas en otro → nunca calcula). Solución: leer los últimos N mensajes de Firebase (fuente de verdad, sobrevive reinicios) y pasarlos como historial al LLM.

## Producto

- **Espejo fiel.** El bot lee precio/capacidad/disponibilidad con la lógica real del sistema de reservas. Data mal cargada (temporadas sin rangos → siempre precio base; capacidad UI vs dato real) es problema del dueño, no del bot, y se autocorrige cuando se arregla la data.

- **Números de negocio por template, no por LLM.** Generar la respuesta con números (precio/disponibilidad) por template; nunca dejar que el LLM los redacte/invente.

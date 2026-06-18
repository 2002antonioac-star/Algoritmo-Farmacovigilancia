
# Título: "Algoritmo para la extracción y verificación de reacciones adversas en textos narrativos"
## Autor: "Antonio Albarrán Carlier"

A continuación se aborda el algoritmo desarrollado para la extracción y verificación de reacciones adversas a medicamentos partiendo de textos narrativos.
# Explicación de herramientas empleadas

El algoritmo de inteligencia artificial está basado en las tres siguientes herramientas instaladas en un mismo equipo:\
-**Docker (v29.3.51)**: Este actúa como soporte de contenedores.\
-**n8n (v1.1222.5)**: Se instaló sobre una instancia de Docker, actuando como la plataforma base sobre la que se ejecuta el algoritmo.\
-**Ollama (v0.30.7)**: Este se instaló directamente sobre el dispositivo, fuera de la instancia de Docker. Este actúa como servidor local para la ejecución de modelos de lenguaje.

Además de estas herramientas que requieren instalación, se empleó la API CIMA REST, accedida mediante consultas HTTP.

## Docker

Docker es una herramienta que permite empaquetar una aplicación con todos sus componentes en "contenedores" dentro del espacio virtual del equipo, lo que garantiza la reproducibilidad del algoritmo en cualquier máquina que ejecute el mismo contenedor, evita conflictos entre versiones del software y permite un despliegue aislado.

Esta aplicación se empleó como base, instalándose sobre un contenedor la aplicación n8n. Así, para ejecutar el algoritmo se ha de instalar una instancia de n8n en un Docker y activar el contenedor en la aplicación Docker desde la aplicación de escritorio o terminal.

## n8n 

n8n es una plataforma de código abierto que permite el diseño de flujos de trabajo basado en interfaces de nodos. Los usuarios conectan componentes modulares como peticiones HTTP, lectura o escritura de archivos, o el uso de agentes de IA para crear flujos de datos automatizados (Eraña-Díaz et al., 2026). Esto permite transparencia al darse una representación visual del flujo de datos, reproducibilidad al permitir exportar los flujos en código JSON y flexibilidad para implementar diferentes códigos, bases de datos, APIs y agentes de IA.  

Esta aplicación se instaló sobre docker, usándose la versión n8n (v.1.1222.5).

Esta plataforma es sobre la que se ejecuta el flujo de trabajo que comprende el algoritmo desarrollado, por lo que para reproducirlo se habrá de colocar el código exportado sobre un proyecto local de n8n. Los flujos de n8n se exportan como un código JSON, siglas de JavaScript Object Notation, es un formato basado en texto usado para el almacenamiento e intercambio de datos estructurados entre sistemas. Así, abriendo un nuevo algoritmo de n8n se puede pegar directamente el código de otro flujo y ejecutarlo. 

## Ollama

Ollama es un espacio de trabajo de código abierto diseñado para permitir ejecutar LLM de manera local, sin depender de servicios situados en la nube (Eraña-Díaz et al., 2026). Esto permite emplear datos clínicos con la seguridad de que no se comparte información de manera externa, pudiendo integrarse directamente en flujos de n8n, contando esta plataforma con un sub-nodo de "Ollama Model" dentro del nodo "LLM Simple Model", lo que permite seleccionar el modelo de lenguaje y las especificaciones de la ejecución del nodo.

La aplicación de Ollama empleada se instaló sobre la memoria central del propio dispositivo, sin emplear contenedores de Docker, empleándose la versión Ollama (v0.30.7). El flujo se conecta a ella mediante una credencial creada desde la propia aplicación de escritorio de Ollama, de tipo "Ollama API" apuntando al servicio local.

## CIMA REST

CIMA REST es una API (Application Programming Interface, o interfaz de programación de aplicaciones) de la AEMPS que permite la consulta automatizada y estructurada en CIMA de los datos de los diferentes medicamentos autorizados en España. Permite realizar peticiones estructuradas para extraer información como las fichas técnicas o el número de identificación de los medicamentos (AEMPS, s.f.).

Esta herramienta no se instaló sobre el propio dispositivo, sino que se empleó accediendo a la API mediante solicitudes HTTP. El acceso se realizó sobre el endpoint `https://cima.aemps.es/cima/rest/medicamentos`, que admite distintos parámetros de consulta. En el algoritmo se emplean dos tipos de consulta sobre este endpoint:

- **Consulta por nombre comercial** (`?nombre=`): busca el medicamento por su nombre comercial. Esta se emplea como primera consulta y como consulta de respaldo.
- **Consulta por principio activo** (`?practiv1=`): busca el medicamento por su principio activo. Actúa como segunda vía cuando la búsqueda por nombre no devuelve resultados válidos.

De forma complementaria, una vez identificado el número de registro del medicamento, se descarga la ficha técnica completa en HTML desde `https://cima.aemps.es/cima/dochtml/ft/{nregistro}/FT_{nregistro}.html` para extraer sus secciones clínicas.

## Código empleado

El flujo, que se exporta de n8n en formato JSON, presenta además de los nodos propios de n8n algunos nodos de código JavaScript (y en una ocasión Python, para el trabajo sobre HTML), empleados para automatizar tareas como el filtrado según determinados criterios, la estructuración de la información entre nodos y la limpieza de datos.

# Explicación de los modelos empleados

Se emplearon dos modelos de la familia Qwen ejecutados sobre la plataforma Ollama.

## qwen2.5:14b-instruct-q5_K_M

Este modelo fue empleado para procesar la información que necesitaba contar con una estructura de datos estricta, siendo los modelos *instruct* ideales para esto. En el flujo se usa en los nodos de extracción, agrupación, normalización de fármacos y verificación, varios de ellos configurados con salida forzada en formato JSON (`format: json`) y ventanas de contexto de 16.384 a 32.768 tokens según la carga de la tarea. 

Es un modelo que fue subido al repositorio de Ollama en septiembre de 2024, contando con soporte multilingüe que lo hace idóneo para sus tareas.

Para importar este modelo en un Ollama local, se ha de ejecutar el siguiente comando en el terminal del sistema:

```
ollama run qwen2.5:14b-instruct-q5_K_M
```
Las especificaciones del modelo se pueden encontrar en su directorio del repositorio de Ollama:
https://ollama.com/library/qwen2.5:14b-instruct-q5_K_M

## qwen3.6:27b

Este modelo, con su mayor capacidad de razonamiento (derivada de un entrenamiento más reciente y un mayor número de parámetros), resulta idóneo cuando se necesita mayor capacidad de interrelación en la IA o cuando es necesario emplear terminología médica, ya que a más actualizado esté un modelo, mayor es la probabilidad de que se haya entrenado sobre una base de datos que contenga estos términos. En el flujo se reserva para la tarea de extracción inicial, que requiere razonamiento clínico sin un formato de salida estricto, y para la normalización de nombres de medicamentos (se beneficia del mayor entrenamiento y se empleó el ajuste `temperature`, fijándolo en 0 para que el modelo actúe con la menor libertad posible).

El modelo empleado fue el disponible en el repositorio de Ollama en abril de 2026, contando con soporte multilingüe que lo hace idóneo para sus tareas.

Para importar este modelo en un Ollama local, se ha de ejecutar el siguiente comando en el terminal del sistema. Cabe destacar que la versión empleada en este estudio corresponde a la primera versión disponible en Ollama, de abril de 2026, y no a la más actualizada que se importaría al ejecutar este comando, ya que este descargaría la versión más reciente del repositorio, que a 17 de junio de 2026 era la actualizada en junio de 2026. Para poder emplear el modelo de Ollama usado en este algoritmo, habría que descargarlo directamente de un repositorio en línea o haberlo descargado antes de la actualización.

```
ollama run qwen3.6:27b
```

Las especificaciones del modelo se pueden encontrar en su directorio del repositorio de Ollama:
https://ollama.com/library/qwen3.6:27b

# Con respecto a la ejecución del código

Para poder emplear el siguiente código, se debe ejecutar el código JSON sobre una instalación de n8n en un contenedor local de Docker, empleado Ollama desde el equipo tras instalar los modelos especificados. Los pasos a seguir, tras descargar e instalar este software, serían los siguientes:

-Tener Docker en ejecución, ya sea desde la aplicación de escritorio o mediante terminal, y levantar el contenedor de n8n.\
-Tener Ollama instalado en el equipo con ambos modelos y configurar en n8n la credencial 'Ollama API', que se ha de crear previamente desde la aplicación de escritorio de Ollama.\
-Crear un nuevo flujo de n8n y pegar el código JSON.\
-Cambiar la ruta de la carpeta vigilada en el nodo de detección inicial y los dos nodos de exportación a CSV por la carpeta compartida del contenedor de n8n (parámetro path, con valor actual `/data/shared/`).

Si solo se desea observar el flujo general, con instalar n8n es suficiente para visualizar la mayoría del código. 

# Resumen del flujo

A continuación se da una breve descripción de las diferentes fases del proceso realizado por este algoritmo tras activarse en n8n:

1. **Entrada del texto narrativo y preparación del texto**: Un nodo trigger detecta cuando se añade un archivo a la carpeta `/data/shared/`, cuando esto se da se carga el archivo, extrae el texto y se prepara este como entrada para el primer agente de IA.

2. **Extracción de RAM**: Un primer agente de IA, el Agente Extracción, extrae de forma literal la información relacionada con posibles reacciones adversas en el texto (medicamento sospechoso, síntoma, fechas y mejora tras retirada), empleando para esto lógica temporal. Tras esto, devuelve una tabla estructurada en formato Markdown.

3. **Agrupación de sintomatología**: Un segundo agente, el Agente Agrupación, agrupa la sintomatología en diferentes reacciones adversas, acercando la terminología empleada en la agrupación a MedDRA en español. Para realizar la agrupación, se instruye a la IA para que solo agrupe cuando el diagnóstico final está explícito en el texto. Este nodo trabaja sobre los síntomas ya extraídos y da como salida una estructura JSON para cada reacción.

4. **Preparación de los datos para la consulta en CIMA**: Tras la salida del agente de agrupación, un nodo de código estructura los datos y asigna un identificador a cada RAM (`med_id`), empleado para evitar cruces entre reacciones. A continuación, un agente de IA limpia y revisa el nombre del medicamento y extrae su vía de administración, su salida se vuelve a estructurar, y finalmente un diccionario clínico sustituye las denominaciones habituales por la nomenclatura de la base de datos CIMA.

5. **Consulta a CIMA y selección del medicamento**: Mediante consultas HTTP, se realizan consultas a CIMA en cascada en el siguiente orden: Nombre Comercial; Principio Activo; Rescate por nombre comercial. En cada paso se emplean nodos `if` para comprobar que se ha dado un resultado y que la vía del medicamento concuerde con la detectada. Un nodo de código asigna puntuaciones para seleccionar la ficha técnica a emplear, priorizando medicamentos comercializados y estableciendo criterios como los principios activos o el nombre del producto. Los medicamentos no localizados en CIMA se desvían a una rama alternativa.

6. **Descarga y procesado de la ficha técnica**: Habiendo seleccionado el número de registro, se solicita la ficha técnica en HTML, tras esto, un nodo de código reestructura la información rescatando los datos previos del medicamento y asociándolos a la ficha descargada, y por último un bloque de Python extrae y formatea las secciones relevantes (4.3 contraindicaciones, 4.4 advertencias, 4.5 interacciones y 4.8 reacciones adversas).

7. **Agentes finales**: Con toda la información necesaria, en primer lugar el Agente Interacciones redacta una nota orientada a posibles interacciones con otros medicamentos, empleando de nuevo el caso narrativo inicial para encontrar la medicación concomitante. Tras esto, el Generador Sinónimos genera sinónimos clínicos de la RAM y de los síntomas, que el Agente Verificación emplea para comprobar si la reacción adversa está recogida en la ficha técnica. Para ello recurre a esos sinónimos y a los apartados 4.3, 4.4 y 4.8, indicando dónde se encuentra y mostrando una breve observación al respecto.

8. **Consolidación y salida**: Mediante diferentes bloques de JavaScript, los resultados se consolidan, eliminando los duplicados mediante `med_id`, formateándose el archivo de salida que se exporta como CSV en `/data/shared/`. Durante el proceso se generan también CSV intermedios para trazabilidad.

# Información de contacto

Este algoritmo ha sido desarrollado por Antonio Albarrán Carlier como parte del proyecto "Identificación de Reacciones Adversas a Medicamentos sobre Historias Clínicas mediante uso de Inteligencia Artificial: Prueba de Concepto (POC)" del Hospital Universitari Vall d'Hebron.

Información de contacto:

- **antonio.albarran.carlier@gmail.com**\
- **antonio.albarran@vhir.org**

# Bibliografía

- Agencia Española de Medicamentos y Productos Sanitarios. (s. f.). *CIMA REST API (versión 1.19)* [Documento técnico]. Recuperado el 23 de mayo de 2026, de https://sede.aemps.gob.es/docs/CIMA-REST-API_1_19.pdf \
- Eraña-Díaz, M. L., Rosales-Lagarde, A., Arango-de-Montis, I., & Velázquez-Monzón, J. A. (2026). Generative AI-Assisted automation of clinical data processing: A methodological framework for streamlining behavioral research workflows. *Informatics, 13*(4), 48. https://doi.org/10.3390/informatics13040048 
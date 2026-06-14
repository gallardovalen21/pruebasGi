# Reporte Técnico: Sistema de Búsqueda Multimodal de Imágenes

## 1. Portada
* **Materia:** Inteligencia Artificial
* **Cátedra:** UTN FRC
* **Cuatrimestre y Año:** 1er cuatrimestre 5k3 2026
* **Grupo:** Grupo 9
* **Integrantes:** Valentin Alberto Gallardo Scaltriti - 92781
            * Mauricio Herrera - 95205
     * Felipe Nicolas Olivera - 95304

  
![Descripción de la imagen](https://drive.google.com/uc?export=view&id=1I73vkkVdQwcAdys9jDYwYdklJytZg4N1)


## 2. Análisis del Dataset (EDA)
Durante la exploración inicial del dataset (Pascal VOC 2012) se obtuvieron las siguientes estadísticas y características:
* **Cantidad total de imágenes:** 17.125
* **Cantidad total de anotaciones de objetos:** 40.138
* **Clases identificadas:** 20 clases correspondientes a Pascal VOC 2012 (person, chair, car, dog, bottle, cat, bird, pottedplant, sheep, boat, aeroplane, tvmonitor, sofa, bicycle, horse, motorbike, diningtable, cow, train, bus). La clase mayoritaria es `person` con 17.401 instancias.
* **Dimensiones de imágenes:**
  * Rango de ancho: 142 - 500 píxeles.
  * Rango de alto: 71 - 500 píxeles.
  * Dimensión más frecuente (Moda): 500 x 375 píxeles.

![Descripción de la imagen](https://drive.google.com/uc?export=view&id=1xjWPQh8pPA242MWI6q6JVb2c2X0nYkBC)
![Descripción de la imagen](https://drive.google.com/uc?export=view&id=1qfz_KqdOexUApRDgE50XqFPsIX7bBBv3)
![Descripción de la imagen](https://drive.google.com/uc?export=view&id=1R9iQigHrm1ucd7NiaekZmcJfIzDJdVee)
## 3. Metodología
El sistema implementa un pipeline de búsqueda multimodal compuesto por los siguientes módulos:
* **Generación de Embeddings (Baseline):** Se utilizó el modelo **CLIP (ViT-B/32)** para proyectar tanto las imágenes del dataset como las consultas de texto en un espacio latente compartido.
* **Búsqueda Vectorial:** Se empleó **FAISS** (índice L2 plano) para realizar la búsqueda de los vecinos más cercanos de manera eficiente.
* **Reformulación con LLM (Agente Phi-3-mini):** Antes de la búsqueda, la consulta original pasa por un agente basado en **Phi-3-mini**. Este agente realiza varias tareas:
  * Detección de idioma (traducción al inglés si es necesario).
  * Expansión de la consulta agregando términos relacionados y variantes para enriquecer el contexto.
  * Extracción de la intención visual positiva y de términos de negación.
  * Validación semántica de la consulta.
* **Esquema de Reranking y Negaciones:** Los candidatos devueltos por FAISS ($k=500$) son reordenados evaluando una combinación lineal del score original de CLIP, el embedding de la consulta positiva, y penalizando el score asociado al embedding de los términos negativos.
* **Decisiones de Diseño:** * Pesos de penalización: Se definió un multiplicador (`NEGATIVE_PENALTY_WEIGHT = 1.40`) alto para castigar severamente las imágenes que contuvieran el concepto negado.
  * Boost de Clases VOC: Se aplicaron bonificaciones adicionales (`VOC_CLASS_BOOST` y `VOC_CLASS_STRONG_BOOST`) a las imágenes cuya clase de bounding box coincidiera con las inferidas de la consulta.


## 4. Resultados
A partir de la traza de ejecución del pipeline, se validó el correcto funcionamiento de la búsqueda y la reformulación. Un caso de prueba destacado fue la consulta *"auto que no sea rojo"*:

* **Búsqueda Baseline:** La búsqueda directa (CLIP + FAISS sin reformulación) recuperó imágenes sin aplicar traducción ni expansión de la consulta, mostrando limitaciones lógicas al tratar de interpretar la restricción de color de forma pura.
* **Reformulación Agéntica (Phi-3-mini):** El agente procesó la consulta y extrajo con éxito la intención visual principal (*'automobile'*) y el concepto negativo a restringir (*'red'*). Esta reformulación logró introducir 5 imágenes nuevas relevantes respecto al baseline, demostrando una mejora clara en la alineación semántica.
* **Esquema de Reranking y Negaciones:** En el último paso, el sistema aplicó el *boost* de las clases VOC y la penalización en el espacio latente para el término 'red'. Este reranking logró rescatar e incorporar 5 imágenes adicionales respecto a la fase del agente, desplazando hacia abajo en los resultados a los vehículos de color rojo y cumpliendo exitosamente con la restricción impuesta por el usuario.

![Descripción de la imagen](https://drive.google.com/uc?export=view&id=1EdOw5p7JLEK_KH4MLoX-n0J6k1Avb9sn)
![Descripción de la imagen](https://drive.google.com/uc?export=view&id=1F_EclVSEsKY3kJwwFIB0oXotKeUF16PT)

  



## 5. Discusión y Análisis Crítico
* **Qué funcionó bien:** El uso del LLM (Phi-3-mini) como agente planificador fue altamente efectivo para sortear la limitación de CLIP con el idioma (pasando de español a inglés) y para desambiguar consultas. El reranking negativo en el espacio de embeddings demostró ser una solución robusta para imponer restricciones que CLIP puro suele ignorar (el problema de *bag of words* de los modelos contrastivos).
* **Limitaciones del enfoque:** El costo computacional de realizar inferencia con un LLM en tiempo real para cada consulta introduce una latencia significativa comparada con un pipeline CLIP-FAISS clásico. Además, las penalizaciones negativas a veces pueden ser demasiado agresivas si el concepto negativo tiene alta superposición contextual con el concepto principal.

## 6. Trabajo Futuro
Si dispusiéramos de más tiempo y recursos computacionales, nos interesaría explorar las siguientes vías para llevar el sistema de búsqueda al siguiente nivel:

*   **Feedback Loop interactivo (RLHF):** Implementar un mecanismo en la interfaz donde el usuario pueda marcar los resultados devueltos como "relevantes" o "no relevantes". Utilizando estos datos, podríamos ajustar dinámicamente los hiperparámetros del *Reranking* (`LAMBDA_WEIGHT`, `NEGATIVE_PENALTY_WEIGHT`) mediante técnicas de *Machine Learning* para que el sistema aprenda de las preferencias del usuario con el tiempo.
*   **Sustitución del Agente de Texto por un VLM (Vision-Language Model):** En lugar de usar Phi-3-mini solo para procesar el texto de entrada, sería fascinante incorporar un modelo multimodal ligero (como LLaVA o Phi-3-Vision). Esto permitiría que el agente no solo reformule la consulta, sino que *observe* los top-5 resultados devueltos por FAISS y genere un resumen en lenguaje natural explicando *por qué* esas imágenes específicas responden a la intención del usuario.

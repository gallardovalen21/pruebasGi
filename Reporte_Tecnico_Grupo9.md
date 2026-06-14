# Reporte Técnico: Sistema de Búsqueda Multimodal de Imágenes

## 1. Portada
* **Materia:** Inteligencia Artificial
* **Cátedra:** UTN FRC
* **Cuatrimestre y Año:** 1r cuatrimestre 5k3 2026
* **Grupo:** Grupo 9
* **Integrantes:** Valentin Gallardo Scaltriti, Mauricio Herrera, Nicolas Olivera


## 2. Análisis del Dataset (EDA)
Durante la exploración inicial del dataset (Pascal VOC 2012) se obtuvieron las siguientes estadísticas y características:
* **Cantidad total de imágenes:** 17.125
* **Cantidad total de anotaciones de objetos:** 40.138
* **Clases identificadas:** 20 clases correspondientes a Pascal VOC 2012 (person, chair, car, dog, bottle, cat, bird, pottedplant, sheep, boat, aeroplane, tvmonitor, sofa, bicycle, horse, motorbike, diningtable, cow, train, bus). La clase mayoritaria es `person` con 17.401 instancias.
* **Dimensiones de imágenes:**
  * Rango de ancho: 142 - 500 píxeles.
  * Rango de alto: 71 - 500 píxeles.
  * Dimensión más frecuente (Moda): 500 x 375 píxeles.

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
A partir de la traza de ejecución del pipeline, se validó el correcto funcionamiento de la búsqueda y la reformulación:
* **Búsqueda Baseline:** Consultas simples como `car` o `boat` se expandieron adecuadamente a descripciones más ricas (e.g., *'vehicle with four wheels'*, *'small sailing vessel with sails'*), mejorando la alineación con las representaciones visuales de CLIP.
* **Esquema de Negaciones:** En la consulta *'mujer con sombrero pero sin lentes'*, el agente identificó exitosamente 'glasses' como un término negativo. El sistema aplicó la penalización en el espacio latente, desplazando hacia abajo en el ranking a las imágenes de mujeres que llevaban anteojos, logrando extraer resultados acordes a la restricción.
* **Consultas complejas:** Para la consulta *'perro en el agua con una pelota en la boca'*, la traducción y validación semántica extrajeron términos visuales clave y clases VOC esperadas (`cat`, `person` descartados si aplicara), logrando identificar la acción compleja.

## 5. Discusión y Análisis Crítico
* **Qué funcionó bien:** El uso del LLM (Phi-3-mini) como agente planificador fue altamente efectivo para sortear la limitación de CLIP con el idioma (pasando de español a inglés) y para desambiguar consultas. El reranking negativo en el espacio de embeddings demostró ser una solución robusta para imponer restricciones que CLIP puro suele ignorar (el problema de *bag of words* de los modelos contrastivos).
* **Limitaciones del enfoque:** El costo computacional de realizar inferencia con un LLM en tiempo real para cada consulta introduce una latencia significativa comparada con un pipeline CLIP-FAISS clásico. Además, las penalizaciones negativas a veces pueden ser demasiado agresivas si el concepto negativo tiene alta superposición contextual con el concepto principal.

## 6. Trabajo Futuro
Si dispusiéramos de más tiempo y recursos computacionales, nos interesaría explorar las siguientes vías para llevar el sistema de búsqueda al siguiente nivel:

*   **Feedback Loop interactivo (RLHF):** Implementar un mecanismo en la interfaz donde el usuario pueda marcar los resultados devueltos como "relevantes" o "no relevantes". Utilizando estos datos, podríamos ajustar dinámicamente los hiperparámetros del *Reranking* (`LAMBDA_WEIGHT`, `NEGATIVE_PENALTY_WEIGHT`) mediante técnicas de *Machine Learning* para que el sistema aprenda de las preferencias del usuario con el tiempo.
*   **Sustitución del Agente de Texto por un VLM (Vision-Language Model):** En lugar de usar Phi-3-mini solo para procesar el texto de entrada, sería fascinante incorporar un modelo multimodal ligero (como LLaVA o Phi-3-Vision). Esto permitiría que el agente no solo reformule la consulta, sino que *observe* los top-5 resultados devueltos por FAISS y genere un resumen en lenguaje natural explicando *por qué* esas imágenes específicas responden a la intención del usuario.

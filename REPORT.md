# REPORTE TÉCNICO - RETO COPY FAIL CVE-2026-31431

## 1. Análisis de la Vulnerabilidad
Este reporte analiza a fondo la vulnerabilidad del módulo algif_aead. El bug raíz se debe a una confusión en el manejo de memoria al procesar operaciones in-place (donde el destino de los datos es el mismo que el origen) versus operaciones out-of-place utilizando la llamada de sistema splice.

Esta vulnerabilidad es extremadamente silenciosa debido a que la corrupción ocurre directamente en el page cache (en la memoria RAM) y no altera inmediatamente los archivos reflejados en el disco duro. Esto permite evadir sistemas de detección basados en integridad de archivos.

En el contexto del curso, un atacante local puede aprovechar esta corrupción de memoria para manipular procesos en ejecución y alterar binarios críticos con permisos setuid (como por ejemplo /usr/bin/su o sudo). Al desviar el flujo de ejecución de un binario setuid, un usuario sin privilegios puede evadir las restricciones normales de inodos y permisos del sistema de archivos, logrando una escalada de privilegios exitosa y obteniendo una shell completa de root (uid=0).

---

## 2. Cuestionario de Evaluación (5 Preguntas del Reporte)

### Pregunta 1: ¿Cuál es el origen o causa raíz de la vulnerabilidad explotada en el módulo algif_aead?
**Respuesta:** La causa raíz es una falla en el manejo de estructuras de dispersión/reunión (SGL) durante operaciones "in-place". El código reutilizaba el mismo búfer (TX SGL) para origen y destino sin la debida separación ni validación de tamaño en llamadas encadenadas como `splice()`, provocando un desbordamiento o corrupción de punteros en las páginas internas del kernel.

### Pregunta 2: ¿Por qué la mitigación temporal mediante 'rmmod' evita que el exploit funcione?
**Respuesta:** El comando `rmmod algif_aead` descarga por completo el módulo vulnerable de la memoria del kernel. Al no estar disponible la interfaz de sockets del tipo `AF_ALG`, cualquier intento del exploit de abrir un socket de cifrado AEAD fallará de inmediato antes de poder invocar las llamadas de sistema corruptas, neutralizando el vector de ataque.

### Pregunta 3: ¿Cómo soluciona el parche aplicado (fix_algif_aead.patch) el problema de manera permanente?
**Respuesta:** El parche introduce un aislamiento estricto al forzar que las operaciones de transmisión utilicen estructuras TX SGL completamente separadas e independientes para los datos de entrada y de salida. Esto evita la superposición de buffers en memoria y previene que una llamada maliciosa pueda corromper el caché de páginas del sistema operativo.

### Pregunta 4: ¿Qué relación tiene el concepto de permisos 'setuid' con el éxito del exploit para obtener root?
**Respuesta:** El exploit aprovecha la vulnerabilidad para corromper el mapa de memoria de un binario que se ejecuta con el bit `setuid` activado (el cual corre con los privilegios del propietario del archivo, en este caso, root). Al alterar la memoria en ejecución de ese proceso privilegiado, el atacante desvías el flujo del programa para ejecutar comandos arbitrarios, heredando legítimamente la identidad `uid=0`.

### Pregunta 5: ¿Por qué este tipo de fallos en el kernel representan un riesgo crítico para entornos multiusuario o compartidos?
**Respuesta:** Porque rompen el principio fundamental de aislamiento del sistema operativo. Un usuario local completamente restringido y sin privilegios puede saltarse las políticas de control de acceso, inodos y directivas de seguridad del sistema de archivos para tomar control total de la máquina física o virtual, comprometiendo la confidencialidad e integridad de todos los demás usuarios.

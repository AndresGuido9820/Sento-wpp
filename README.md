## 1. Contexto del Proyecto

El bot IoT de Sento está diseñado para facilitar la interacción con dispositivos IoT mediante una interfaz familiar como WhatsApp. [cite_start]Los usuarios pueden enviar comandos para obtener el estado de sus dispositivos, consultar datos históricos o configurar alertas, así como recibir notificaciones automáticas de Ubidots cuando se detectan eventos importantes.

[cite_start]La arquitectura del sistema es serverless, lo que garantiza alta escalabilidad, confiabilidad y un bajo costo operativo, aprovechando los servicios de Amazon Web Services (AWS) e integraciones con APIs externas como Twilio (para WhatsApp) y Ubidots (para la gestión de dispositivos IoT y datos de sensores).

## 2. Arquitectura de Alto Nivel

La arquitectura del bot se basa en un enfoque serverless en AWS, optimizada para la escalabilidad y el procesamiento asíncrono de eventos. Los componentes clave incluyen:

* **Twilio WhatsApp API**: Gestiona la recepción de mensajes de usuarios y el envío de notificaciones.
* **AWS API Gateway**: Actúa como el punto de entrada para los comandos de WhatsApp (`/whatsapp`) y las alertas de Ubidots (`/alerts`).
* **AWS Lambda**:
    * `CommandProcessor`: Procesa los comandos de usuario, interactúa con Ubidots para obtener datos y genera las respuestas.
    * `AlertHandler`: Procesa las alertas recibidas de Ubidots a través de una cola SQS y envía notificaciones a los usuarios vía Twilio.
* **Amazon SQS (`AlertQueue`)**: Almacena las alertas de forma asíncrona para asegurar que no se pierdan y sean procesadas de manera confiable.
* **Amazon DynamoDB**: Base de datos NoSQL que almacena información crítica como datos de usuarios, reglas de alertas y detalles de dispositivos.
* **AWS Secrets Manager**: Almacena de forma segura credenciales sensibles, como los tokens de autenticación para Ubidots y Twilio.
* **Ubidots**: Plataforma IoT utilizada para almacenar datos de sensores y configurar eventos que disparan alertas.

Para una representación visual detallada, consulte el `diagrams/architecture.png`.

### Flujo de Interacción: Comandos de Usuario

1.  El usuario envía un comando o selecciona una opción del menú interactivo en WhatsApp.
2.  Twilio recibe el mensaje y lo envía como un webhook POST al endpoint `/whatsapp` de AWS API Gateway.
3.  API Gateway invoca la función `CommandProcessor` (AWS Lambda).
4.  `CommandProcessor` valida la autenticidad del mensaje, verifica los permisos del usuario en DynamoDB y consulta/actualiza datos en Ubidots (utilizando tokens de Secrets Manager).
5.  La respuesta se envía de vuelta al usuario a través de Twilio WhatsApp API.

### Flujo de Interacción: Alertas de Ubidots

1.  Ubidots detecta una condición de alerta (ej., temperatura fuera de rango) y envía un webhook POST al endpoint `/alerts` de AWS API Gateway.
2.  API Gateway encola la alerta en `AlertQueue` (Amazon SQS).
3.  La función `AlertHandler` (AWS Lambda) consume la alerta de SQS.
4.  `AlertHandler` valida los permisos del usuario en DynamoDB y envía una notificación clara al usuario a través de Twilio WhatsApp API.

[cite_start]Este proceso está diseñado para ser rápido (menos de un segundo desde la detección hasta la notificación) y eficiente en costos.

## 3. Modelo de Datos

El modelo de datos está diseñado para soportar las operaciones del bot, garantizando una estructura normalizada y relaciones claras entre las entidades. [cite_start]Se utiliza Amazon DynamoDB como base de datos.

Los esquemas de tabla principales son:

* **`User`**: Almacena información de los usuarios que interactúan con el bot, incluyendo su número de WhatsApp (PK) y su rol (`admin` o `user`).
    * `phone` (PK): String, número de WhatsApp (ej., "+51987654321").
    * `role`: String, define el tipo de acceso (`admin` o `user`).
* **`ClientCompany`**: Registra las empresas cliente.
    * `company_id` (PK): String, ID único de la empresa.
    * `name`: String, nombre de la empresa.
* **`Farm`**: Almacena las granjas, asociadas a una empresa.
    * `farm_id` (PK): String, ID único de la granja.
    * `company_id` (FK): String, ID de la empresa propietaria.
    * `location`: String, ubicación física.
* **`Device`**: Registra los dispositivos IoT, asociados a una granja.
    * `device_id` (PK): String, ID único del dispositivo.
    * `farm_id` (FK): String, ID de la granja a la que pertenece.
    * `type`: String, tipo de sensor (ej., "temperature").
* **`AlertRule`**: Define las reglas de alerta configuradas por los usuarios.
    * `alert_id` (PK): String, ID único de la regla de alerta.
    * `user_phone` (FK): String, número de teléfono del usuario que configuró la alerta.
    * `device_id` (FK): String, ID del dispositivo monitoreado.
    * `condition`: String, condición de la alerta (ej., "> 30").
* **`AlertEvent`**: Registra el historial de las alertas disparadas.
    * `event_id` (PK): String, ID único del evento de alerta.
    * `alert_id` (FK): String, ID de la regla de alerta asociada.
    * `value`: Float, valor que disparó la alerta.
    * `timestamp`: DateTime, fecha y hora del evento.
* **`UserFarmAccess`**: Tabla puente para gestionar los permisos de los usuarios estándar (`user`) a granjas específicas.
    * `access_id` (PK): String, ID único del registro de acceso.
    * `user_phone` (FK): String, número de teléfono del usuario.
    * `farm_id` (FK): String, ID de la granja asignada.

### Relaciones Clave:

* 1 Empresa → N Granjas: `ClientCompany` → `Farm`
* 1 Granja → N Dispositivos: `Farm` → `Device`
* 1 Usuario → N Alertas: `User` → `AlertRule`
* N Usuarios ↔ N Granjas: Resuelto con `UserFarmAccess` (tabla puente).
* 1 Alerta → N Eventos: `AlertRule` → `AlertEvent`

[cite_start]El modelo de datos cumple con la Tercera Forma Normal (3FN), asegurando atomicidad, dependencia completa de la clave primaria y ausencia de dependencias transitivas.

Para un diagrama completo del modelo de datos, consulte el archivo `diagrams/data_model.png`.

## 4. Roles y Permisos

El sistema define dos roles principales para los usuarios:

* **`admin`**: Usuarios con privilegios de gestión. Pueden agregar/eliminar usuarios, asignar granjas, y configurar dispositivos y alertas globales.
* **`user`**: Usuarios estándar con acceso operativo limitado. Solo pueden consultar datos y configurar alertas para las granjas a las que tienen acceso.

[cite_start]Estos roles se gestionan a través del atributo `role` en la tabla `User` y la tabla `UserFarmAccess` para los permisos de granja específicos de los usuarios estándar.

## 5. Pipeline CI/CD con GitHub Actions

[cite_start]Para automatizar el despliegue seguro del sistema, se ha implementado un robusto pipeline de CI/CD utilizando GitHub Actions. Este pipeline asegura la calidad del código, pruebas rigurosas y un despliegue controlado a producción.

El pipeline consta de las siguientes etapas:

1.  **Lint (Validación de Estilo)**:
    * **Tecnología**: `flake8` (Python).
    * **Función**: Verifica el cumplimiento de PEP8 y detecta errores de sintaxis o complejidad excesiva.
    * [cite_start]**Importancia**: Mantiene la consistencia del código y previene errores sutiles.

2.  **Tests Unitarios (Validación de Lógica)**:
    * **Tecnología**: `pytest` + `pytest-cov` (Python).
    * **Función**: Ejecuta pruebas unitarias con una cobertura de código mínima del 80%.
    * [cite_start]**Importancia**: Detecta regresiones tempranas y asegura la fiabilidad del código.

3.  **Package (Empaquetado para AWS Lambda)**:
    * **Tecnología**: `pip` + `zip`.
    * **Función**: Instala dependencias y comprime el código en un archivo ZIP listo para Lambda.
    * [cite_start]**Importancia**: Prepara el artefacto de despliegue.

4.  **Deploy Staging (Despliegue en Entorno de Pruebas)**:
    * **Tecnología**: Terraform + AWS CLI.
    * **Función**: Despliega toda la infraestructura (Lambda, API Gateway, DynamoDB) en un entorno de `staging` aislado.
    * [cite_start]**Importancia**: Permite pruebas de integración sin afectar la producción.

5.  **Pruebas de Integración (Validación End-to-End)**:
    * **Tecnología**: `pytest` (Python).
    * **Función**: Ejecuta pruebas de flujos completos, como el envío de comandos por WhatsApp y el disparo/notificación de alertas.
    * [cite_start]**Importancia**: Valida que todos los componentes interactúen correctamente.

6.  **Aprobación Manual (Control de Calidad)**:
    * **Tecnología**: GitHub Environments.
    * **Función**: Requiere confirmación manual antes del despliegue en producción.
    * [cite_start]**Importancia**: Previene despliegues no verificados.

7.  **Deploy Prod (Despliegue Gradual en Producción)**:
    * **Tecnología**: Terraform + AWS Lambda Aliases.
    * **Función**: Implementa un "canary deployment" (10% de tráfico inicial) y escala al 100% si no hay errores.
    * [cite_start]**Importancia**: Minimiza el riesgo de fallos en producción.

8.  **Rollback Automático (Seguridad)**:
    * **Tecnología**: CloudWatch Alarms + GitHub Actions.
    * **Función**: Si se detecta un aumento de errores (>5% en 5 minutos) en CloudWatch, revierte automáticamente a la versión estable anterior.
    * [cite_start]**Importancia**: Mitiga impactos críticos rápidamente.

Para una representación visual del pipeline, consulte el archivo `diagrams/ci_cd_pipeline.png`.

## 6. Supuestos y Decisiones de Diseño

### Supuestos:

* [cite_start]**Disponibilidad de APIs Externas**: Se asume la disponibilidad y el correcto funcionamiento de las APIs de Twilio y Ubidots para la interacción y gestión de datos.
* **Conectividad de Dispositivos IoT**: Se asume que los dispositivos IoT están correctamente configurados y enviando datos a Ubidots de manera consistente.
* **Formato de Comandos**: Se espera que los comandos de usuario y las estructuras de datos de Ubidots sigan los formatos predefinidos para un procesamiento adecuado.
* [cite_start]**Gestión de Credenciales**: Se asume que las credenciales para AWS, Twilio y Ubidots se gestionan de forma segura a través de AWS Secrets Manager y las GitHub Actions secrets.
* [cite_start]**Volumen de Datos y Tráfico**: Aunque la arquitectura es escalable, se asume que los volúmenes de tráfico iniciales y de datos de IoT están dentro de los límites operativos de los servicios serverless (ej., API Gateway soporta 10,000 solicitudes/segundo; Lambda, 1,000 invocaciones concurrentes).

### Decisiones de Diseño:

* **Arquitectura Serverless en AWS**:
    * [cite_start]**Ventajas**: Alta escalabilidad automática, reducción de costos operativos (pago por uso), menor carga de mantenimiento de infraestructura, y alta disponibilidad inherente.
    * **Alternativas Consideradas**: Despliegues en máquinas virtuales (EC2) o contenedores (ECS/EKS). **Razón del descarte**: Mayor complejidad de gestión, escalabilidad manual o semi-manual, y mayores costos fijos.

* **Uso de Twilio para WhatsApp**:
    * [cite_start]**Ventajas**: Plataforma robusta y probada para la integración de mensajería, facilidad de uso de su API para webhooks y envío de mensajes.
    * **Alternativas Consideradas**: Implementación directa con la API de WhatsApp Business (requiere una configuración más compleja y certificaciones). **Razón del descarte**: Twilio simplifica la integración y la gestión de la mensajería.

* **Uso de Ubidots para IoT**:
    * [cite_start]**Ventajas**: Plataforma dedicada a IoT que simplifica la ingesta, almacenamiento y visualización de datos de sensores, y la configuración de eventos/alertas sin necesidad de desarrollar esa lógica internamente.
    * **Alternativas Consideradas**: Construcción de una solución de ingesta y procesamiento de datos IoT desde cero en AWS (ej., Kinesis, IoT Core). **Razón del descarte**: Ubidots acelera el desarrollo al proporcionar una solución "out-of-the-box" para la gestión de datos IoT y alertas.

* **Amazon DynamoDB como Base de Datos Principal**:
    * **Ventajas**: Base de datos NoSQL totalmente gestionada y serverless, ideal para cargas de trabajo de alto rendimiento y baja latencia, con escalabilidad automática. [cite_start]Adecuada para almacenar datos de usuarios, reglas y dispositivos con esquemas flexibles.
    * **Alternativas Consideradas**: Bases de datos relacionales como RDS (PostgreSQL/MySQL) o bases de datos documentales como MongoDB. **Razón del descarte**: DynamoDB se alinea mejor con la filosofía serverless y el modelo de datos sin relaciones complejas de grafos, ofreciendo mayor escalabilidad y menor sobrecarga operacional.

* **Amazon SQS para Colas de Alertas**:
    * [cite_start]**Ventajas**: Garantiza la durabilidad y el procesamiento asíncrono de las alertas, evitando la pérdida de mensajes y desacoplando los componentes del sistema.
    * **Alternativas Consideradas**: Llamadas directas de Ubidots a Lambda (mayor riesgo de pérdida de datos si Lambda falla o está sobrecargada). **Razón del descarte**: SQS añade una capa de resiliencia crítica al flujo de alertas.

* **Estrategia de Canary Deployment para Producción**:
    * [cite_start]**Ventajas**: Minimiza el riesgo en los despliegues al redirigir gradualmente el tráfico a la nueva versión, permitiendo la detección temprana de problemas y un rollback rápido si es necesario.
    * **Alternativas Consideradas**: Despliegue azul/verde (más complejo de implementar para Lambda) o "all-at-once" (mayor riesgo). **Razón del descarte**: Canary deployment ofrece un buen equilibrio entre seguridad y complejidad para entornos serverless.

* **Rollback Automático con CloudWatch Alarms**:
    * [cite_start]**Ventajas**: Proporciona una capa adicional de seguridad, revirtiendo automáticamente a una versión estable si las métricas de monitoreo detectan anomalías, reduciendo el tiempo de inactividad.
    * **Alternativas Consideradas**: Rollback manual. **Razón del descarte**: La automatización mejora la resiliencia y la capacidad de respuesta ante incidentes.

## 7. Estructura del Repositorio

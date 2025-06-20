# Sento IoT Bot: Integración WhatsApp y Dispositivos IoT

Este repositorio contiene la implementación del bot IoT de Sento, una solución que permite a los usuarios interactuar con dispositivos IoT a través de WhatsApp, procesar opciones de menú y recibir alertas automáticas.

## 1. Contexto del Proyecto

**Sento** es una empresa dedicada a proporcionar soluciones innovadoras para la gestión y monitoreo de dispositivos IoT en diversos sectores. Nuestro enfoque es simplificar la interacción con la tecnología, haciéndola accesible y eficiente para nuestros clientes.

En el sector agrícola y otras industrias que dependen de la monitorización de ambientes (como granjas y almacenes), nuestros clientes ya cuentan con sistemas de monitoreo y análisis de datos a través de plataformas como Ubidots. Sin embargo, existe una problemática recurrente: la **dificultad para que el usuario final, el operario de campo o el dueño de la granja, obtenga información en tiempo real de los sensores IoT** y la **respuesta tardía ante eventos críticos** (como cambios bruscos de temperatura o humedad). Los métodos actuales a menudo implican interfaces complejas o la necesidad de acceso constante a plataformas especializadas, lo que limita la inmediatez y la toma de decisiones ágil para aquellos que necesitan la información al instante.

El **por qué** de esta solución radica en la necesidad de democratizar el acceso a la información crucial de los dispositivos IoT. Buscamos empoderar a los usuarios, permitiéndoles **interactuar de manera intuitiva y recibir alertas proactivas**, sin la barrera de aprender nuevas aplicaciones complejas o tener que acceder constantemente a dashboards. Queremos que la información relevante esté al alcance de la mano, literalmente, a través de una herramienta de uso diario y de fácil acceso como WhatsApp.

Con este bot, **se busca obtener**:
* **Monitoreo en Tiempo Real y de Fácil Acceso**: Los usuarios podrán consultar el estado actual de sus dispositivos IoT, como la temperatura o la humedad, directamente desde WhatsApp, sin necesidad de iniciar sesión en Ubidots.
* **Alertas Personalizadas y Oportunas**: Configuración de reglas de alerta personalizadas y recepción de notificaciones inmediatas cuando los valores de los sensores superen o caigan por debajo de los umbrales definidos, complementando o mejorando el sistema de alertas existente en Ubidots para el usuario final.
* **Acceso Sencillo y Flexible**: Ofrecer una interfaz de usuario familiar y fácil de usar, permitiendo la interacción mediante un menú interactivo numerado o, para usuarios avanzados, mediante comandos textuales.
* **Optimización de Operaciones**: Reducir los tiempos de respuesta ante anomalías y mejorar la eficiencia en la gestión de las granjas o instalaciones monitorizadas, al proporcionar información crítica de forma directa y proactiva a los responsables.

La **solución** propuesta es un bot IoT que integra WhatsApp con dispositivos IoT a través de una arquitectura serverless, escalable y de bajo costo. Utiliza servicios de AWS (API Gateway, Lambda, SQS, DynamoDB, Secrets Manager) e integraciones con APIs externas como Twilio (para la comunicación por WhatsApp) y Ubidots (para la gestión de datos de sensores y el disparo de eventos). Este enfoque garantiza un flujo de datos optimizado y una experiencia de usuario fluida, desde la interacción inicial con un menú interactivo hasta el manejo eficiente de alertas críticas, actuando como una capa de acceso simplificado a la información ya existente en Ubidots.

## 2. Arquitectura de Alto Nivel

La arquitectura del bot se basa en un enfoque serverless en AWS, optimizada para la escalabilidad y el procesamiento asíncrono de eventos. Los componentes clave incluyen:

* **Twilio WhatsApp API**: Gestiona la recepción de interacciones de usuarios y el envío de notificaciones.
* **AWS API Gateway**: Actúa como el punto de entrada para las interacciones de WhatsApp (`/whatsapp`) y las alertas de Ubidots (`/alerts`).
* **AWS Lambda**:
    * `CommandProcessor`: Procesa las opciones seleccionadas por el usuario, interactúa con Ubidots para obtener datos y genera las respuestas.
    * `AlertHandler`: Procesa las alertas recibidas de Ubidots a través de una cola SQS y envía notificaciones a los usuarios vía Twilio.
* **Amazon SQS (`AlertQueue`)**: Almacena las alertas de forma asíncrona para asegurar que no se pierdan y sean procesadas de manera confiable.
* **Amazon DynamoDB**: Base de datos NoSQL que almacena información crítica como datos de usuarios, reglas de alertas y detalles de dispositivos.
* **AWS Secrets Manager**: Almacena de forma segura credenciales sensibles, como los tokens de autenticación para Ubidots y Twilio.
* **Ubidots**: Plataforma IoT utilizada para almacenar datos de sensores y configurar eventos que disparan alertas.

Para una representación visual detallada, consulte el `diagrams/architecture.png`.

### Flujo de Interacción: Opciones de Menú y Comandos de Usuario

1.  Cuando un usuario inicia un chat con el número de WhatsApp de Sento, recibe un menú interactivo con opciones numeradas (ej., "1. Estado actual", "2. Máximo histórico", "3. Configurar alerta").
2.  Al seleccionar una opción (ej., "1"), Twilio envía esta elección al endpoint `/whatsapp` en API Gateway de AWS.
3.  El `CommandProcessor` (Lambda) verifica la autenticidad del mensaje mediante la firma de Twilio y valida los permisos del usuario en DynamoDB.
4.  Para consultas de estado, el sistema recupera los datos de Ubidots (usando credenciales seguras de Secrets Manager) y muestra respuestas claras como "🌡️ Temperatura: 23.5°C".
5.  Si el usuario configura una alerta (ej., "3"), el sistema guarda la regla en DynamoDB y programa notificaciones en Ubidots.
6.  La respuesta se envía de vuelta al usuario a través de Twilio WhatsApp API.

### Flujo de Interacción: Alertas de Ubidots

1.  Cuando Ubidots detecta un valor fuera de rango (ej., temperatura > 25°C), envía una notificación al endpoint `/alerts`.
2.  API Gateway deriva esta alerta a una cola SQS (`AlertQueue`) para garantizar su procesamiento.
3.  El `AlertHandler` (Lambda) consume la alerta, verifica la regla en DynamoDB y envía un mensaje claro al usuario vía Twilio, como "🚨 Alerta: Temperatura = 26.5°C (Límite: 25°C)".
4.  Todo el proceso, desde la detección hasta la notificación, ocurre en menos de un segundo, con un costo mínimo gracias a la arquitectura serverless y sin perder alertas gracias a SQS.

Este proceso está diseñado para ser rápido y eficiente en costos. Los usuarios pueden interactuar tanto con comandos textuales (ej: "/status farm-001") como con el menú numerado, asegurando flexibilidad y facilidad de uso.

**Diagrama de Arquitectura de Alto Nivel:**
(Pegar aquí el diagrama `diagrams/architecture.png`)

## 3. Modelo de Datos

El modelo de datos está diseñado para soportar las operaciones del bot, garantizando una estructura normalizada y relaciones claras entre las entidades. Se utiliza Amazon DynamoDB como base de datos.

Los esquemas de tabla principales son:

* **`User`**: Almacena información de los usuarios que interactúan con el bot.
    * `phone` (PK): String, número de WhatsApp (ID único).
    * `role`: String, `admin` (acceso total) o `user` (acceso limitado).
* **`ClientCompany`**: Registra las empresas cliente.
    * `company_id` (PK): String, ID único.
    * `name`: String, nombre de la empresa.
* **`Farm`**: Almacena las granjas por empresa.
    * `farm_id` (PK): String, ID único.
    * `company_id` (FK): String, empresa dueña.
    * `location`: String, ubicación física.
* **`Device`**: Registra los dispositivos IoT.
    * `device_id` (PK): String, ID único.
    * `farm_id` (FK): String, granja a la que pertenece.
    * `type`: String, tipo de sensor.
* **`AlertRule`**: Define las reglas de alerta.
    * `alert_id` (PK): String, ID único.
    * `user_phone` (FK): String, usuario que la configuró.
    * `device_id` (FK): String, dispositivo monitoreado.
    * `condition`: String, condición (ej: `> 30`).
* **`AlertEvent`**: Registra el historial de alertas.
    * `event_id` (PK): String, ID único.
    * `alert_id` (FK): String, regla asociada.
    * `value`: Float, valor que disparó la alerta.
    * `timestamp`: DateTime, fecha/hora del evento.
* **`UserFarmAccess`**: Permisos de usuarios normales.
    * `access_id` (PK): String, ID único.
    * `user_phone` (FK): String, usuario con acceso.
    * `farm_id` (FK): String, granja asignada.

### Relaciones Clave:

* 1 Empresa → N Granjas: `ClientCompany` → `Farm` (vía `company_id`).
* 1 Granja → N Dispositivos: `Farm` → `Device` (vía `farm_id`).
* 1 Usuario → N Alertas: `User` → `AlertRule` (vía `user_phone`).
* N Usuarios ↔ N Granjas: Resuelto con `UserFarmAccess` (tabla puente).
* 1 Alerta → N Eventos: `AlertRule` → `AlertEvent` (vía `alert_id`).

El modelo de datos cumple con la Tercera Forma Normal (3FN):
* 1FN: Todos los atributos son atómicos.
* 2FN: Atributos no clave dependen completamente de la PK.
* 3FN: No hay dependencias transitivas.

**Diagrama del Modelo de Datos:**
(Pegar aquí el diagrama `diagrams/data_model.png`)

## 4. Roles y Permisos

El sistema define dos roles principales para los usuarios:

* **`admin`**: Usuario con privilegios de gestión.
    * **Permisos Clave**: Agregar/eliminar usuarios, asignar granjas a usuarios, configurar dispositivos y alertas globales.
    * Acceso a todas las empresas/granjas.
* **`user`**: Usuario estándar (solo acceso operativo).
    * **Permisos Clave**: Consultar datos de sus granjas asignadas, configurar alertas personales.
    * Solo ve granjas asignadas en `UserFarmAccess`.

Estos roles se gestionan a través del atributo `role` en la tabla `User` y la tabla `UserFarmAccess` para los permisos de granja específicos de los usuarios estándar.

## 5. Pipeline CI/CD con GitHub Actions

Para automatizar el despliegue seguro del sistema, se ha implementado un robusto pipeline de CI/CD utilizando GitHub Actions. Este pipeline asegura la calidad del código, pruebas rigurosas y un despliegue controlado a producción.

El pipeline consta de las siguientes etapas:

1.  **Lint (Validación de Estilo)**:
    * **Tecnología**: `flake8` (Python).
    * **Función**: Verifica que el código cumpla con PEP8 (estilo estándar de Python), revisa errores de sintaxis, variables no usadas y complejidad excesiva.
    * **Importancia**: Mantiene consistencia en el código y evita errores sutiles.

2.  **Tests Unitarios (Validación de Lógica)**:
    * **Tecnología**: `pytest` + `pytest-cov` (Python).
    * **Función**: Ejecuta pruebas unitarias para validar funciones individuales y mide cobertura de código (mínimo 80% requerido).
    * **Importancia**: Detecta regresiones antes de integrar cambios.

3.  **Package (Empaquetado para AWS Lambda)**:
    * **Tecnología**: `pip` + `zip`.
    * **Función**: Instala dependencias (`requirements.txt`) y comprime el código en un ZIP.
    * **Importancia**: Prepara el artefacto listo para desplegar en Lambda.

4.  **Deploy Staging (Despliegue en Entorno de Pruebas)**:
    * **Tecnología**: Terraform + AWS CLI.
    * **Función**: Despliega toda la infraestructura (Lambda, API Gateway, DynamoDB) en un entorno de staging, usando variables específicas (`env=staging`) para aislar recursos.
    * **Importancia**: Permite probar integraciones (Twilio, Ubidots) sin afectar producción.

5.  **Pruebas de Integración (Validación End-to-End)**:
    * **Tecnología**: `pytest` (Python).
    * **Función**: Prueba flujos completos: envío de comandos por WhatsApp → Respuesta de Lambda → Consulta a Ubidots; y disparo de alertas → Notificación por Twilio.
    * **Importancia**: Valida que todos los componentes funcionen juntos correctamente.

6.  **Aprobación Manual (Control de Calidad)**:
    * **Tecnología**: GitHub Environments.
    * **Función**: Requiere confirmación manual antes de desplegar en producción y notifica al equipo via Slack o email.
    * **Importancia**: Evita despliegues automáticos de código no verificado.

7.  **Deploy Prod (Despliegue Gradual en Producción)**:
    * **Tecnología**: Terraform + AWS Lambda Aliases.
    * **Función**: Implementa canary deployment: enruta el 10% del tráfico a la nueva versión y, si no hay errores en 15 minutos, escala al 100%.
    * **Importancia**: Minimiza riesgo de fallos en producción.

8.  **Rollback Automático (Seguridad)**:
    * **Tecnología**: CloudWatch Alarms + GitHub Actions.
    * **Función**: Si CloudWatch detecta un aumento de errores (>5% por 5 minutos), revierte automáticamente a la versión estable.
    * **Importancia**: Mitiga impactos críticos sin intervención manual.

**Diagrama del Pipeline CI/CD:**
(Pegar aquí el diagrama `diagrams/ci_cd_pipeline.png`)

## 6. Supuestos y Decisiones de Diseño

### Supuestos:

* **Disponibilidad de APIs Externas**: Se asume la disponibilidad y el correcto funcionamiento de las APIs de Twilio y Ubidots para la interacción y gestión de datos.
* **Conectividad de Dispositivos IoT**: Se asume que los dispositivos IoT están correctamente configurados y enviando datos a Ubidots de manera consistente.
* **Formato de Interacción**: Se espera que las opciones de menú y los comandos textuales, así como las estructuras de datos de Ubidots, sigan los formatos predefinidos para un procesamiento adecuado.
* **Gestión de Credenciales**: Se asume que las credenciales para AWS, Twilio y Ubidots se gestionan de forma segura a través de AWS Secrets Manager y las GitHub Actions secrets.
* **Volumen de Datos y Tráfico**: Aunque la arquitectura es escalable, se asume que los volúmenes de tráfico iniciales y de datos de IoT están dentro de los límites operativos de los servicios serverless (ej., API Gateway soporta 10,000 solicitudes/segundo; Lambda, 1,000 invocaciones concurrentes).

### Decisiones de Diseño:

* **Arquitectura Serverless en AWS**:
    * **Ventajas**: Alta escalabilidad automática, reducción de costos operativos (pago por uso), menor carga de mantenimiento de infraestructura, y alta disponibilidad inherente.
    * **Alternativas Consideradas**: Despliegues en máquinas virtuales (EC2) o contenedores (ECS/EKS). **Razón del descarte**: Mayor complejidad de gestión, escalabilidad manual o semi-manual, y mayores costos fijos.

* **Uso de Twilio para WhatsApp**:
    * **Ventajas**: Plataforma robusta y probada para la integración de mensajería, facilidad de uso de su API para webhooks y envío de mensajes.
    * **Alternativas Consideradas**: Implementación directa con la API de WhatsApp Business (requiere una configuración más compleja y certificaciones). **Razón del descarte**: Twilio simplifica la integración y la gestión de la mensajería.

* **Uso de Ubidots para IoT**:
    * **Ventajas**: Plataforma dedicada a IoT que simplifica la ingesta, almacenamiento y visualización de datos de sensores, y la configuración de eventos/alertas sin necesidad de desarrollar esa lógica internamente.
    * **Integración Existente**: Permite aprovechar la inversión y configuración de monitoreo que los clientes ya tienen en Ubidots, añadiendo una capa de acceso más amigable.
    * **Alternativas Consideradas**: Construcción de una solución de ingesta y procesamiento de datos IoT desde cero en AWS (ej., Kinesis, IoT Core). **Razón del descarte**: Ubidots acelera el desarrollo al proporcionar una solución "out-of-the-box" para la gestión de datos IoT y alertas.

* **Amazon DynamoDB como Base de Datos Principal**:
    * **Ventajas**: Base de datos NoSQL totalmente gestionada y serverless, ideal para cargas de trabajo de alto rendimiento y baja latencia, con escalabilidad automática. Adecuada para almacenar datos de usuarios, reglas y dispositivos con esquemas flexibles.
    * **Alternativas Consideradas**: Bases de datos relacionales como RDS (PostgreSQL/MySQL) o bases de datos documentales como MongoDB. **Razón del descarte**: DynamoDB se alinea mejor con la filosofía serverless y el modelo de datos sin relaciones complejas de grafos, ofreciendo mayor escalabilidad y menor sobrecarga operacional.

* **Amazon SQS para Colas de Alertas**:
    * **Ventajas**: Garantiza la durabilidad y el procesamiento asíncrono de las alertas, evitando la pérdida de mensajes y desacoplando los componentes del sistema.
    * **Alternativas Consideradas**: Llamadas directas de Ubidots a Lambda (mayor riesgo de pérdida de datos si Lambda falla o está sobrecargada). **Razón del descarte**: SQS añade una capa de resiliencia crítica al flujo de alertas.

* **Estrategia de Canary Deployment para Producción**:
    * **Ventajas**: Minimiza el riesgo en los despliegues al redirigir gradualmente el tráfico a la nueva versión, permitiendo la detección temprana de problemas y un rollback rápido si es necesario.
    * **Alternativas Consideradas**: Despliegue azul/verde (más complejo de implementar para Lambda) o "all-at-once" (mayor riesgo). **Razón del descarte**: Canary deployment ofrece un buen equilibrio entre seguridad y complejidad para entornos serverless.

* **Rollback Automático con CloudWatch Alarms**:
    * **Ventajas**: Proporciona una capa adicional de seguridad, revirtiendo automáticamente a una versión estable si las métricas de monitoreo detectan anomalías, reduciendo el tiempo de inactividad.
    * **Alternativas Consideradas**: Rollback manual. **Razón del descarte**: La automatización mejora la resiliencia y la capacidad de respuesta ante incidentes.

## 7. Estructura del Repositorio

* **Rollback Automático con CloudWatch Alarms**:
    * [cite_start]**Ventajas**: Proporciona una capa adicional de seguridad, revirtiendo automáticamente a una versión estable si las métricas de monitoreo detectan anomalías, reduciendo el tiempo de inactividad.
    * **Alternativas Consideradas**: Rollback manual. **Razón del descarte**: La automatización mejora la resiliencia y la capacidad de respuesta ante incidentes.

## 7. Estructura del Repositorio

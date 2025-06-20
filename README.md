# Proyecto Sento: Bot de WhatsApp para Monitoreo IoT

Sento se dedica a proporcionar soluciones innovadoras para la gestión y monitoreo de dispositivos IoT en diversos sectores, simplificando la interacción con la tecnología para nuestros clientes. Este proyecto aborda la necesidad de un acceso más fácil y en tiempo real a los datos de sensores IoT para operarios de campo y dueños de granjas, complementando sus sistemas de monitoreo existentes como Ubidots.

Buscamos democratizar el acceso a información crucial de IoT, permitiendo a los usuarios interactuar de manera intuitiva y recibir alertas proactivas directamente a través de WhatsApp.

---

## 1. Contexto del Proyecto

Nuestros clientes en el sector agrícola y otras industrias (granjas, almacenes) utilizan plataformas como Ubidots para el monitoreo de ambientes. Sin embargo, la **dificultad para que el usuario final obtenga información en tiempo real** y la **respuesta tardía ante eventos críticos** son problemas recurrentes. Los métodos actuales a menudo implican interfaces complejas o la necesidad de acceso constante a plataformas especializadas, lo que limita la inmediatez y la toma de decisiones ágil.

**¿Por qué esta solución?**
Para empoderar a los usuarios, permitiéndoles **interactuar de manera intuitiva y recibir alertas proactivas**, sin la barrera de aprender nuevas aplicaciones complejas o tener que acceder constantemente a dashboards. Queremos que la información relevante esté al alcance de la mano, a través de una herramienta de uso diario y de fácil acceso como WhatsApp.

**Con este bot, se busca obtener:**
* **Monitoreo en Tiempo Real y de Fácil Acceso**: Consulta el estado actual de dispositivos IoT (temperatura, humedad) directamente desde WhatsApp, sin iniciar sesión en Ubidots.
* **Alertas Personalizadas y Oportunas**: Configuración de reglas de alerta y recepción de notificaciones inmediatas cuando los valores de los sensores superen o caigan por debajo de umbrales definidos, complementando o mejorando el sistema de alertas de Ubidots para el usuario final.
* **Acceso Sencillo y Flexible**: Interfaz de usuario familiar y fácil de usar mediante un menú interactivo numerado para acceso a información y configuración de alertas.

La **solución** propuesta es una automatización que integra WhatsApp con dispositivos IoT a través de una **arquitectura serverless, escalable y de bajo costo**. Utiliza servicios de AWS (API Gateway, Lambda, SQS, Secrets Manager) e integraciones con APIs externas como **Twilio** (para la comunicación por WhatsApp), **Ubidots** (para la gestión de datos de sensores y el disparo de eventos) y **Supabase (para la base de datos PostgreSQL)**.

---

## 2. Arquitectura de Alto Nivel

La arquitectura del bot se basa en un enfoque serverless en AWS, optimizada para la escalabilidad y el procesamiento asíncrono de eventos. Los componentes clave incluyen:

* **Twilio WhatsApp API**: Gestiona la recepción de interacciones de usuarios y el envío de notificaciones.
* **AWS API Gateway**: Punto de entrada para las interacciones de WhatsApp (`/whatsapp`) y las alertas de Ubidots (`/alerts`).
* **AWS Lambda**:
    * `CommandProcessor`: Procesa las opciones seleccionadas por el usuario, interactúa con Ubidots para obtener datos y genera las respuestas.
    * `AlertHandler`: Procesa las alertas recibidas de Ubidots y envía notificaciones a los usuarios vía Twilio.
* **Supabase (PostgreSQL)**: Base de datos relacional que almacena información crítica como datos de usuarios, reglas de alertas y detalles de dispositivos.
* **AWS Secrets Manager**: Almacena de forma segura credenciales sensibles para Ubidots, Twilio y Supabase.
* **Ubidots**: Plataforma IoT para almacenar datos de sensores y configurar eventos que disparan alertas.

Para una representación visual detallada, consulte el `diagrams/architecture.png`.

### Flujo de Interacción: Opciones de Menú y Comandos de Usuario

1.  Cuando un usuario inicia un chat con el número de WhatsApp de Sento, recibe un menú interactivo con opciones numeradas.
2.  Al seleccionar una opción (ej., "1. Estado actual"), Twilio envía esta elección al endpoint `/whatsapp` en API Gateway.
3.  El `CommandProcessor` (Lambda) verifica la autenticidad del mensaje y valida los permisos del usuario en **Supabase**.
4.  Para consultas de estado, el sistema recupera los datos de Ubidots y muestra respuestas claras (ej., "Temperatura: 23.5°C").
5.  Si el usuario configura una alerta, el sistema guarda la regla en **Supabase** y programa notificaciones en Ubidots.
6.  La respuesta se envía de vuelta al usuario a través de Twilio WhatsApp API.

### Flujo de Interacción: Alertas de Ubidots

1.  Cuando Ubidots detecta un valor fuera de rango (ej., temperatura > 25°C), envía una notificación al endpoint `/alerts`.
2.  API Gateway deriva esta alerta al `AlertHandler`.
3.  El `AlertHandler` (Lambda) consume la alerta, verifica la regla en **Supabase** y envía un mensaje claro al usuario vía Twilio (ej., "Alerta: Temperatura = 26.5°C (Límite: 25°C)").
4.  Todo el proceso está diseñado para ser rápido y eficiente en costos. Los usuarios pueden interactuar tanto con comandos textuales (ej: "/status farm-001") como con el menú numerado.

**Diagrama de interacción de flujo:**
![Diagrama de Interacción de Flujo](https://raw.githubusercontent.com/AndresGuido9820/Sento-wpp/f70380684a2294a5edc2f9655fa21a9a7aee5093/Diagrams/Diagrama%20interacci%C3%B3n%20de%20flujo.png)

**Diagrama de Arquitectura de Alto Nivel:**
![Diagrama de Arquitectura](https://github.com/AndresGuido9820/Sento-wpp/blob/18e8282766f6ef006d04e924f8ce776f1d3d57d1/Diagrams/Arquitectura-sup.png)

---

## 3. Modelo de Datos

El modelo de datos utiliza **Supabase con una base de datos PostgreSQL**, garantizando una estructura normalizada y relaciones claras entre las entidades.

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

**Diagrama Entidad Relación:**
![Diagrama de Arquitectura](https://raw.githubusercontent.com/AndresGuido9820/Sento-wpp/90fd49b1c57ccb242a4ee6f508ce4f20c2f6b34f/Diagrams/Captura%20de%20pantalla%202025-06-20%20105406.png)

---

## 4. Roles y Permisos

El sistema define dos roles principales para los usuarios:

* **`admin`**: Usuario con privilegios de gestión.
    * **Permisos Clave**: Agregar/eliminar usuarios, asignar granjas a usuarios, configurar dispositivos y alertas globales.
    * Acceso a todas las empresas/granjas.
* **`user`**: Usuario estándar (solo acceso operativo).
    * **Permisos Clave**: Consultar datos de sus granjas asignadas, configurar alertas personales.
    * Solo ve granjas asignadas en `UserFarmAccess`.

Estos roles se gestionan a través del atributo `role` en la tabla `User` y la tabla `UserFarmAccess` para los permisos de granja específicos de los usuarios estándar.

---

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
    * **Función**: Despliega toda la infraestructura (Lambda, API Gateway, configuración de conexión a Supabase) en un entorno de staging, usando variables específicas (`env=staging`) para aislar recursos.
    * **Importancia**: Permite probar integraciones (Twilio, Ubidots, Supabase) sin afectar producción.

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
![Pipeline](https://raw.githubusercontent.com/AndresGuido9820/Sento-wpp/3aa4a05cc5cb22e5036cb8b8dc3c5e44c7988078/Diagrams/Pipeline.png)

---

## 6. Supuestos y Decisiones de Diseño

### Supuestos:

* **Disponibilidad de APIs Externas**: Se asume la disponibilidad y el correcto funcionamiento de las APIs de Twilio, Ubidots y **Supabase** para la interacción y gestión de datos.
* **Conectividad de Dispositivos IoT**: Se asume que los dispositivos IoT están correctamente configurados y enviando datos a Ubidots de manera consistente.
* **Formato de Interacción**: Se espera que las opciones de menú y los comandos textuales, así como las estructuras de datos de Ubidots, sigan los formatos predefinidos para un procesamiento adecuado.
* **Gestión de Credenciales**: Se asume que las credenciales para AWS, Twilio, Ubidots y **Supabase** se gestionan de forma segura a través de AWS Secrets Manager y las GitHub Actions secrets.
* **Volumen de Datos y Tráfico**: Aunque la arquitectura es escalable, se asume que los volúmenes de tráfico iniciales y de datos de IoT están dentro de los límites operativos de los servicios serverless (ej., API Gateway soporta 10,000 solicitudes/segundo; Lambda, 1,000 invocaciones concurrentes) y **los límites de la capa de Supabase seleccionada**.

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

* **Supabase (PostgreSQL) como Base de Datos Principal**:
    * **Ventajas**: Base de datos PostgreSQL relacional completamente gestionada, que ofrece la robustez, integridad de datos y flexibilidad de un RDBMS, con el beneficio de ser serverless y escalable. Ideal para datos estructurados con relaciones claras entre entidades. **Supabase** simplifica la gestión y la autenticación.
    * **Alternativas Consideradas**: Bases de datos NoSQL como DynamoDB o MongoDB. **Razón del descarte**: PostgreSQL en **Supabase** permite un modelo de datos relacional más estructurado y el uso de consultas SQL complejas para las necesidades de gestión de usuarios, dispositivos y reglas de alerta, ofreciendo mayor familiaridad y herramientas avanzadas para la gestión de datos tabulares.

* **Amazon SQS para Colas de Alertas**:
    * **Ventajas**: Garantiza la durabilidad y el procesamiento asíncrono de las alertas, evitando la pérdida de mensajes y desacoplando los componentes del sistema.
    * **Alternativas Consideradas**: Llamadas directas de Ubidots a Lambda (mayor riesgo de pérdida de datos si Lambda falla o está sobrecargada). **Razón del descarte**: SQS añade una capa de resiliencia crítica al flujo de alertas.

* **Estrategia de Canary Deployment para Producción**:
    * **Ventajas**: Minimiza el riesgo en los despliegues al redirigir gradualmente el tráfico a la nueva versión, permitiendo la detección temprana de problemas y un rollback rápido si es necesario.
    * **Alternativas Consideradas**: Despliegue azul/verde (más complejo de implementar para Lambda) o "all-at-once" (mayor riesgo). **Razón del descarte**: Canary deployment ofrece un buen equilibrio entre seguridad y complejidad para entornos serverless.

* **Rollback Automático con CloudWatch Alarms**:
    * **Ventajas**: Proporciona una capa adicional de seguridad, revirtiendo automáticamente a una versión estable si las métricas de monitoreo detectan anomalías, reduciendo el tiempo de inactividad.
    * **Alternativas Consideradas**: Rollback manual. **Razón del descarte**: La automatización mejora la resiliencia y la capacidad de respuesta ante incidentes.

---

## 7. Estructura del Repositorio

 ````
sento-whatsapp-bot/
├── .github/
│   └── workflows/
│       └── ci-cd.yml                # ARCHIVO CLAVE: Define tu pipeline de Integración Continua/Despliegue Continuo (CI/CD).
│                                    # Automatiza desde las pruebas de código hasta el despliegue en AWS.
│                                    # Incluye pasos para linting, tests, empaquetado, despliegue a staging,
│                                    # pruebas de integración, APROBACIÓN MANUAL y despliegue a producción.
│
├── src/                             # CARPETA CLAVE: Aquí vive todo el código Python de tu aplicación.
│   ├── common/                      # Código y utilidades COMPARTIDAS entre tus funciones Lambda.
│   │   ├── __init__.py              # Hace que 'common' sea un paquete Python.
│   │   ├── database/                # Configuración del ORM (SQLAlchemy) y modelos de base de datos.
│   │   │   ├── __init__.py
│   │   │   ├── connection.py        # CLAVE: Configura la conexión a Supabase (PostgreSQL) usando SQLAlchemy.
│   │   │                            # Incluye manejo de errores para la conexión y sesiones de DB.
│   │   │   └── models.py            # CLAVE: Define tus tablas de base de datos como CLASES PYTHON (modelos ORM).
│   │   │                            # Aquí se mapean User, ClientCompany, Farm, Device, AlertRule, AlertEvent, UserFarmAccess.
│   │   ├── schemas/                 # CLAVE: Modelos Pydantic para VALIDACIÓN y SERIALIZACIÓN de datos.
│   │   │   ├── __init__.py
│   │   │   ├── user.py              # Esquemas Pydantic para la entidad 'User' (cómo se ve un usuario al crearlo, leerlo, etc.).
│   │   │   ├── device.py            # Esquemas Pydantic para la entidad 'Device'.
│   │   │   ├── alert.py             # Esquemas Pydantic para 'AlertRule' y 'AlertEvent'.
│   │   │   └── base.py              # Esquemas Pydantic base con campos comunes (ej. `created_at`, `updated_at`).
│   │   ├── exceptions.py            # CLAVE: Definición de CLASES DE EXCEPCIONES PERSONALIZADAS.
│   │   │                            # Esto permite un manejo de errores más específico y legible en tu código.
│   │   ├── utils.py                 # Funciones de utilidad general (ej. validar la firma de Twilio, parsear mensajes).
│   │   ├── constants.py             # Constantes que se usan en toda la aplicación (ej. nombres de comandos, mensajes fijos).
│   │   └── config.py                # CLAVE: Módulo para CARGAR DE FORMA SEGURA las configuraciones y secretos.
│   │                                # Usa `.env` en desarrollo local y AWS Secrets Manager en la nube.
│   │
│   ├── command_processor/           # CARPETA CLAVE: Contiene el código de la FUNCIÓN LAMBDA 'CommandProcessor'.
│   │   │                            # Esta Lambda maneja los mensajes entrantes de WhatsApp y los comandos de usuario.
│   │   ├── __init__.py
│   │   ├── app.py                   # CLAVE: El HANDLER PRINCIPAL de la Lambda. Es el punto de entrada de cada evento.
│   │   │                            # Aquí se valida la entrada (Pydantic), se llama a los servicios y se manejan errores generales.
│   │   ├── services/                # Lógica de NEGOCIO específica para procesar comandos.
│   │   │   ├── __init__.py
│   │   │   ├── user_service.py      # Lógica para gestionar usuarios (obtener, crear, validar rol).
│   │   │   ├── device_service.py    # Lógica para interactuar con Ubidots y gestionar dispositivos.
│   │   │   └── alert_config_service.py # Lógica para configurar y almacenar reglas de alerta.
│   │   ├── repositories/            # Capa de ABSTRACCIÓN para interactuar con la base de datos a través del ORM.
│   │   │   ├── __init__.py
│   │   │   ├── user_repo.py         # Métodos CRUD (Crear, Leer, Actualizar, Borrar) para la tabla 'User'.
│   │   │   └── alert_repo.py        # Métodos CRUD para las tablas de 'AlertRule' y 'AlertEvent'.
│   │   ├── requirements.txt         # CLAVE: Dependencias Python ESPECÍFICAS para esta Lambda (Pydantic, SQLAlchemy, etc.).
│   │   └── templates/               # (Opcional) Plantillas para mensajes complejos de WhatsApp (ej. menú dinámico).
│   │       └── menu_messages.py
│   │
│   ├── alert_handler/               # CARPETA CLAVE: Contiene el código de la FUNCIÓN LAMBDA 'AlertHandler'.
│   │   │                            # Esta Lambda se encarga de procesar las alertas recibidas de Ubidots.
│   │   ├── __init__.py
│   │   ├── app.py                   # CLAVE: El HANDLER PRINCIPAL de esta Lambda. Procesa los eventos de alerta.
│   │   │                            # Incluye validación, llamada a servicios y manejo de errores.
│   │   ├── services/                # Lógica de NEGOCIO para manejar las alertas.
│   │   │   ├── __init__.py
│   │   │   └── notification_service.py # Lógica para enviar notificaciones a través de Twilio y consultar reglas de alerta.
│   │   ├── repositories/            # Capa de ABSTRACCIÓN para la interacción con la base de datos para alertas.
│   │   │   ├── __init__.py
│   │   │   └── alert_repo.py        # Métodos CRUD para las tablas de 'AlertRule' y 'AlertEvent'.
│   │   ├── requirements.txt         # Dependencias Python ESPECÍFICAS para esta Lambda.
│   │
│   └── tests/                       # CARPETA CLAVE: Todas las PRUEBAS de tu aplicación.
│       ├── unit/                    # Pruebas UNITARIAS: Verifican componentes aislados (funciones, clases).
│       │   ├── test_db_connection.py # Pruebas para la conexión a la base de datos.
│       │   ├── test_schemas.py      # Pruebas para tus modelos Pydantic (validación de datos).
│       │   ├── test_exceptions.py   # Pruebas para tus clases de excepción personalizadas.
│       │   ├── test_utils.py        # Pruebas para las funciones de utilidad.
│       │   └── command_processor/   # Pruebas unitarias para los servicios y repositorios del procesador de comandos.
│       │   └── alert_handler/       # Pruebas unitarias para los servicios y repositorios del manejador de alertas.
│       └── integration/             # Pruebas de INTEGRACIÓN: Verifican flujos completos y la interacción entre componentes.
│           ├── test_e2e_whatsapp_flow.py # Simula un mensaje de WhatsApp y verifica toda la cadena hasta la respuesta.
│           └── test_ubidots_alert_flow.py # Simula una alerta de Ubidots y verifica la notificación final.
│
├── terraform/                     
│   │                                # Define cómo se construyen y configuran tus recursos en AWS.
│   ├── environments/                # Configuración ESPECÍFICA para cada entorno de despliegue.
│   │   ├── staging/                 # Entorno de pruebas (pre-producción).
│   │   │   └── main.tf              # Recursos y configuraciones para el entorno de staging.
│   │   │   └── variables.tf
│   │   │   └── outputs.tf
│   │   └── production/              
│   │       └── main.tf              # Recursos y configuraciones para el entorno de producción.
│   │                                # Aquí se pueden aplicar estrategias como Canary Deployments para despliegues seguros.
│   │       └── variables.tf
│   │       └── outputs.tf
│   │
│   ├── modules/                     # MÓDULOS Terraform REUTILIZABLES.
│   │                                # Cada módulo define un conjunto de recursos que puedes usar una y otra vez.
│   │   ├── lambda/                  # Módulo para desplegar y configurar FUNCIONES LAMBDA.
│   │   │   └── main.tf              # Define la función Lambda (memoria, runtime, rol IAM) y sus VARIABLES DE ENTORNO.
│   │   │                            # También configura CloudWatch Logs y ALARMAS para la Lambda.
│   │   │                            # ESPECIFICA EL ARCHIVO .ZIP DEL CÓDIGO A DESPLEGAR.
│   │   ├── api_gateway/             # Módulo para configurar API GATEWAY.
│   │   │   └── main.tf              # Define los ENDPOINTS HTTP (`/whatsapp`, `/alerts`), los métodos (POST),
│   │   │                            # y cómo se INTEGRAN con tus funciones Lambda.
│   │   ├── sqs/                     # Módulo para configurar colas SQS (si las usas directamente como recurso de Terraform).
│   │   │   └── main.tf
│   │   ├── secrets_manager/         # Módulo para almacenar SECRETOS en AWS Secrets Manager.
│   │   │   └── main.tf              # CLAVE: Define cómo se crean los secretos (credenciales de Twilio, Ubidots, Supabase)
│   │   │                            # y cómo tus Lambdas tienen permiso para ACCEDER A ELLOS.
│   │   └── iam_roles/               # Módulo para definir ROLES y POLÍTICAS de IAM (manejo de permisos en AWS).
│   │       └── main.tf              # Roles con los PERMISOS MÍNIMOS necesarios para tus Lambdas y API Gateway.
│   │
│   └── main.tf                      # Archivo principal de Terraform para recursos GLOBALES/comunes.
│   └── variables.tf                 # Variables globales compartidas por todos los entornos Terraform.
│   └── outputs.tf                   # Salidas importantes del despliegue Terraform (ej. URLs de API Gateway).
│   └── versions.tf                  # Define las versiones de los providers Terraform que usas.
│
├── diagrams/                        # Contiene todos los DIAGRAMAS VISUALES del proyecto.
│   ├── architecture.png             # Diagrama de ARQUITECTURA de alto nivel.
│   └── flow_interaction.png         # Diagrama de FLUJO de interacción de usuario y alertas.
│   └── pipeline.png                 # Diagrama visual del pipeline CI/CD.
│   └── erd.png                      # Diagrama Entidad-Relación de tu modelo de datos en Supabase.
│
├── docs/                            # Documentación ADICIONAL del proyecto.
│   └── setup_guide.md               # Guía de configuración inicial para nuevos desarrolladores.
│   └── api_spec.md                  # Especificaciones de las APIs externas que consumes (Twilio, Ubidots, Supabase).
│   └── error_handling_guide.md      
├── .env                             # CLAVE: ARCHIVO para VARIABLES DE ENTORNO LOCALES.
│                                   
│                                    # Contiene credenciales y configuraciones para tu desarrollo y pruebas locales.
│                                    
│
├── .gitignore                       # Especifica archivos y carpetas que Git debe IGNORAR (ej. .env, logs, caché).
├── README.md                     
└── requirements-dev.txt         
 ```` 

# Proyecto Sento: Bot de WhatsApp para Monitoreo IoT

Sento se dedica a proporcionar soluciones innovadoras para la gestión y monitoreo de dispositivos IoT en diversos sectores, simplificando la interacción con la tecnología para nuestros clientes. Este proyecto aborda la necesidad de un acceso más fácil y en tiempo real a los datos de sensores IoT para operarios de campo y dueños de granjas, complementando sus sistemas de monitoreo existentes como Ubidots.

Buscamos optimizar el acceso a información crucial de IoT, permitiendo a los usuarios interactuar de manera intuitiva y recibir alertas proactivas directamente a través de WhatsApp.

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
  

**Código pipeline**
````
name: Sento CI/CD Pipeline

on:
  push:
    branches: [ "develop" ]
  pull_request:
    branches: [ "main" ]

env:
  AWS_REGION: us-east-1
  TF_VERSION: 1.5.0
  PYTHON_VERSION: 3.9

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python ${{ env.PYTHON_VERSION }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest pytest-cov
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      - name: Lint with flake8
        run: |
          flake8 src/ --count --show-source --statistics --max-line-length=120

      - name: Run unit tests
        run: |
          pytest src/tests/unit/ -v --cov=src --cov-report=xml
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL_STAGING }}
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY_STAGING }}

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          fail_ci_if_error: false

  package:
    needs: lint-and-test
    runs-on: ubuntu-latest
    outputs:
      lambda_zip: ${{ steps.package-lambda.outputs.lambda_zip }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Package Lambda functions
        id: package-lambda
        run: |
          cd src/command_processor/
          zip -r ../../command_processor.zip .
          cd ../alert_handler/
          zip -r ../../alert_handler.zip .
          echo "lambda_zip=command_processor.zip,alert_handler.zip" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: package
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_STAGING }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_STAGING }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init -backend-config="environments/staging/backend.tfvars"

      - name: Terraform Plan
        run: terraform plan -var-file="environments/staging/variables.tfvars"

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file="environments/staging/variables.tfvars"

  integration-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run integration tests
        run: |
          pytest src/tests/integration/ -v
        env:
          TWILIO_ACCOUNT_SID: ${{ secrets.TWILIO_ACCOUNT_SID }}
          TWILIO_AUTH_TOKEN: ${{ secrets.TWILIO_AUTH_TOKEN }}
          UBIDOTS_TOKEN: ${{ secrets.UBIDOTS_TOKEN_STAGING }}

  deploy-production:
    needs: integration-tests
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init -backend-config="environments/production/backend.tfvars"

      - name: Terraform Plan
        run: terraform plan -var-file="environments/production/variables.tfvars"

      - name: Manual Approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.token }}

      - name: Terraform Apply (Canary)
        run: |
          terraform apply -auto-approve -var-file="environments/production/variables.tfvars" \
            -var="lambda_traffic_percentage=10"
          
          sleep 900  # Espera 15 minutos para monitoreo
          
          terraform apply -auto-approve -var-file="environments/production/variables.tfvars" \
            -var="lambda_traffic_percentage=100"

      - name: Notify Slack on Success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          channel-id: ${{ secrets.SLACK_CHANNEL }}
          slack-message: "✅ Despliegue en producción completado: ${{ github.repository }}@${{ github.sha }}"
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

  rollback:
    if: failure() && needs.deploy-production.result == 'failure'
    runs-on: ubuntu-latest
    needs: deploy-production
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Rollback Lambda to Previous Version
        run: |
          aws lambda update-alias \
            --function-name sento-alert-handler \
            --name PROD \
            --function-version $(aws lambda list-versions-by-function \
              --function-name sento-alert-handler \
              --query "Versions[-2].Version" --output text)

      - name: Notify Slack
        uses: slackapi/slack-github-action@v1.24.0
        with:
          channel-id: ${{ secrets.SLACK_CHANNEL }}
          slack-message: "⚠️ Rollback ejecutado por fallos en producción: ${{ github.repository }}"
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
````

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
## Tabla con costos 
# Desglose de Costos de Servicios

A continuación se presenta una tabla detallada con los costos asociados a los diferentes servicios utilizados.

## Comunicaciones y APIs

### Twilio

| Servicio | Costo por Unidad | Unidad de Facturación | Nivel Gratuito (Mensual) | Notas Clave |
| :--- | :--- | :--- | :--- | :--- |
| **WhatsApp Business API** | $0.005 (adicional) | Por mensaje | N/A | Cargo adicional sobre las tarifas por conversación de Meta. |
| **SMS y MMS** | $0.0083 | Por mensaje | N/A | Envío y recepción. |
| **Voice API (recibir)** | $0.0085 | Por minuto | N/A | |
| **Voice API (hacer)** | $0.014 | Por minuto | N/A | |
| **Functions** | $0.0001 | Por invocación | 10,000 invocaciones | |



---



---

## Base de Datos y Backend

### Supabase

| Plan | Costo Mensual | Base de Datos | Usuarios Activos Mensuales (MAU) | Ancho de Banda | Almacenamiento | Notas |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Plan Gratuito** | $0 | 500 MB | 50,000 | 5 GB | 1 GB | |
| **Plan Pro** | Desde $25 | 8 GB ($0.125/GB extra) | 100,000 ($0.00325/MAU extra) | 250 GB ($0.09/GB extra) | 100 GB ($0.021/GB extra) | Incluye $10 en créditos de cómputo. |
| **Instancia Micro (Cómputo)** | $10 | N/A | N/A | N/A | N/A | 2 núcleos ARM, 1 GB de memoria. Cubierto por créditos en planes de pago. |

---

## Servicios en la Nube (AWS)

### AWS Lambda

| Concepto | Costo | Nivel Gratuito (Mensual) | Notas |
| :--- | :--- | :--- | :--- |
| **Solicitudes** | $0.20 / 1M de solicitudes | 1 millón de solicitudes | |
| **Duración (x86)** | $0.0000166667 / GB-segundo | 400,000 GB-segundos (agregado) | |
| **Duración (Arm)** | $0.0000133334 / GB-segundo | 400,000 GB-segundos (agregado) | Hasta 34% mejor rendimiento/precio. |
| **Lambda@Edge Solicitudes** | $0.60 / 1M de solicitudes | N/A | |

### AWS API Gateway

| Tipo de API | Costo (por 1 millón) | Nivel Gratuito (12 meses) | Notas |
| :--- | :--- | :--- | :--- |
| **HTTP API Calls** | $1.00 | 1 millón de llamadas | Hasta 71% más barato que REST APIs. |
| **REST API Calls**| $3.50 | 1 millón de llamadas | |
| **WebSocket Messages** | $1.00 | 1 millón de mensajes | Mensajes medidos en incrementos de 32 KB. |

### Amazon SQS

| Tipo de Cola | Costo (por 1 millón de solicitudes) | Nivel Gratuito (Indefinido) | Notas |
| :--- | :--- | :--- | :--- |
| **Estándar** | $0.40 | 1 millón de solicitudes | Cada 64 KB de carga útil es 1 solicitud. |
| **FIFO** | $0.50 | 1 millón de solicitudes | Cada 64 KB de carga útil es 1 solicitud. |

### AWS Secrets Manager

| Concepto | Costo | Nivel Gratuito | Notas |
| :--- | :--- | :--- | :--- |
| **Por Secreto** | $0.40 / secreto / mes | Prueba gratuita de 30 días | Los secretos de réplica se facturan aparte. |
| **Llamadas a la API** | $0.05 / 10,000 llamadas | N/A | |



---

## CI/CD y Almacenamiento

### GitHub Actions

| Ejecutor | Costo por Minuto | Minutos Gratuitos (Plan Free) | Notas |
| :--- | :--- | :--- | :--- |
| **Linux 2-core** | $0.008 | 2,000 | Uso gratuito para repositorios públicos. |
| **Windows 2-core**| $0.016 | 2,000 | Multiplicador 2x de minutos. |
| **macOS 3/4-core**| $0.08 | 2,000 | Multiplicador 10x de minutos. |

| Concepto | Costo | Almacenamiento Gratuito (Plan Free) |
| :--- | :--- | :--- |
| **Almacenamiento (sobreuso)** | $0.008 / GB / día | 500 MB |



---

## 7. Estructura del Repositorio

 ````
sento-whatsapp-bot/
├── .github/
│   └── workflows/
│       └── ci-cd.yml                # Pipeline CI/CD automatizado con aprobación manual para producción
│
├── src/
│   ├── common/                      # Código compartido entre Lambdas
│   │   ├── database/                # ORM y modelos de base de datos
│   │   ├── schemas/                 # Validación de datos con Pydantic  
│   │   └── config.py                # Gestión centralizada de configuraciones
│   │
│   ├── command_processor/           # Lambda para mensajes WhatsApp
│   │   ├── app.py                   # Handler principal
│   │   ├── services/                # Lógica de negocio
│   │   └── repositories/            # Acceso a datos
│   │
│   ├── alert_handler/               # Lambda para alertas Ubidots
│   │   ├── app.py                   # Handler principal  
│   │   ├── services/
│   │   └── repositories/
│   │
│   └── tests/                       # Suite de pruebas
│       ├── unit/                    # Pruebas de componentes
│       └── integration/             # Pruebas de flujos completos
│
├── terraform/
│   ├── environments/                # Configuración por entorno
│   │   ├── staging/
│   │   └── production/             
│   │
│   ├── modules/                     # Componentes reutilizables
│   │   ├── lambda/                  # Configuración Lambda
│   │   ├── api_gateway/             # Endpoints REST  
│   │   └── secrets_manager/         # Gestión de credenciales
│   │
│   └── main.tf                      # Configuración global
│
├── diagrams/                       
├── docs/                            # Guías técnicas
├── .env                             # Variables locales (gitignored)
├── .gitignore
└── README.md
 ```` 

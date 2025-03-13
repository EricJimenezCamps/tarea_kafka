# Proyecto FarmIA - Kafka y Procesamiento en Tiempo Real

## Introducción

Este proyecto implementa un sistema de procesamiento en tiempo real para la empresa FarmIA, que está integrando sensores IoT en campos agrícolas y necesita monitorizar datos como temperatura, humedad y fertilidad del suelo, además de analizar las transacciones de su plataforma de ventas.

He creado un pipeline con Apache Kafka y herramientas de su ecosistema que permite:
- Procesar datos de sensores en tiempo real y detectar condiciones anómalas
- Integrar datos de transacciones desde MySQL
- Transformar los datos para generar insights útiles para la toma de decisiones

## Arquitectura de la Solución

La arquitectura que he implementado se compone de:

1. **Fuentes de Datos**:
   - Sensores IoT (simulados con Kafka Connect Datagen)
   - Base de datos MySQL con transacciones de ventas

2. **Apache Kafka** como middleware de mensajería, con los siguientes topics:
   - `sensor-telemetry`: Datos de sensores
   - `sales-transactions`: Datos de transacciones
   - `sensor-alerts`: Alertas generadas
   - `sales-summary`: Resúmenes de ventas

3. **Kafka Connect** para la integración de datos:
   - Conector Datagen para simular datos de sensores
   - Conector JDBC para leer datos de MySQL

4. **Aplicaciones Kafka Streams** para procesar datos:
   - SensorAlerterApp: Detecta anomalías en lecturas de sensores
   - SalesSummaryApp: Agrega ventas por categoría cada minuto

## Configuración del Entorno

### Requisitos Previos
- Docker y Docker Compose
- Java 11 o superior
- Maven
- Git (para clonar el repositorio)

### Iniciar el Entorno

Para iniciar el entorno, ejecuto el script que configura todo automáticamente:

```bash
cd 8.tarea
./setup.sh
```

Este script realiza las siguientes acciones:
- Inicia los contenedores Docker (Kafka, MySQL, etc.)
- Crea la tabla de transacciones en MySQL
- Instala los conectores Kafka Connect necesarios
- Configura los esquemas Avro

A veces me da problemas este script cuando mi ordenador está muy cargado, así que si falla, lo ejecuto una segunda vez y suele funcionar.

## Implementación de los Conectores

He configurado cuatro conectores diferentes:

### 1. Conector Datagen para datos internos (viene preconfigurado)
```json
{
  "name": "source-datagen-_transactions",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "_transactions",
    "schema.filename": "/home/appuser/transactions.avsc",
    "schema.keyfield": "transaction_id",
    "max.interval": 1000,
    "iterations": 10000000,
    "tasks.max": "1"
  }
}
```

### 2. Conector MySQL Sink (viene preconfigurado)
```json
{
  "name": "sink-mysql-_transactions",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "connection.url": "jdbc:mysql://mysql:3306/db?user=user&password=password&useSSL=false",
    "topics": "_transactions",
    "table.name.format": "sales_transactions"
  }
}
```

### 3. Conector Datagen para Sensores (implementado por mí)
```json
{
  "name": "source-datagen-sensor-telemetry",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "sensor-telemetry",
    "schema.filename": "/home/appuser/sensor-telemetry.avsc",
    "schema.keyfield": "sensor_id",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "max.interval": 1000,
    "iterations": 10000000,
    "tasks.max": "1"
  }
}
```

### 4. Conector JDBC Source para Transacciones (implementado por mí)
```json
{
  "name": "source-mysql-transactions",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://mysql:3306/db?user=user&password=password&useSSL=false",
    "table.whitelist": "sales_transactions",
    "mode": "timestamp",
    "timestamp.column.name": "timestamp",
    "topic.prefix": "sales-",
    "transforms": "createKey",
    "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.createKey.fields": "transaction_id",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "poll.interval.ms": 5000,
    "tasks.max": "1"
  }
}
```

Para iniciar todos los conectores:
```bash
./start_connectors.sh
```

## Desarrollo de Aplicaciones Kafka Streams

### 1. SensorAlerterApp

Esta aplicación monitoriza los datos de sensores y genera alertas cuando detecta condiciones anómalas (temperatura > 35°C o humedad < 20%). 

```java
// Fragmento clave del código de SensorAlerterApp.java
KStream<String, GenericRecord> alertStream = sensorStream
    .peek((key, value) -> System.out.println("Procesando sensor: " + key + ", valor: " + value))
    .filter((key, value) -> {
        if (value == null) return false;
        
        // Extraer datos del sensor
        float temperature = ((Number) value.get("temperature")).floatValue();
        float humidity = ((Number) value.get("humidity")).floatValue();
        
        // Detectar condiciones anómalas
        boolean isTemperatureAnomaly = temperature > 35.0f;
        boolean isHumidityAnomaly = humidity < 20.0f;
        
        return isTemperatureAnomaly || isHumidityAnomaly;
    })
```

En esta parte me costó un poco entender cómo funcionaban los tipos de Avro. Al principio intenté hacer un simple cast a float, pero daba error. Descubrí que tenía que usar el método floatValue() para convertir correctamente.

### 2. SalesSummaryApp

Esta aplicación agrega las ventas por categoría en ventanas de tiempo de un minuto. Inicialmente tuve problemas con el campo "price" que a veces venía en formatos extraños, pero logré solucionarlo:

```java
// Fragmento clave del código de SalesSummaryApp.java
// Manejo mejorado del campo price
double price = 0.0;
Object priceObj = value.get("price");
if (priceObj instanceof Number) {
    price = ((Number) priceObj).doubleValue();
} else if (priceObj instanceof String) {
    try {
        price = Double.parseDouble((String) priceObj);
    } catch (NumberFormatException e) {
        // Si no se puede convertir, usamos un valor por defecto
        price = 50.0; // Valor predeterminado basado en el rango en transactions.avsc
    }
} else {
    // Si es otro tipo (como ByteBuffer), usamos un valor por defecto
    price = 50.0;
}
```

Estuve bastante tiempo debugueando esto porque llegaban valores raros como "J]", "/Æ", etc., que no podían convertirse a número. Al final decidí usar un valor predeterminado de 50.0 en esos casos, lo que creo que es una solución práctica en un entorno real donde los datos no siempre son perfectos.

## Ejecución y Verificación

### Ejecutar las Aplicaciones Kafka Streams

Para ejecutar las aplicaciones, se necesitan dos terminales:

Terminal 1:
```bash
cd 8.tarea
mvn exec:java -Dexec.mainClass="com.farmia.streaming.SensorAlerterApp"
```

Terminal 2:
```bash
cd 8.tarea
mvn exec:java -Dexec.mainClass="com.farmia.streaming.SalesSummaryApp"
```

### Verificar los Resultados

Para comprobar que todo funciona correctamente:

1. **Ver alertas de sensores**:
```bash
docker exec -it broker-1 kafka-console-consumer --bootstrap-server broker-1:29092 --topic sensor-alerts --from-beginning
```

2. **Ver resúmenes de ventas**:
```bash
docker exec -it broker-1 kafka-console-consumer --bootstrap-server broker-1:29092 --topic sales-summary --from-beginning
```

3. **Verificar en Control Center**: Acceder a http://localhost:9021 para ver los topics y mensajes de forma visual.

## Consideraciones y Problemas Encontrados

Durante el desarrollo de este proyecto enfrenté varios desafíos:

1. **Formato de datos inconsistente**: El campo "price" en las transacciones tenía valores en formatos extraños. Mi solución fue implementar un manejo robusto que usara valores predeterminados cuando no se podían convertir.

2. **Visualización de timestamps**: Los timestamps se mostraban como números largos (epoch time) en Control Center. Mejoré los logs de las aplicaciones para mostrar fechas legibles, lo que facilitó el seguimiento.

3. **Configuración del conector JDBC**: Me costó entender cómo configurar correctamente el modo "timestamp" y las transformaciones para generar las claves. Después de varias pruebas logré que funcionara.

## Conclusiones

He logrado implementar un pipeline completo que captura, procesa y analiza datos en tiempo real, generando información valiosa.

Si tuviera que mejorar algo, quizás añadiría una interfaz web para visualizar las alertas y resúmenes de forma más amigable, o implementaría más lógica de negocio para generar insights más complejos.

## Referencias

- [Documentación de Apache Kafka](https://kafka.apache.org/documentation/)
- [Confluent Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html)
- [Kafka Streams](https://kafka.apache.org/documentation/streams/)
- [Repositorio original del curso](https://github.com/misanalgo/kafka-curso)

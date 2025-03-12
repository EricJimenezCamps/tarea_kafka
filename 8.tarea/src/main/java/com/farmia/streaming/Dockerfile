# Usar una imagen base de OpenJDK con Java 11
FROM openjdk:11

# Establecer el directorio de trabajo dentro del contenedor
WORKDIR /app

# Copiar el código fuente al contenedor
COPY . /app

# Instalar Maven y compilar el código
RUN apt update && apt install -y maven && mvn clean package

# Ejecutar la aplicación cuando el contenedor inicie
CMD ["java", "-jar", "target/SalesSummaryApp.jar"]

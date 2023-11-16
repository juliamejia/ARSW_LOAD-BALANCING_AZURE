### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

### Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonnaci.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

### Parte 1 - Escalabilidad vertical

1. Diríjase a el [Portal de Azure](https://portal.azure.com/) y a continuación cree una maquina virtual con las características básicas descritas en la imágen 1 y que corresponden a las siguientes:
    * Resource Group = SCALABILITY_LAB
    * Virtual machine name = VERTICAL-SCALABILITY
    * Image = Ubuntu Server 
    * Size = Standard B1ls
    * Username = scalability_lab
    * SSH publi key = Su llave ssh publica

![Imágen 1](images/part1/part1-vm-basic-config.png)

2. Para conectarse a la VM use el siguiente comando, donde las `x` las debe remplazar por la IP de su propia VM (Revise la sección "Connect" de la virtual machine creada para tener una guía más detallada).

    `ssh scalability_lab@xxx.xxx.xxx.xxx`

3. Instale node, para ello siga la sección *Installing Node.js and npm using NVM* que encontrará en este [enlace](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).
4. Para instalar la aplicación adjunta al Laboratorio, suba la carpeta `FibonacciApp` a un repositorio al cual tenga acceso y ejecute estos comandos dentro de la VM:

    `git clone <your_repo>`

    `cd <your_repo>/FibonacciApp`

    `npm install`

5. Para ejecutar la aplicación puede usar el comando `npm FibinacciApp.js`, sin embargo una vez pierda la conexión ssh la aplicación dejará de funcionar. Para evitar ese compartamiento usaremos *forever*. Ejecute los siguientes comando dentro de la VM.

    ` node FibonacciApp.js`

6. Antes de verificar si el endpoint funciona, en Azure vaya a la sección de *Networking* y cree una *Inbound port rule* tal como se muestra en la imágen. Para verificar que la aplicación funciona, use un browser y user el endpoint `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`. La respuesta debe ser `The answer is 8`.

![](images/part1/part1-vm-3000InboudRule.png)

7. La función que calcula en enésimo número de la secuencia de Fibonacci está muy mal construido y consume bastante CPU para obtener la respuesta. Usando la consola del Browser documente los tiempos de respuesta para dicho endpoint usando los siguintes valores:
    * 1000000
    * 1010000
    * 1020000
    * 1030000
    * 1040000
    * 1050000
    * 1060000
    * 1070000
    * 1080000
    * 1090000    

8. Dírijase ahora a Azure y verifique el consumo de CPU para la VM. (Los resultados pueden tardar 5 minutos en aparecer).

![Imágen 2](images/part1/part1-vm-cpu.png)

9. Ahora usaremos Postman para simular una carga concurrente a nuestro sistema. Siga estos pasos.
    * Instale newman con el comando `npm install newman -g`. Para conocer más de Newman consulte el siguiente [enlace](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
    * Diríjase hasta la ruta `FibonacciApp/postman` en una maquina diferente a la VM.
    * Para el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` cambie el valor del parámetro `VM1` para que coincida con la IP de su VM.
    * Ejecute el siguiente comando.

    ```
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
    ```

10. La cantidad de CPU consumida es bastante grande y un conjunto considerable de peticiones concurrentes pueden hacer fallar nuestro servicio. Para solucionarlo usaremos una estrategia de Escalamiento Vertical. En Azure diríjase a la sección *size* y a continuación seleccione el tamaño `B2ms`.

![Imágen 3](images/part1/part1-vm-resize.png)

11. Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9.
12. Evalue el escenario de calidad asociado al requerimiento no funcional de escalabilidad y concluya si usando este modelo de escalabilidad logramos cumplirlo.
13. Vuelva a dejar la VM en el tamaño inicial para evitar cobros adicionales.

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?
2. ¿Brevemente describa para qué sirve cada recurso?
3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?
4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.
5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.
6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.
7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?
8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?
9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?
10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?
11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

### Parte 2 - Escalabilidad horizontal

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Probar el resultado final de nuestra infraestructura

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

![image](https://github.com/juliamejia/ARSW_LOAD-BALANCING_AZURE/assets/111186898/a4ea9c9a-582d-4046-8dd3-f3dda4d947e0)  
![image](https://github.com/juliamejia/ARSW_LOAD-BALANCING_AZURE/assets/111186898/b1a06530-8ac9-40c1-969f-1518b207b67a)





2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

![image](https://github.com/juliamejia/ARSW_LOAD-BALANCING_AZURE/assets/111186898/4cad68b8-c8fc-4439-8a88-1992539a3b72)  

En contraste con los resultados de la primera parte, se registró un fallo en el envío de una solicitud durante la segunda fase. La parte 2 implica un mayor costo en infraestructura debido al aumento en el número de máquinas y la inclusión de un balanceador de cargas.


3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

Dado que la suscripción de Azure para estudiantes no permitía la creación de una tercera ni cuarta máquina, se llevó a cabo este ejercicio con solo dos instancias, y este fue el resultado obtenido:  

![image](https://github.com/juliamejia/ARSW_LOAD-BALANCING_AZURE/assets/111186898/068e0a8c-16dd-4b4f-9cdd-3848ea4f3a0b)  

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?
  R: Existen tres tipos de balanceadores de carga, cada uno operando en una capa diferente de la red:

1. **Balanceador de carga de nivel de aplicación (Application Gateway):**
   - Opera en la capa 7 (capa de aplicación) del modelo OSI.
   - Enruta el tráfico según datos específicos de la aplicación, como la URL o el encabezado HTTP.
   - Ofrece funcionalidades avanzadas de seguridad, como cifrado SSL y protección contra ataques DDoS.

2. **Balanceador de carga de tráfico de red (Load Balancer):**
   - Opera en la capa 4 (capa de transporte) del modelo OSI.
   - Distribuye el tráfico entrante de manera uniforme entre los servidores de back-end.
   - Equilibra la carga en base a reglas de IP, puerto y protocolo.

3. **Gateway VPN (Virtual Private Network):**
   - Se utiliza para conectar una red virtual de Azure a una red local mediante una conexión VPN segura.
   - Enruta el tráfico de entrada y salida entre la red virtual y la red local, permitiendo la integración de servicios en la nube y locales.

En resumen, el balanceador de carga de nivel de aplicación se centra en el enrutamiento basado en datos específicos de la aplicación, el balanceador de carga de tráfico de red se especializa en distribuir uniformemente el tráfico entre los servidores de back-end, y el Gateway VPN proporciona conectividad segura entre redes virtuales de Azure y redes locales, facilitando la integración de servicios en la nube y locales.  


El SKU (Stock Keeping Unit) es una designación única asignada a un recurso que especifica sus características, capacidades y precios. Esta identificación facilita a los usuarios la selección de la opción que mejor se ajusta a sus necesidades. Los SKU varían en términos de capacidad, escalabilidad, rendimiento y disponibilidad, y están disponibles para diversos recursos en Azure, abarcando máquinas virtuales, bases de datos, almacenamiento, servicios de red y otros servicios en la nube.

1. **SKU de máquina virtual (Virtual Machine SKU):**
   - Define las características y capacidades de una instancia de máquina virtual.
   - Incluye detalles como el número de núcleos de CPU, la cantidad de RAM, la capacidad de almacenamiento y el rendimiento de red.
   - Disponible en categorías como de uso general, optimizados para cómputo, memoria y almacenamiento.

2. **SKU de base de datos (Database SKU):**
   - Define las características y capacidades de una base de datos en Azure.
   - Incluye parámetros como el tamaño de almacenamiento, el rendimiento y la disponibilidad.
   - Disponible en categorías como de uso general, optimizados para memoria y de alta disponibilidad.

3. **SKU de almacenamiento (Storage SKU):**
   - Define las características y capacidades de un recurso de almacenamiento en Azure.
   - Incluye detalles como la capacidad de almacenamiento, el rendimiento de entrada/salida y la durabilidad de los datos.
   - Disponible en categorías como de uso general, optimizados para rendimiento y para archivos.

4. **SKU de servicio de red (Network Service SKU):**
   - Define las características y capacidades de un servicio de red en Azure.
   - Incluye aspectos como la capacidad de ancho de banda, la disponibilidad y la seguridad.
   - Disponible en categorías como de uso general, optimizados para aplicaciones web y para aplicaciones empresariales.


Un balanceador de carga requiere una dirección IP pública para posibilitar la comunicación de los clientes con los recursos de Azure ubicados detrás del mismo. La dirección IP pública se asigna al balanceador de carga y se utiliza para dirigir el tráfico de entrada hacia los recursos de Azure configurados en el conjunto de escalado asociado al balanceador. La ausencia de una dirección IP pública asignada al balanceador de carga impediría la capacidad de enrutamiento del tráfico entrante, resultando en la inaccesibilidad de los recursos situados detrás del balanceador de carga.

* ¿Cuál es el propósito del *Backend Pool*?
  R: El Backend Pool cumple principalmente la función de permitir que el balanceador de carga dirija el tráfico entrante hacia los recursos adecuados que se encuentran detrás de él. Los recursos que están configurados en el conjunto de escalado, tales como máquinas virtuales o instancias de contenedor, son agregados al Backend Pool del balanceador de carga. Posteriormente, el balanceador de carga emplea algoritmos de enrutamiento de tráfico, como Round Robin o Hash de IP, para distribuir de manera equitativa el tráfico entrante hacia los recursos de destino presentes en el Backend Pool. Este proceso asegura una distribución eficiente y equilibrada de la carga entre los recursos disponibles.
  
* ¿Cuál es el propósito del *Health Probe*?
  R: El Health Probe es una herramienta fundamental en el contexto del balanceador de carga. Su función principal es supervisar y validar el estado de los recursos de destino que están configurados en el Backend Pool. El objetivo es asegurar que solo los recursos de destino que se encuentren disponibles y funcionando de manera adecuada reciban tráfico entrante del balanceador de carga. En otras palabras, el Health Probe actúa como un mecanismo de control de la salud de los recursos, garantizando que solo aquellos en un estado óptimo sean considerados para la distribución de carga, lo que mejora la confiabilidad y la eficiencia del sistema.
  
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.
  R: Load Balancing Rule (Regla de Balanceo de Carga) es una directriz que especifica cómo el balanceador de carga debe dirigir el tráfico entrante hacia los recursos de destino configurados en el Backend Pool. Estas reglas establecen los criterios y las condiciones para determinar cómo se distribuirá la carga entre los recursos disponibles. Pueden incluir información como el puerto de destino, el protocolo utilizado y el algoritmo de balanceo de carga a aplicar.

  La persistencia de sesión es fundamental para mantener el estado coherente de las interacciones de los usuarios, incluso en situaciones donde hay interrupciones o cambios en la infraestructura subyacente. Dos enfoques comunes para lograr esta persistencia son:

1. **Sesión basada en cookies:**
   - La información de sesión se almacena en una cookie enviada al cliente y recuperada en solicitudes posteriores.
   - El balanceador de carga de Azure puede configurarse para distribuir el tráfico según la cookie de sesión.
   - Asegura que las solicitudes posteriores del mismo usuario se dirijan siempre al mismo servidor, manteniendo la continuidad de la sesión.

2. **Sesión basada en IP:**
   - La información de sesión se almacena en un servidor de sesión dedicado y se asocia con la dirección IP del cliente.
   - El balanceador de carga de Azure puede configurarse para distribuir el tráfico según la dirección IP del cliente.
   - Garantiza que las solicitudes posteriores del mismo cliente se dirijan siempre al mismo servidor de sesión, proporcionando persistencia.

La persistencia de sesión es crucial en aplicaciones web o móviles que requieren autenticación del usuario o retienen información significativa del usuario, como el carrito de compras en un sitio de comercio electrónico. Al garantizar que las interacciones del usuario se mantengan con coherencia a lo largo del tiempo, se mejora la experiencia del usuario y se evitan problemas asociados con la pérdida de datos de sesión.  

* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?
  R: Una Virtual Network (VNet) es un servicio que permite la creación de una red virtual aislada en la nube. Este entorno puede ser configurado con su propio conjunto de direcciones IP, subredes, reglas de seguridad y puertas de enlace, permitiendo la conexión con otras redes, ya sea Internet o redes locales.

Una Subnet, por otro lado, es una subdivisión de una VNet que facilita la segmentación de la red en partes más pequeñas. Cada Subnet tiene su propio rango de direcciones IP dentro del espacio de direcciones IP asignado a la VNet. Las subredes pueden tener reglas de seguridad y puertas de enlace independientes, proporcionando una mayor segmentación y control de la red.

El Address Space se refiere al rango de direcciones IP privadas que se pueden utilizar dentro de una VNet. Este espacio de direcciones se configura durante la creación de la VNet y define el conjunto de direcciones IP disponibles tanto para la VNet como para sus subredes.

En cuanto al Address Range, se trata del rango específico de direcciones IP disponible para una Subnet dentro de la VNet. Al crear una Subnet, es necesario definir un Address Range que esté dentro del Address Space de la VNet. Las direcciones IP asignadas a los recursos implementados en esa Subnet provendrán de este Address Range. En conjunto, estas definiciones proporcionan una estructura organizada y controlada para la red virtual en la nube.  

* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?
  R: Una Availability Zone (Zona de Disponibilidad) en Azure es un conjunto de centros de datos interconectados en una región. Cada zona está ubicada en un lugar físico independiente y diseñada para ser autónoma, lo que significa que permanece aislada de las fallas en otras zonas de la misma región. Este diseño proporciona mayor disponibilidad y resiliencia a las aplicaciones alojadas en Azure.

La elección de tres zonas de disponibilidad diferentes se realiza con el objetivo de mejorar la disponibilidad y la resiliencia de la aplicación. Distribuyendo los recursos de la aplicación en tres zonas distintas, se garantiza que la aplicación pueda continuar operando incluso en caso de fallas en una o dos zonas de disponibilidad. Además, esta distribución permite mejorar el rendimiento al permitir que el tráfico se distribuya entre las distintas zonas de disponibilidad.

La IP zone-redundant es una dirección IP pública asignable a un recurso de Azure, como una máquina virtual o un balanceador de carga. Esta dirección IP está disponible en todas las zonas de disponibilidad de una región de Azure. En caso de una falla en una zona de disponibilidad, la dirección IP pública sigue siendo accesible desde otras zonas, asegurando así la disponibilidad continua de la aplicación en situaciones adversas.  

* ¿Cuál es el propósito del *Network Security Group*?
  R: El Network Security Group (NSG) cumple un papel fundamental al proporcionar una capa adicional de seguridad en una red virtual. Se trata de un grupo de seguridad que contiene reglas de filtrado de tráfico de red, permitiendo un control preciso sobre el tráfico que entra y sale de la red virtual en función de diversos criterios. Estas reglas del NSG pueden permitir o denegar el tráfico en base a:

1. **Dirección IP de origen y destino:**
   - Especifica qué direcciones IP pueden enviar o recibir tráfico.

2. **Puerto de origen y destino:**
   - Controla los puertos de origen y destino asociados al tráfico, regulando qué servicios pueden ser accedidos.

3. **Protocolo utilizado:**
   - Define el protocolo de red permitido (por ejemplo, TCP, UDP) para el tráfico.

4. **Otros criterios:**
   - Puede incluir restricciones adicionales basadas en otros parámetros específicos de la red.

Esta capacidad de filtrado permite a los administradores de red personalizar las políticas de seguridad según las necesidades específicas de la aplicación y garantiza que solo el tráfico autorizado pueda acceder a los recursos de la red virtual. El NSG es una herramienta esencial para fortalecer la seguridad de la infraestructura en la nube.  

* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.

![image](https://github.com/juliamejia/ARSW_LOAD-BALANCING_AZURE/assets/111186898/abc62ada-5a75-4cc7-8ef7-8ecded18c74b)

  





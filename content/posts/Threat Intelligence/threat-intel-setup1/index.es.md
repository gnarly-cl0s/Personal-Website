---
title: "Configuración de inteligencia de amenazas, parte 1 (Graylog y PFSense)"
date: 2025-02-06T08:00:00-08:00
description: Esta guía es la primera de una serie sobre cómo enviar registros desde pfSense (un firewall de código abierto) a Graylog (un analizador y agregador de registros de código abierto). Una vez que se ingieren los registros, se utilizarán los pipelines de Graylog para enriquecer los datos del firewall, que luego se visualizarán en un panel de Grafana para obtener métricas de registro útiles.
menu:
  sidebar:
    name: Intel Contra Amenazas, Parte 1
    identifier: threat-and-intel
    weight: 22
    parent: Purple-Team
tags: ["Graylog", "PFSense", "Plurilingüe"]
categories: ["Purple-Team"]
---

En el panorama de la ciberseguridad, que está en constante evolución, es fundamental saber qué sucede dentro de la red. Imagine poder obtener información en tiempo real sobre cada acción: qué está permitido, qué está bloqueado y quién intenta entrar.

Esta guía le muestra el proceso de configuración de una potente solución de inteligencia de amenazas mediante el envío de registros desde pfSense (un firewall de código abierto) a Graylog (un analizador y agregador de registros de código abierto). Al finalizar este proyecto, no solo tendrá una configuración funcional, sino también una comprensión más profunda de la postura de seguridad de su red. Piense en ella como una herramienta de detección que le ayuda a monitorear, analizar y responder a las amenazas de la red de manera más efectiva, ¡todo mientras aprende y se divierte! Sumérjase en el mundo de la inteligencia de amenazas y tome el control de la seguridad de su red hoy mismo.

Elegí PFSense para el firewall de mi red doméstica porque es de código abierto y me lo recomendaron durante mi investigación sobre firewalls. Tenía algunos servidores que pude reutilizar en el trabajo, ya que suelen desechar equipos viejos y funcionales cuando reciben equipos nuevos.  

Estas instrucciones describen un método para enviar datos desde pfSense y Suricata (probado en pfSense 2.7.2) a Graylog (probado en la versión 5.2.2). Además, se proporcionarán los pasos para procesar los registros de Suricata mediante syslog-ng.

Con un poco más de esfuerzo, confío en que el mismo enfoque que se analiza aquí se pueda aplicar a algo como OPNSense. (Estoy considerando hacer la transición a esto en algún momento para probar algo nuevo).

Me gustaría agradecer a [Jake Stride](https://jakestride.com/) por su serie PFSense y Graylog que me inspiró a configurar esto en mi laboratorio en casa y realizar actualizaciones a lo largo del camino.

# 1. Instalar Graylog
En lugar de duplicar guías de instalación bien mantenidas, debería poder seguir las [instrucciones de Graylog](https://go2docs.graylog.org/current/downloading_and_installing_graylog/installing_graylog.html) para poner en funcionamiento una instalación básica.
Recomiendo realizar una [instalación de Ubuntu](https://go2docs.graylog.org/current/downloading_and_installing_graylog/ubuntu_installation.htm) para mantenerme sincronizado con mi configuración.

Una vez instalado Graylog, continúe con el siguiente paso.

# 2. Configuración de Graylog
Cree una nueva entrada Syslog-UDP para pfSense. En la pestaña Sistema -> Entradas
Seleccione Syslog-UDP como tipo de entrada y luego inicie una nueva entrada para proporcionar detalles adicionales.

{{< img src="/posts/threat-intelligence/threat-intel-setup1/graylog1.png" height="200" width="900" align="center" title="Graylog Settings 1" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/graylog2.png" height="200" width="900" align="center" title="Graylog Settings 2" >}}

La siguiente configuración nos permite recopilar registros de PFSense, así como cualquier otro mensaje de syslog de servidores que puedan reenviar syslog a esta interfaz. Puede utilizar cualquier puerto que prefiera, siempre que no entre en conflicto con otro servicio en su instancia de Graylog o con un rango de puertos inferior a 1000. Puede configurar la dirección Bind 0.0.0.0 para escuchar en todas las interfaces posibles o darle la IP estática/DHCP de su instancia de Graylog. Aumenté los subprocesos de trabajo a 6 ya que tengo otros servidores que llegan a esta interfaz para la administración central de registros.

{{< img src="/posts/threat-intelligence/threat-intel-setup1/graylog3.png" height="600" width="500" align="center" title="Input Config 5" >}}

Asegúrese de que la casilla de verificación "¿Expandir datos estructurados?" esté habilitada/activada para ayudar con el análisis de mensajes.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/graylog4.png" height="300" width="500" align="center" title="Input Config 6" >}}

# 3. Configuración de registro de PFSense
Una vez configurado el servidor Graylog, pasaremos a la configuración de registro de PFSense para reenviar todos los posibles contenidos/eventos de syslog a la nueva interfaz que hemos creado.

Desde el firewall de pfSense, visite Estado -> Registros del sistema -> Configuración.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense1.png" height="500" width="500" align="center" title="PFSense Settings 1" >}}

Desde aquí, envíe los registros a Graylog reemplazando el campo de entrada del servidor de registro remoto con el nombre de host o la dirección IP de su servidor Graylog y agregue: 5143 como puerto, a menos que decida desviarse de las instrucciones y usar un puerto diferente. Luego, presione Guardar.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense2.png" height="500" width="500" align="center" title="PFSense Settings 2" >}}

# 4. Surricata Logging Configuration 
En mi caso, tengo Suricata ejecutándose en mi interfaz WAN con fines de aprendizaje. Sin embargo, si tuviera que implementar algo como esto en su entorno, lo ideal sería que se ejecutara y bloqueara activamente en la LAN, especialmente si busca una configuración de seguridad más estricta. Por lo general, no debería originarse tráfico malicioso desde la LAN a menos que haya patrones de tráfico inusuales, protocolos sospechosos o intentos de conexión a hosts maliciosos conocidos. Suricata proporciona a los cazadores de amenazas información valiosa sobre el origen del tráfico malicioso o el lugar al que intenta llegar si se activa un conjunto de reglas.

Al principio, pensé que la configuración de registro remoto de PFSense sería suficiente para enviar mensajes de Suricata a Graylog hasta que comencé a revisar los mensajes entrantes. Me di cuenta de que analizar los registros de Suricata requeriría un enfoque diferente, ya que después de varias ejecuciones de prueba, noté que las alertas de Suricata que esperaba no aparecían. Esto me llevó a descubrir que el demonio syslog de FreeBSD truncará automáticamente los mensajes exportados a un máximo de 480 bytes. Afortunadamente, podemos instalar syslog-ng para configurar un proceso syslog personalizado, lo que nos permite indicarle a PFSense que reenvíe los registros a nuestra instancia de Graylog para su posterior procesamiento.

Para instalar Syslog-ng vamos a: Sistema -> Paquete -> Administrador -> Paquetes disponibles y luego buscamos/descargamos syslog-ng.
Con una instalación exitosa, Syslog-ng debería estar disponible como servicio en Servicios -> Syslog-ng.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/syslog-install1.png" height="300" width="600" align="center" title="Syslog Install 1" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/syslog-install2.png" height="300" width="500" align="center" title="Syslog Install 2" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/syslog-install3.png" height="300" width="500" align="center" title="Syslog Install 3" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense6.png" height="500" width="150" align="center" title="PFSense Settings 2" >}}

Para la siguiente parte, querrás usar SSH si tienes el servicio habilitado/expuesto en tu host PFSense o usar la consola del hipervisor para acceder al host, que es el método que usaré ya que tengo PFSense alojado en mi servidor ESXi.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense7.png" height="300" width="500" align="center" title="PFSense Settings 3" >}}
Usaremos este acceso para ir a /var/log/suricata y verificar en qué directorio Suricata almacena alertas y archivos de registro. Solo analizaremos alertas de Suricata con esta configuración, pero esta configuración se puede modificar para reenviar otros registros como EVE o TLS y HTTP, ya que el script de origen a continuación usa un patrón de comodín para encontrar cualquier archivo en subdirectorios llamados alerts.log. Esto se puede cambiar a algo como eve.json, que tiene todo en un archivo, pero requiere cambios de configuración de análisis en los que no he trabajado, pero es posible que planee hacerlo en el futuro.

{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense8.png" height="200" width="800" align="center" title="PFSense Settings 4" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense10.png" height="200" width="800" align="center" title="PFSense Settings 5" >}}

Con esta información, configuraremos y habilitaremos syslog-ng para que use la ruta anterior como directorio de registro predeterminado. Tengo mi servidor Graylog en LAN, por lo que solo necesito que syslog-ng escuche y funcione dentro de la LAN.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense9.png" height="750" width="500" align="center" title="PFSense Settings 6" >}}

La configuración más básica de syslog-ng tiene 3 componentes: una fuente, un destino y un tipo de objeto de registro que le indica a syslog-ng que envíe la fuente X al destino Y. La configuración de ajustes de registro específicos se realiza en la pestaña Avanzado. Haga clic en Agregar para comenzar a escribir una configuración. Los nombres de objeto con Suricata son lo que crearemos.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/syslog-config1.png" height="300" width="550" align="center" title="Syslog config 1" >}}

La fuente wildcard-file() que aparece a continuación recopila mensajes de registro de varios archivos de texto sin formato y de varios directorios. La fuente wildcard-file() está disponible en la versión 3.10 y posteriores de OSE de syslog-ng. Establezca lo siguiente:
- Object Type: Source
- Object Name: Suricata

```json
{
  wildcard-file(
    base-dir("/var/log/suricata/suricata_em056333")
    filename-pattern("alerts.log")
    recursive(yes)
    follow-freq(1)
    flags(no-parse)
  );
};
```
<BR>
El tipo de objeto de registro conecta los distintos componentes básicos de Syslog-ng y, por lo tanto, define la ruta de los mensajes de registro entrantes. Puede contener orígenes, destinos, filtros, indicadores y otros objetos.
<BR>

Añadir otro objeto:
- Object Type: Log
- Object Name: Suricata

```json
{
    source(Suricata);
    destination(Suricata);
};
```
<BR>

Añade el objeto final
- Object Type: Destination
- Object Name: Suricata

```json
{
    udp("SERVER_IP" port(5143));
};
```
<BR>
Desde su firewall pfSense, visite Servicios -> Suricata -> Interfaces.
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense3.png" height="300" width="700" align="center" title="Suricata Settings 1" >}}
<BR>
Haga clic en el icono de lápiz resaltado arriba para configurar los ajustes de registro. Las áreas resaltadas en rojo son a las que debe prestar atención, ya que, de manera predeterminada, el registro JSON de EVE está deshabilitado y lo tengo configurado para el registro detallado del tráfico que me interesa, pero puede refinarlo como desee. Asegúrese de deshabilitar "Enviar alertas al registro del sistema" para evitar el error que cometí cuando intenté configurar esto inicialmente.

{{< img src="/posts/threat-intelligence/threat-intel-setup1/Suricata-settings1.png" height="500" width="500" align="center" title="Suricata Settings 2" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense4.png" height="500" width="500" align="center" title="Suricata Settings 3" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/pfsense5.png" height="500" width="800" align="center" title="Suricata Settings 4" >}}

# 5. Verificando la configuración
En la pestaña Sistema, navegue hasta Entradas dentro de Graylog. El rendimiento y las métricas de Syslog-UDP en el puerto 5143 deberían comenzar a aparecer. Al hacer clic en Mostrar mensajes recibidos, podrá ver las alertas del firewall de pfSense y los registros de eventos del sistema, así como los registros de Suricata. En la próxima entrega de esta serie, exploraremos cómo enrutar los mensajes entrantes en función de las fuentes de registro en transmisiones y luego mejoraremos los datos enriqueciéndolos con datos de geolocalización.

{{< img src="/posts/threat-intelligence/threat-intel-setup1/verify.png" height="250" width="850" align="center" title="Verify Logs Received 1" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup1/verify2.png" height="100" width="1500" align="center" title="Verify Logs Received 2" >}}

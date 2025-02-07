---
title: "Configuración de inteligencia de amenazas, parte 3 (Grafana)"
date: 2025-02-06T10:50:00-08:00
hero: hero.jpg
description: En esta última entrega de nuestra serie, exploraremos el panel de control de Grafana actualizado, diseñado específicamente para este proyecto. Puede descargar y personalizar el panel de control para que se ajuste a sus necesidades una vez que se hayan conectado las fuentes de datos para cada índice en su instancia de Graylog.
menu:
  sidebar:
    name: Intel Contra Amenazas, Parte 3
    identifier: threat-and-intel-part3
    weight: 23
    parent: Purple-Team
tags: ["Grafana", "Multi-lingual"]
categories: ["Purple-Team"]
---

Esta será la última sección de esta serie Threat Intel por ahora... A lo largo de este recorrido, hemos configurado varios sistemas (PFSense, Graylog y ahora Grafana) para recrear un entorno sólido similar a un centro de operaciones de seguridad (SOC) en su laboratorio doméstico mediante software de código abierto. Nuestro objetivo final es crear un ecosistema en el que pueda analizar sin problemas el tráfico de la red y visualizar los datos de los registros ingeridos mientras buscamos implementar más servicios en nuestro laboratorio.

En esta publicación, exploraremos cómo importar un panel que he actualizado con el tiempo para resumir rápidamente los eventos de firewall e IDS durante un momento determinado. Me gustaría expresar mi gratitud a [jenniferhatches](https://grafana.com/orgs/jenniferhatches) por crear inicialmente el panel en 2020 utilizando Elasticsearch como fuente de datos. Desde entonces, Elasticsearch ha experimentado un cambio de licencia, lo que llevó a AWS a bifurcar Elasticsearch y crear su versión del software, llamada OpenSearch. OpenSearch es un proyecto de código abierto impulsado por la comunidad que es totalmente compatible con Elasticsearch.

Si tiene ideas adicionales sobre cómo visualizar los registros de PFSense y/o Suricata, comente a continuación. ¡Colaboremos para mejorar nuestros conocimientos!

# Instalación de Grafana
https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

Estoy usando Grafana V10.2.3, lanzado en 2023, que es cuando comencé a explorar la inteligencia de amenazas. La última versión puede tener una interfaz de usuario diferente o configuraciones reorganizadas, pero los principios para configurar nuevas fuentes de datos y conectarse a un panel siguen siendo los mismos. Siga los pasos descritos en el enlace anterior para implementar una instancia de Grafana OSS. Elegí instalar Grafana en el mismo host que Graylog, lo que nos permite configurar una nueva conexión para un índice OpenSearch en modo de acceso al servidor, evitando así los posibles requisitos de uso compartido de recursos de origen cruzado (CORS).

Asegúrese de [iniciar el servicio](https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/#linux) después de la instalación y verificar si hay errores. 

{{< img src="/posts/threat-intelligence/threat-intel-setup3/grafana-service-status.png" height="200" width="900" align="center" title="Grafana Service Status" >}}

Una vez instalado, puede utilizar la [siguiente información](https://grafana.com/docs/grafana/latest/setup-grafana/sign-in-to-grafana/) para su inicio de sesión inicial y, como práctica recomendada, cambie la contraseña de administrador predeterminada después de iniciar sesión.

{{< img src="/posts/threat-intelligence/threat-intel-setup3/change-pw.png" height="300" width="300" align="center" title="Change PW" >}}
# Configuración de la fuente de datos
El primer paso para configurar Grafana es crear nuevas fuentes de datos, lo que le permitirá a Grafana localizar registros y realizar consultas en ellos.

{{< img src="/posts/threat-intelligence/threat-intel-setup3/new-connection1.png" height="300" width="250" align="center" title="New Connection" >}}

Al añadir una nueva conexión, busque OpenSearch, lo que requerirá la instalación del complemento. En mi caso, ya tengo el complemento instalado:

{{< img src="/posts/threat-intelligence/threat-intel-setup3/Grafana-2-Opensearch-Install.png" height="150" width="900" align="center" title="OpenSearch Plugin Install" >}}

A continuación, debemos hacer referencia a cada flujo que creamos previamente en Graylog. Al ver cualquier mensaje dentro de cada flujo, podemos guardar el valor "Almacenado en el índice", que luego se utilizará como parte de los detalles de conexión de OpenSearch al ingresar el nombre del índice. Tenga en cuenta que he incluido un asterisco (*) al final del nombre del índice para que actúe como comodín, ya que se crea un nuevo índice cada 40 días como parte del proceso de rotación de registros que establecimos anteriormente.

{{< img src="/posts/threat-intelligence/threat-intel-setup3/Grafana-3-index.png" height="450" width="900" align="center" title="Stored Index Prefix" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup3/Grafana-4-data-source.png" height="650" width="600" align="center" title="OpenSearch Data Source Config Filterlog" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup3/Grafana-4.2-data-source.png" height="200" width="800" align="center" title="OpenSearch Data Source Config Suricata" >}}

Guarde y pruebe la conexión una vez que se hayan completado los detalles de OpenSearch.
{{< img src="/posts/threat-intelligence/threat-intel-setup3/Grafana-5-test-link.png" height="250" width="900" align="center" title="Save Connection" >}}
<BR>
{{< img src="/posts/threat-intelligence/threat-intel-setup3/Grafana-4.3-data-source.png" height="350" width="800" align="center" title="Output Example" >}}

# Hora del tablero de instrumentos
Puede utilizar cualquiera de los enlaces siguientes para descargar el panel en Grafana para su uso:

1. [Grafana Site](https://grafana.com/grafana/dashboards/22780-pfsense-firewall-and-ids-dashboard-2025/)
2. [My Github Repo](https://github.com/gnarly-cl0s/Threat-Intel/blob/main/Grafana-Dashboards/PFsense%20Firewall%20and%20IDS%20Dashboard%202025-1738204003974.json)
{{< img src="/posts/threat-intelligence/threat-intel-setup3/dashboard-import1.png" height="250" width="800" align="center" title="Import Dashboard1" >}}

{{< img src="/posts/threat-intelligence/threat-intel-setup3/dashboard-import2.png" height="750" width="700" align="center" title="Import Dashboard2" >}}

Durante el proceso de importación, se le presentarán opciones como nombrar el nuevo panel, seleccionar una carpeta donde almacenarlo y asignar una identificación única si aún no es única en su configuración. Por último, para los registros de inteligencia de amenazas, señalaremos los registros de filtro de Pfsense como punto de partida.
{{< img src="/posts/threat-intelligence/threat-intel-setup3/dashboard-import3.png" height="750" width="700" align="center" title="Import Dashboard Config" >}}

Una vez que todo esté configurado e importado correctamente, el resultado final debería ser un panel completo. En el lado izquierdo, verá las métricas de las reglas de firewall de PFSense, mientras que en el lado derecho se muestran las métricas de las alertas activadas por Suricata. Los filtros se encuentran en la parte superior del panel, lo que le permite desglosar los resultados en función de las direcciones IP de origen interesantes o las interfaces configuradas en su configuración de PFSense.

{{< img src="/posts/threat-intelligence/threat-intel-setup3/hero.jpg" height="550" width="1300" align="center" title="Grafana Dashboard" >}}


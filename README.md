# Documentación revisión y proceso de duplicados - Nuevatel
Nuevatel presenta una alta cantidad de contactos creados en Marketing Automation, principalmente por los siguientes motivos:
1. No se estaba usando el formulario de "Vista Contactos MAS" de OCC para iniciar nuevas interaciones: se debe usar dicho formulario offline para buscar si el contacto al que se desea contactar ya existe en MAS, en caso exista el formulario cuenta con un botón para iniciar una nueva interacción. (Principalmente en el caso de llamadas manuales no se estaba siguiendo este proceso).
2. Unificación de contactos webchat: se identifico que las interacciones de webchat no se estaban validando correctamente si el contactos ya existia en MAS o no, por lo que en OCC se añadió a cada flujo o campaña la validación de existencia de cierto contacto (consultando directamente a la BD MAS) para saber si existia o no, si no existe se crea uno nuevo pero si existe asociará la interacción entrante con el contacto ya existente. Este proceso puede fallar si es que el servidor de MAS demora en responder la petición de "merge_interaction" lo que ocasionaria algunos contactos duplicados.
3. Criterios de duplicidad en importaciones manuales y automáticas: se identifico que algunas importaciones manuales y automáticas no tenian al "Teléfono" como criterio de duplicidad se corrigio y se indico ello al equipo de Nuevatel, solo la importación automática tiene como criterio de duplicidad al campo "id_cobranza" ya que hay veces que el teléfono viene vacio o con un código diferente que no necesariamente es un número teléfonico.

En base a los ajustes de los puntos anteriores, adicionalmente se tienen que unificar aquellos contactos que se encuentran duplicados en la plataforma (3 MM duplicados aproximadamente). Para ello se realizo un desarrollo en .NET que realiza los siguientes pasos:

1. Obtiene los contactos a unificar mediante un SP en la BD MAS.
2. Por cada registro devuelto (se devuelven 2 IDs de contactos los cuales se unificarán) se utiliza el método "merge_contacts" del API de MAS.
3. Finalmente, se guarda en un log de BD los pares de IDs unificados.

El proceso ejecuta en promedio 400 a 500 unificaciones por hora (esto depende de la carga del servidor de MAS).

El paso a paso de la implementación es la siguiente:

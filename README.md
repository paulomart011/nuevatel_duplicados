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

## 1. Creación de store procedure y tabla de logs
En una tabla de negocio de la BD MAS se deberán crear el siguiente store procedure el cual se encargará de listar las parejas a unificar:

```
USE [Nuevatel]
GO

/****** Object:  StoredProcedure [dbo].[spGetContactosPorUnificar]    Script Date: 5/6/2024 5:58:28 PM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO


CREATE PROCEDURE[dbo].[spGetContactosPorUnificar]
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	-- interfering with SELECT statements.
	SET NOCOUNT ON;
	
	WITH TempContactos AS (
		SELECT 
			C.Instance,
			C.ID,
			C.Phone
		FROM [InconcertOnline].[dbo].[Contact] AS C WITH(NOLOCK) 
		LEFT JOIN [InconcertOnline].[dbo].[EntityMetadata] AS EM WITH(NOLOCK) ON C.Instance = EM.Instance AND C.ID = EM.EntityId AND EM.EntityType = 'contact'
		WHERE 
			C.Instance = 'nuevatel' 
			--AND C.CreatedDate < DATEADD(DAY, -400, GETDATE())
			AND C.Phone IS NOT NULL
			AND C.Phone != ''
			AND C.Phone != 'anonymous'
			AND COALESCE(C.Deleted, 0) = 0
			AND EM.EntityId IS NULL
	),
	TempDuplicadosPorPhone AS (
		SELECT 
			C.Instance,
			C.ID,
			C.Phone
		FROM [InconcertOnline].[dbo].[Contact] AS C WITH(NOLOCK) 
		INNER JOIN TempContactos AS T WITH(NOLOCK) ON C.Instance = T.Instance AND C.Phone = T.Phone
		WHERE 
			C.Instance = 'nuevatel' 
			AND COALESCE(C.Deleted, 0) = 0
	)
	SELECT top 300
		MIN(CONVERT(INT, ID)) AS ContactId_Antiguo,
		MAX(CONVERT(INT, ID)) AS ContactId_Nuevo,
		COUNT(*) Repetidos

	FROM TempDuplicadosPorPhone
	GROUP BY Phone
	HAVING COUNT(*) > 1
	order by ContactId_Antiguo asc;
	   
END
GO
```
Luego se deberá crear la siguiente tabla para logs:
```
USE [Nuevatel]
GO

/****** Object:  Table [dbo].[logUnificador]    Script Date: 5/6/2024 6:01:48 PM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [dbo].[logUnificador](
	[contactoAntiguoOficial] [varchar](250) NOT NULL,
	[contactoNuevoUnificado] [varchar](250) NOT NULL,
	[resultado] [nvarchar](max) NOT NULL,
	[fecha] [datetime] NOT NULL
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
GO

ALTER TABLE [dbo].[logUnificador] ADD  CONSTRAINT [DF_logUnificador_fecha]  DEFAULT (getdate()) FOR [fecha]
GO
```
Y el siguiente SP para logs:
```
USE [Nuevatel]
GO
/****** Object:  StoredProcedure [dbo].[spLogUnificador]    Script Date: 5/6/2024 6:02:25 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[spLogUnificador]
@contactoAntiguoOficial varchar(250),
@contactoNuevoUnificado varchar(250),
@resultado nvarchar(max)

AS
BEGIN
	
	insert into  [Nuevatel].[dbo].[logUnificador] ([contactoAntiguoOficial]
      ,[contactoNuevoUnificado]
      ,[resultado])
	values( @contactoAntiguoOficial, @contactoNuevoUnificado,@resultado)
	
END
```

Luego, dentro del archivo App.config , se deben modificar los siguientes datos en base al ambiente:
```
<appSettings>
		<!-- Configuración de la BD -->
		<add key="IPServerBD" value="10.151.68.200"/> <!-- IP del servidor donde se encuentra el SP para obtener contactos a unificar -->
		<add key="BDCatalog" value="Nuevatel"/>	<!-- Nombre de la BD donde se encuentra el SP para obtener contactos a unificar -->
		<add key="mas_domain" value="https://mas-nuevatel.inconcertcc.com"/> <!-- Dominio MAS -->
		<add key="mas_endpoint" value="/api/contact/merge_contacts"/>
		<add key="mas_token" value="token"/> <!-- Token de usuario API de MAS para poder usar los métodos, se genera desde MAS en la sección Configuración>Usuario>API -->
</appSettings>
```
Las fuentes del desarrollo se encuentra en la carpeta "Desarrollo".
Finalmente, se debe realizar el build y lo que se genera en la ruta de Debug nos servirá para crear el servicio Windows, desde la consola ejecutar:
```
C:\Windows\Microsoft.NET\Framework\v4.0.30319\InstallUtil.exe "C:\Program Files (x86)\inConcert\Shared\inConcertUnificadorContactos\inConcertUnificadorContactos.exe"
```
El servicio Windows se ejcuta cada 5 minutos, en caso la ejecución demore mas de 5 minutos el servicio no se ejcutará hasta que termine dicha ejecución en curso.


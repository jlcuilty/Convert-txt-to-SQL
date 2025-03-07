# Convert-txt-to-SQL
Convertir archivo de texto delimitado por pipe en una tabla de SQL.
Este código lo hice porque tuve la necesidad de convertir archivos con demasiadas columnas (que exceden el limite de SQL) y que podemos extraer solo ciertas columnas en una tabla de SQL.
Espero les sirva. Saludos!

Importante:
Si necesitas convertir archivos de texto en tabla de SQL puedes usar estas funciones y procedimiento almacenado.
Lo hice con SQL Express 2022 en una instancia local utilizando el siguiente tutorial en : https://kb.rolosa.com/crear-una-instancia-en-sql-server/
Ahí esta la liga para descargar el SQL Excpress y el Management Studio de esa versión.
Sugiero que al instalar lo hagan en modo avanzado para que la instancia tenga inicio de sesión hibrido (por usuario local y sa).
También (en mi caso) modifique la compaginación (colation) como SQL_Latin1_General_CP1_CI_AS.
Ya creada la instancia tuve que ejectuar la siguiente instrucción para que funcionara el STRING_SPLIT: ALTER DATABASE [Nombre de la Base que creaste] SET COMPATIBILITY_LEVEL = 130

Ejemplo y consiferaciones a utilizar el procedimiento almacenado Crear_Tabla):
Parametros: 
@variables_a_buscar = '(aqui poner todas las columnas NOMBRE y TIPO que necesites separados por un espacio, si son de texto pornerle VARCHAR(MAX), si es númerico ponerle INT)'
@archivo_texto= '(ruta del archivo y el arhivo tipo texto)'
@tabla_destino= '(Tabla que se va a crear, OJO... si ya existe la borra y la crea de nuevo)'
Nota: No considere los decimales. Cualquier aportación es agradecida.

EXECUTE Crea_Tabla @variables_a_buscar='NOMBRE VARCHAR(MAX) EDAD INT', @archivo_texto='C:\Archivos\Archivo.txt', @tabla_destino='Tabla_Datos2'
Al ejecutarlo pone un mensaje de se creo la Tabla Destino que pusiste como parametro, en este caso "Tabla_Datos2"

SELECT * FROM Tabla_Datos2



Estas son las funciones (2) y Procedimiento (1) que hay que correr y se van a crear dentro de su Base de Datos :

IF EXISTS(SELECT * FROM sys.objects WHERE  object_id = OBJECT_ID(N'[dbo].[busca_valor]') AND type IN ( N'FN', N'IF', N'TF', N'FS', N'FT' )) BEGIN 
	DROP FUNCTION [dbo].[busca_valor];
END
GO
CREATE FUNCTION [dbo].[busca_valor] (@cadena VARCHAR(MAX), @donde_en_contados INT)
RETURNS VARCHAR(MAX)
BEGIN
    DECLARE @posicion INT = 1;
    DECLARE @longitud INT = LEN(@cadena);
	DECLARE @caracter VARCHAR(1)='|';
	DECLARE @cuantos_contados INT = 0;
	DECLARE @valor_encontrado VARCHAR(MAX) = '';
    WHILE @posicion <= @longitud
    BEGIN
        IF SUBSTRING(@cadena, @posicion, 1) = @caracter BEGIN
			SET @cuantos_contados = @cuantos_contados + 1;
		END;
		IF @cuantos_contados = @donde_en_contados BEGIN 
			IF SUBSTRING(@cadena, @posicion, 1) <> @caracter BEGIN 
				SET @valor_encontrado = @valor_encontrado + SUBSTRING(@cadena, @posicion, 1) 
			END
		END
		IF @cuantos_contados > @donde_en_contados BEGIN	BREAK END
        SET @posicion = @posicion + 1;
    END
	RETURN @valor_encontrado
END
GO
                

IF EXISTS(SELECT * FROM sys.objects WHERE  object_id = OBJECT_ID(N'[dbo].[busca_variable]') AND type IN ( N'FN', N'IF', N'TF', N'FS', N'FT' )) BEGIN 
	DROP FUNCTION [dbo].[busca_variable];
END
GO
CREATE FUNCTION [dbo].[busca_variable](@Encabezado VARCHAR(MAX), @variable_a_buscar VARCHAR(10))
RETURNS INT
BEGIN
	DECLARE @Largo_Cadena int
	SET @Largo_Cadena = (SELECT len(@Encabezado))
	DECLARE @cuantos_pipe int
	SET @cuantos_pipe = 0
	DECLARE @i int
	SET @i = 0
	DECLARE @pos_dela_variable_a_buscar int
	SET @pos_dela_variable_a_buscar=0
	DECLARE @campo as VARCHAR(MAX)
	SET @campo = ''
	WHILE (@i<=@Largo_Cadena AND @pos_dela_variable_a_buscar=0) BEGIN
		if ( SUBSTRING(@Encabezado, @i, 1)='|') BEGIN 
			SET @cuantos_pipe=@cuantos_pipe+1
			SET @campo = ''
		END ELSE BEGIN
			SET @campo = @campo + SUBSTRING(@Encabezado, @i, 1)
			IF (@campo=@variable_a_buscar) BEGIN SET @pos_dela_variable_a_buscar=@cuantos_pipe END
		END
		SET @i = @i + 1
	END
	RETURN @pos_dela_variable_a_buscar
END
GO


IF EXISTS (SELECT * FROM sys.procedures WHERE name = 'Crea_Tabla') BEGIN 
	DROP PROCEDURE Crea_Tabla
END
GO
CREATE PROCEDURE Crea_Tabla
    @variables_a_buscar VARCHAR(MAX),
    @archivo_texto NVARCHAR(MAX),
    @tabla_destino VARCHAR(MAX)
AS
BEGIN
	-- Crea la instrucción para modificar la tabla donde a dejar los datos para hacer la consulta
	DECLARE @cuantos INT
	DECLARE @variable_911 VARCHAR(MAX) = ''
	DECLARE @tipo_variable_911 VARCHAR(MAX) = ''
	SET @cuantos = (SELECT count(value) FROM STRING_SPLIT(@variables_a_buscar, ' '))
	DECLARE @contador INT = 1
	DECLARE @SQLString AS NVARCHAR(MAX) = 'ALTER TABLE #tabla_datos ADD '
	WHILE @contador <= @cuantos BEGIN
		SET @variable_911 = (SELECT value FROM STRING_SPLIT(@variables_a_buscar, ' ', 1) WHERE ordinal = @contador)
		SET @contador = @contador + 1
		SET @tipo_variable_911 = (SELECT value FROM STRING_SPLIT(@variables_a_buscar, ' ', 1) WHERE ordinal = @contador)
		SET @SQLString = @SQLString + @variable_911 + ' ' + @tipo_variable_911 
		SET @contador = @contador + 1
		IF (@contador+1 <= @cuantos) BEGIN SET @SQLString = @SQLString + ', ' END
	END
	-- Borrar tabla temporal donde se van a guardar los datos que se van a consultar
	IF OBJECT_ID('tempdb..#tabla_datos') IS NOT NULL DROP TABLE #tabla_datos
	-- Creamos la tabla temporal en base a la instruccion anterior
	CREATE TABLE #tabla_datos (coltemp VARCHAR(1)) -- crea tabla temporal con una columna temooral
	EXEC(@SQLString)
	ALTER TABLE #tabla_datos DROP COLUMN coltemp -- borra columna temporal para dejar solo las columnas necesarias
	--PRINT '->Crea tabla temporal #tabla_datos con los campos donde vamos a dejar los datos.'
	-- Crea la tabla temporal con todos los datos del archivo de texto
	SET NOCOUNT ON
	DROP TABLE IF EXISTS #tempTable
	CREATE TABLE #tempTable (FILA VARCHAR(MAX)) 
	DECLARE @sql NVARCHAR(MAX) = 'BULK INSERT #tempTable FROM ''' + @archivo_texto + ''' '
	EXEC(@sql);
	--PRINT '->Crea tabla temporal #tempTable con los datos del archivo de texto en un solo campo FILA.'
	DECLARE @Encabezados varchar(MAX)
	SET @Encabezados = (SELECT TOP(1) FILA FROM #tempTable)
	--PRINT '->Obtiene de la tabla temporal #tempTable el primer registro que contiene el encabezado con todos los campos.'
	/* Procedimiento para obtener la posicion de cada variable en la tabla de texto*/
	IF OBJECT_ID('tempdb..#tabla_temp_variables_a_buscar') IS NOT NULL DROP TABLE #tabla_temp_variables_a_buscar
	CREATE TABLE #tabla_temp_variables_a_buscar (variable_a_buscar VARCHAR(max), posicion INT, tipo VARCHAR(max))
	--PRINT '->Crea la tabla temporal #tabla_temp_variables_a_buscar donde se va a guardar información de las variables solicitadas.'
	DECLARE @c INT = 1
	WHILE @c <= @cuantos BEGIN
		SET @variable_911 = (SELECT value FROM STRING_SPLIT(@variables_a_buscar, ' ', 1) WHERE ordinal = @c)
		SET @c = @c + 1
		SET @tipo_variable_911 = (SELECT value FROM STRING_SPLIT(@variables_a_buscar, ' ', 1) WHERE ordinal = @c)
		INSERT #tabla_temp_variables_a_buscar (variable_a_buscar, posicion, tipo) SELECT @variable_911, dbo.busca_variable(@Encabezados, @variable_911), @tipo_variable_911
		SET @c = @c + 1
	END
	--PRINT '->Guarda en #tabla_temp_variables_a_buscar los datos de cada variable: Nombre, posiciion y tipo.'
	/************ Esta es la rutina que recorre la tabla de campos **********************/
	DECLARE @registro_911 varchar(MAX)
	DECLARE @variable_a_buscar varchar(MAX)
	DECLARE @posicion INT
	DECLARE @tipo VARCHAR(MAX)
	DECLARE @upper_tipo VARCHAR(MAX)
	DECLARE @comando_variables VARCHAR(MAX) =''
	DECLARE @comando_valores VARCHAR(MAX) =''
	DROP TABLE IF EXISTS #tabla_variable_a_buscar
	SELECT * INTO #tabla_variable_a_buscar FROM #tabla_temp_variables_a_buscar 
	DELETE FROM #tabla_variable_a_buscar
	--PRINT '->Crea la tabla temporal #tabla_variable_a_buscar que va a estar recorriendo con cada variable que va a buscar en la tabla #tempTable con los datos del archivo de texto.'
	-- Crea cursor
	INSERT INTO #tabla_variable_a_buscar SELECT * FROM #tabla_temp_variables_a_buscar 
	DECLARE @Column_variable NVARCHAR(MAX) 
	DECLARE @Column_posicion NVARCHAR(MAX) 
	DECLARE @Column_tipo NVARCHAR(MAX) 
	IF CURSOR_STATUS('global', 'cursorRecords') >= -1 BEGIN	CLOSE cursorRecords; DEALLOCATE cursorRecords; END
	DECLARE cursorRecords SCROLL CURSOR FOR SELECT variable_a_buscar, posicion, tipo FROM #tabla_temp_variables_a_buscar
	--DECLARE @contador10 INT = 0;
	-- delay
	WAITFOR DELAY '00:00:02'
	DROP TABLE IF EXISTS #MYTEMP
	SELECT NULL AS mykey, * INTO #MYTEMP FROM #tempTable WHERE FILA != @Encabezados
	--PRINT '->Crea tabla temporal "MYTEMP con los datos de la tabla temporal #tempTable que va a recorrer registro por registro sin en el Encabezado.'
	UPDATE TOP(1) #MYTEMP SET mykey = 1
	DECLARE @row_count as INT = 0
	SET @row_count = (SELECT COUNT(*) FROM #MYTEMP)
	--PRINT @row_count
	WHILE @row_count > 0
	BEGIN
		SET @comando_variables = ''
		SET @comando_valores = ''
		SET @upper_tipo =''
		SET @registro_911 = (SELECT FILA FROM #MYTEMP WHERE mykey = 1)
		--PRINT @registro_911
		OPEN cursorRecords
		FETCH NEXT FROM cursorRecords INTO @Column_variable, @Column_posicion, @Column_tipo
		WHILE @@FETCH_STATUS = 0 BEGIN
			--PRINT 'variable:' + @Column_variable +' posicion:'+ @Column_posicion +' tipo:'+ @Column_tipo + ' valor:' + dbo.busca_valor(@registro_911, @Column_posicion)
			SET @comando_variables = @comando_variables + @Column_variable + ','
			SELECT @tipo = (SELECT IIF((SELECT CHARINDEX('VARCHAR', @Column_tipo))>0, 'VARCHAR', 'INT'))
			SET @comando_valores = @comando_valores + IIF(@tipo='VARCHAR','''','')+ dbo.busca_valor(@registro_911, @Column_posicion) +IIF(@tipo='VARCHAR',''',',',')
			FETCH NEXT FROM cursorRecords INTO @Column_variable, @Column_posicion, @Column_tipo
		END
		CLOSE cursorRecords
		SET @comando_variables = SUBSTRING (@comando_variables, 1, Len(@comando_variables) - 1 )
		SET @comando_valores = SUBSTRING (@comando_valores, 1, Len(@comando_valores) - 1 )
		--PRINT 'INSERT INTO #tabla_datos ('+@comando_variables+') VALUES ('+@comando_valores+')'
		EXEC('INSERT INTO #tabla_datos ('+@comando_variables+') VALUES ('+@comando_valores+')')
		SET @comando_variables = ''
		SET @comando_valores = ''
		DELETE FROM #MYTEMP WHERE mykey = 1
		UPDATE TOP(1) #MYTEMP SET mykey = 1
		SET @row_count = (SELECT COUNT(*) FROM #MYTEMP)
		-- Incrementa el contador.
		--SET @contador10 = @contador10 + 1;
		-- Sal del bucle después de 10 iteraciones.
		--IF @contador10 >= 10  BEGIN BREAK; END
	END
	DEALLOCATE cursorRecords
	--PRINT 'INSERT INTO #tabla_datos ('+@comando_variables+') VALUES ('+@comando_valores+')'
	--SELECT * FROM #tabla_datos
	DECLARE @sql_tabla_destino NVARCHAR(MAX);
	IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = @tabla_destino) BEGIN
		-- Si la tabla existe, se elimina
		SET @sql_tabla_destino = 'DROP TABLE ' + QUOTENAME(@tabla_destino);
		EXEC sp_executesql @sql_tabla_destino;
	END
	SET @sql_tabla_destino = 'SELECT * INTO ' + QUOTENAME(@tabla_destino) + ' FROM #tabla_datos';
	-- Ejecutar la consulta dinámica
	EXEC sp_executesql @sql_tabla_destino;
	IF OBJECT_ID('tempdb..#tabla_datos') IS NOT NULL DROP TABLE #tabla_datos
	IF OBJECT_ID('tempdb..#tabla_temp_variables_a_buscar') IS NOT NULL DROP TABLE #tabla_temp_variables_a_buscar
	IF OBJECT_ID('tempdb..#MYTEMP') IS NOT NULL DROP TABLE #MYTEMP
	IF CURSOR_STATUS('global', 'cursorRecords') >= -1 BEGIN	CLOSE cursorRecords; DEALLOCATE cursorRecords; END
	PRINT '->Tabla '+@tabla_destino+' creada'
END
GO

---
title: Migraciones ERP
author: Andres Rocabado
date: 2024-08-02
category: LKS
layout: post
mermaid: true
---

## Tratamiento de datos

El tratamiento de datos se realizó mediante la herramienta de [Google Colaboratory](https://colab.research.google.com/).

## Productos, categorías, embalage, fabricante, stocks

### Documentos usados

Los documentos usados para realizar estas migraciones fueron:

- [Excel categorías productos](https://geospatialtechnology.github.io/assets/documents/PROD_CATEGORIAS.xlsx)
- [Excel embalages productos](https://geospatialtechnology.github.io/assets/documents/PROD_EMBALAGE.xlsx)
- [Excel fabricantes productos](https://geospatialtechnology.github.io/assets/documents//PROD_FABRICANTE.xlsx)
- [Excel stocks productos](https://geospatialtechnology.github.io/assets/documents//PROD_STOCKS.xlsx)
- [Excel productos](https://geospatialtechnology.github.io/assets/documents/PRODUCTOS.xlsx)

### Cargado de dataframes

Los documentos se subieron a una carpeta en google drive de modo que se pueda acceder a ellos mediante google colab, para el manejo de datos se usó pandas, esto se realizó de la siguiente manera:

```markdown
import pandas as pd
from google.colab import drive
drive.mount('/content/drive')

categorias = pd.read_excel('/content/drive/MyDrive/PathDocumento/PROD_CATEGORIAS.xlsx')
embalages = pd.read_excel('/content/drive/MyDrive/PathDocumento/PROD_EMBALAGE.xlsx')
fabricantes = pd.read_excel('/content/drive/MyDrive/PathDocumento/PROD_FABRICANTE.xlsx')
stocks = pd.read_excel('/content/drive/MyDrive/PathDocumento/PROD_STOCKS.xlsx')
productos = pd.read_excel('/content/drive/MyDrive/PathDocumento/PRODUCTOS.xlsx')
```

### Dar formato a columnas

En siguiente código es para dar el formato correcto a columnas de productos que toman "." por ",":

```markdown
# Función para reemplazar puntos por comas

def replace_dot_with_comma(x):
if pd.isna(x):
return x
elif isinstance(x, float):
return str(x).replace('.', ',')
elif isinstance(x, str):
return x
else:
return x

productos['COD_INTERNO'] = productos['COD_INTERNO'].apply(replace_dot_with_comma)
productos['COD_BARRAS'] = productos['COD_BARRAS'].apply(replace_dot_with_comma)
productos['COD_FABRICANTE'] = productos['COD_FABRICANTE'].apply(replace_dot_with_comma)
productos['PESO'] = productos['PESO'].apply(replace_dot_with_comma)
productos['VLR_VENDA'] = productos['VLR_VENDA'].apply(replace_dot_with_comma)
productos['VLR_VENDA2'] = productos['VLR_VENDA2'].apply(replace_dot_with_comma)
productos['VLR_VENDA3'] = productos['VLR_VENDA3'].apply(replace_dot_with_comma)
```

### Agregado de nuevas columnas

Se agregaron nuevas columnas en productos de modo que los "Id" antíguos de las categorías y fabricantes conincidan con los nuevos "Id" que se crearán en el ERP, además se agregó una columna con el embalage que corresponda a cada producto y posteriormente se exportó en nuevo documento a un CSV, esto se realizó de la siguiente manera:

```markdown
# Crear un diccionario de código de fabricante y su índice de fila en el archivo de fabricantes

fabricantes_dict = fabricantes.reset_index().set_index('COD_FABRICANTE')['index'].add(1).to_dict()

# Crear una nueva columna en el dataframe de productos

productos['INDICE_FABRICANTE'] = productos['FABRICANTE'].map(lambda x: fabricantes_dict.get(x, None)).astype(pd.Int64Dtype())

# Crear un diccionario de código de categoria y su índice de fila en el archivo de categorias

categorias_dict = categorias.reset_index().set_index('CODIGO')['index'].add(1).to_dict()

# Crear una nueva columna en el dataframe de productos

productos['INDICE_CATEGORIA'] = productos['GRUPO'].map(lambda x: categorias_dict.get(x, None)).astype(pd.Int64Dtype())

# Crear un diccionario de código de embalage y su descripcion

embalages_dict = embalages.set_index('Cod_Embalagem')['Descricao_Embalagem'].to_dict()

# Crear nueva columna en el dataframe de productos

productos['EMBALAGE_NOMBRE'] = productos['EMBALAGEM'].map(lambda x: embalages_dict.get(x,None))

# Exportado de nuevo CSV

productos.to_csv('/content/drive/MyDrive/"PathExportacion".csv', index=False)
```

### Productos que cuenta con stocks

Se tomó en cuenta los stocks que si tengan productos válidos, los productos válidos son aquellos que cuentan con un código interno, una vez obtenido los productos válidos, se tomo en cuenta los stocks de esos productos, esto se realizó de la siguiente manera:

```markdown
# Productos válidos

producto_con_codigo = productos[~productos['COD_INTERNO'].isna() & (productos['COD_INTERNO'].str.strip() != "")]

# Stocks que cuentan con productos

filtro = stocks['MATERIAL_ID'].isin(producto_con_codigo['CODID'])
stocks_con_productos = stocks[filtro]
```

### Agregado de columnas a stocks

Se agregaron nuevas columnas en stocks de modo que se identifique a que almacen del ERP pertence cada "ARMAZEN" del ambar, de igual manera se creó una columna que contenge el "Id" que tiene el producto en el ERP, esto se realizó de la siguiente manera:

```markdown
# Crear un diccionario de mapeo para ARMAZEN

mapeo_armazen = {
1: 'LKS',
2: 'LKS2',
3: 'VACIO'
}

# Crear una nueva columna con los valores mapeados según los almacenes del ERP

stocks_con_productos['ALMACEN'] = stocks_con_productos['ARMAZEM'].map(mapeo_armazen)

# Crear un diccionario de índice de productos válidos y su índice + 1

productos_validos_dict = {i: i + 1 for i in range(len(producto_con_codigo))}

# Crear una columna auxiliar que identifique cambios en MATERIAL_ID

stocks_con_productos['Grupo'] = (stocks_con_productos['MATERIAL_ID'] != stocks_con_productos['MATERIAL_ID'].shift()).cumsum() - 1

# Mapear los valores del diccionario productos_validos_dict a la nueva columna

stocks_con_productos['ID_PRODUCTO'] = stocks_con_productos['Grupo'].map(productos_validos_dict).astype(pd.Int64Dtype())

# Eliminar la columna auxiliar 'Grupo' si no la necesitas

stocks_con_productos = stocks_con_productos.drop(columns=['Grupo'])
```

### Dar formato a columnas de stocks

En siguiente código es para dar el formato correcto a columnas de stocks que tienen en cantidad como valores decimales y convertirlos a valores enteros, posteriormente se exportó en nuevo documento a un CSV, esto se realizó de la siguiente manera:

```markdown
# Dar formato a columna

stocks_con_productos['ESTOQUE'] = pd.to_numeric(stocks_con_productos['ESTOQUE'], errors='coerce').fillna(0).astype(int)

# Exportar a CSV

stocks_con_productos.to_csv('/content/drive/MyDrive/"PathExportacion".csv', index=False)
```

## Proveedores

### Documentos usados

Los documentos usados para realizar estas migraciones fueron:

- [Excel contactos](https://geospatialtechnology.github.io/assets/documents/CONTACTOS.xlsx)

### Cargado de dataframes

Los documentos se subieron a una carpeta en google drive de modo que se pueda acceder a ellos mediante google colab, para el manejo de datos se usó pandas, esto se realizó de la siguiente manera:

```markdown
import pandas as pd
from google.colab import drive
drive.mount('/content/drive')

contactos = pd.read_excel('/content/drive/MyDrive/PathDocumento/CONTACTOS.xlsx', sheet_name='proveedor')
```

### Dar formato a columnas

En siguiente código es para dar el formato correcto a columnas que tengan fechas y posteriormente exportar el nuevo excel:

```markdown
# Formato correcto fechas

proveedores['Data_Cadastro'] = pd.to_datetime(proveedores['Data_Cadastro'], errors='coerce')
proveedores['Data_Cadastro'] = proveedores['Data_Cadastro'].dt.strftime('%Y-%m-%d')

# Guardar excel de proveedores

proveedores.to_excel('/content/drive/MyDrive/"PathExportacion".xlsx', index=False)
```

## Clientes

### Documentos usados

Los documentos usados para realizar estas migraciones fueron:

- [Excel contactos](https://geospatialtechnology.github.io/assets/documents/CONTACTOS.xlsx)

### Cargado de dataframes

Los documentos se subieron a una carpeta en google drive de modo que se pueda acceder a ellos mediante google colab, para el manejo de datos se usó pandas, esto se realizó de la siguiente manera:

```markdown
import pandas as pd
from google.colab import drive
drive.mount('/content/drive')

contactos = pd.read_excel('/content/drive/MyDrive/PathDocumento/CONTACTOS.xlsx', sheet_name='Cliente')
```

### Dar formato a columnas

En siguiente código es para dar el formato correcto a columnas que tengan fechas, además de quitar formato RTF y posteriormente exportar el nuevo CSV:

```markdown
import numpy as np
!pip install striprtf
from striprtf.striprtf import rtf_to_text

# Formato correcto fechas

clientes['DATA'] = pd.to_datetime(clientes['DATA'], errors='coerce')
clientes['DATA'] = clientes['DATA'].dt.strftime('%Y-%m-%d')

# Función segura para convertir RTF a texto y manejar valores NaN o vacíos

def safe_rtf_to_text(rtf_text):
if pd.isna(rtf_text):
return None
elif isinstance(rtf_text, str):
try:
text = rtf_to_text(rtf_text)
if text.strip() == "": # Si el texto convertido es una cadena vacía, retorna None
return None
return text
except Exception as e: # Si la conversión falla, retornar el texto original
return rtf_text
else:
return rtf_text

clientes['OBS'] = clientes['OBS'].apply(safe_rtf_to_text)

# Guardar excel de clientes

clientes.to_csv('/content/drive/MyDrive/GeoTec/"PathExportacion".csv', index=False)
```

## Transferencias

### Documentos usados

Los documentos usados para realizar estas migraciones fueron:

- [Excel transferencias](https://geospatialtechnology.github.io/assets/documents/VENTAS.xlsx)
- [Excel transferencias detalles](https://geospatialtechnology.github.io/assets/documents/VENTAS_DETALLES.xlsx)

### Cargado de dataframes

Los documentos se subieron a una carpeta en google drive de modo que se pueda acceder a ellos mediante google colab, para el manejo de datos se usó pandas, los documentos que se usaron para las transferencias y las ventas son los mismos, ya que el sistema Ambar no contaba con un módulo para hacer transferencias entre almacenes, es por ello que las transferencias se realizaban como ventas entre almacenes, esto se realizó de la siguiente manera:

```markdown
import pandas as pd
from google.colab import drive
drive.mount('/content/drive')

ventas = pd.read_excel('/content/drive/MyDrive/PathDocumento/VENTAS.xlsx')
ventas_detalles = pd.read_excel('/content/drive/MyDrive/PathDocumento/VENTAS_DETALLES.xlsx')
```

### Obtener productos y clientes válidos

Se guardó en variables tanto los productos como los clientes que se tomaron en cuenta en el ERP, esto se realizó de la siguiente manera:

```markdown
# Productos válidos

producto_con_codigo = productos[~productos['COD_INTERNO'].isna() & (productos['COD_INTERNO'].str.strip() != "")]

# Clientes válidos

clientes = pd.read_excel('/content/drive/MyDrive/PathDocumento/CONTACTOS.xlsx', sheet_name='Cliente')
```

### Dar formato a columnas

En siguiente código es para dar el formato correcto a columnas para las transferencias, esto se hizo de la siguiente manera:

```markdown
# Dar formato numerico al Id del producto

ventas_detalles['PRODUCT_ID'] = pd.to_numeric(ventas_detalles['PRODUCT_ID'], errors='coerce').astype(pd.Int64Dtype())

# Función para reemplazar puntos por comas

def replace_dot_with_comma(x):
if pd.isna(x):
return x
elif isinstance(x, float):
return str(x).replace('.', ',')
elif isinstance(x, str):
return x
else:
return x

# Reemplazar "." por "," en columnas

ventas_detalles['VLR_UNIT'] = ventas_detalles['VLR_UNIT'].apply(replace_dot_with_comma)
ventas_detalles['VLR_TOTAL'] = ventas_detalles['VLR_TOTAL'].apply(replace_dot_with_comma)
ventas['TOTAL_PAGO'] = ventas['TOTAL_PAGO'].apply(replace_dot_with_comma)
ventas['TOTAL_PEDIDO'] = ventas['TOTAL_PEDIDO'].apply(replace_dot_with_comma)
ventas['TOTAL_PRODUTOS'] = ventas['TOTAL_PRODUTOS'].apply(replace_dot_with_comma)

# Convertimos a numérico, asegurándonos de que no haya errores

ventas_detalles['QUANT'] = pd.to_numeric(ventas_detalles['QUANT'], errors='coerce')

# Tomamos solo la parte entera de los valores, incluso si son floats

ventas_detalles['QUANT'] = ventas_detalles['QUANT'].fillna(0).astype(int)

# Reemplazar secuencias de caracteres especiales con cadenas vacías y eliminar espacios en blanco

ventas['OBS'] = ventas['OBS'].str.replace('_x000d_', '', regex=False).str.replace('\n', '', regex=False).str.strip()
```

### Análisis de datos

En siguiente código es para filtrar los datos que se tomaran en cuenta en el ERP, esto se hizo de la siguiente manera:

```markdown
# Transferencias que tienen clientes

filtro = ventas['COD_CLIENTE'].isin(clientes['COD_CLIENTE'])
ventas_con_clientes = ventas[filtro]

# Ventas que tienen cliente, son sin factura, no estan canceladas y son de transferencia

ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia = ventas_con_clientes[
(ventas_con_clientes['TVENDA'] != '1') &
(~ventas_con_clientes['TVENDA'].isna()) &
(ventas_con_clientes['POSICAO'] != 'CANCELADO') &
(ventas_con_clientes['NOME'].isin(['L.K.S. 1 COMPRAS 1', 'L.K.S 2 COMPRAS 2', 'DEPOSITO 5º ANILLO']))
]

# Detalles de ventas que tienen venta y pertenecen a la misma empresa

detalles_ventas_con_venta_con_empresa = ventas_detalles[
ventas_detalles.set_index(['PEDIDO','EMPRESA']).index.isin(ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia.set_index(['PEDIDO','EMPRESA']).index)
]

# Detalles de ventas que tienen venta, tienen empresa y tienen productos

filtro = detalles_ventas_con_venta_con_empresa['PRODUCT_ID'].isin(producto_con_codigo['CODID'])
detalles_ventas_con_venta_con_empresa_con_productos = detalles_ventas_con_venta_con_empresa[filtro]

# Ventas que tienen clientes, son sin factura, no son canceladas, son de transferencia y tienen detalle de venta

filtro = ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia.set_index(['PEDIDO','EMPRESA']).index.isin(detalles_ventas_con_venta_con_empresa_con_productos.set_index(['PEDIDO','EMPRESA']).index)
ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia_con_detalle = ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia[filtro]

# Crear columna en ventas con la empresa origen transferencia y la empresa destino de la transferencia

ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia_con_detalle['EMPRESA_DESTINO'] = ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia_con_detalle['NOME'].map(lambda x: 1 if x == "L.K.S. 1 COMPRAS 1" else 2).astype(pd.Int64Dtype())

# Transferencias entre diferentes sucursales y columna con el indice de la transferencia

ventas_validas = ventas_con_clientes_sin_factura_sin_cancelados_con_transferencia_con_detalle.copy()

# Para que sea transferencia entre sucursales distintas

ventas_validas = ventas_validas[
~(
((ventas_validas['RAZAO_SOCIAL'] == "L.K.S. AUTO PIEZAS 1") &
(ventas_validas['NOME'] == "L.K.S. 1 COMPRAS 1")) |
((ventas_validas['RAZAO_SOCIAL'] == "L.K.S AUTO PIEZAS 2") &
(ventas_validas['NOME'] == "L.K.S 2 COMPRAS 2")) |
((ventas_validas['RAZAO_SOCIAL'] == "DEPOSITO L.K.S") &
(ventas_validas['NOME'] == "DEPOSITO 5º ANILLO"))
)
]

# Quitamos transferencia que sean entre DEPOSITO L.K.S y L.K.S 2 COMPRAS 2 (ya que son la misma sucursal)

ventas_validas = ventas_validas[
~(
((ventas_validas['RAZAO_SOCIAL'] == "DEPOSITO L.K.S") &
(ventas_validas['NOME'] == "L.K.S 2 COMPRAS 2")) |
((ventas_validas['RAZAO_SOCIAL'] == "L.K.S 2 COMPRAS 2") &
(ventas_validas['NOME'] == "DEPOSITO L.K.S"))
)
]

# Crear una nueva columna en 'ventas' con el índice +1 usando len

ventas_validas['NUEVO_INDICE'] = range(len(ventas_validas), 0, -1)

# Detalles de venta que tienen venta y productos

filtro = detalles_ventas_con_venta_con_empresa_con_productos.set_index(['PEDIDO','EMPRESA']).index.isin(ventas_validas.set_index(['PEDIDO','EMPRESA']).index)
detalles_ventas_validas = detalles_ventas_con_venta_con_empresa_con_productos[filtro]

# Dar formato a fecha de excel VENTAS

def convert_dates(date_str): # Convierte la cadena de fecha usando el primer formato
dates_format1 = pd.to_datetime(date_str, errors='coerce')

    # Convierte la cadena de fecha usando el segundo formato
    dates_format2 = pd.to_datetime(date_str, format='%Y-%m-%d %H:%M:%S.%f', errors='coerce')

    # Determina cuál de los dos formatos no resultó en NaT
    if pd.isna(dates_format1) and not pd.isna(dates_format2):
        return dates_format2
    elif not pd.isna(dates_format1):
        return dates_format1
    else:
        return pd.NaT

# Uso de la función

ventas_validas['DATA'] = ventas_validas['DATA'].apply(convert_dates)

# Dar formato a fecha entrega de excel VENTAS

ventas_validas['DT_ENTREGA'] = pd.to_datetime(ventas_validas['DT_ENTREGA'], errors='coerce')

# Exportar csv para TRANSFERENCIAS

ventas_validas.to_csv('/content/drive/MyDrive/PathDocumento', index=False)

# Crear columna en VENTAS_DETALLES con los indices de la ventas que se encuentran en el excel de ventas_validas

# Crear un diccionario de 'PEDIDO','EMPRESA' a 'NUEVO_INDICE' desde 'ventas_validas'

pedido_a_indice = ventas_validas.set_index(['PEDIDO','EMPRESA'])['NUEVO_INDICE'].to_dict()

# Asignar 'NUEVO_INDICE' a 'detalles_ventas_validas' usando 'PEDIDO','EMPRESA' como clave

detalles_ventas_validas['ID_TRANSFERENCIA'] = detalles_ventas_validas.set_index(['PEDIDO','EMPRESA']).index.map(pedido_a_indice)

# Crear columna en VENTAS_DETALLES con los productos válidos

# Crear un diccionario de código de producto y su índice de fila en el archivo de productos

productos_dict = {producto_con_codigo['CODID'].iloc[i]: i + 1 for i in range(len(producto_con_codigo))}

# Crear una nueva columna en el dataframe de ventas

detalles_ventas_validas['ID_PRODUCT_ERP'] = detalles_ventas_validas['PRODUCT_ID'].map(lambda x: productos_dict.get(x, None)).astype(pd.Int64Dtype())

# Guardar el dataframe de venta detalles con la nueva columna

detalles_ventas_validas.to_csv('/content/drive/MyDrive/PathDocumento', index=False)
```

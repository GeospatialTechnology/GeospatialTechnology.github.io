---
title: Migraciones ERP
author: Andres Rocabado
date: 2024-08-02
category: LKS
layout: post
---

Productos, fabricantes y categorías

# Documentos usados

Los documentos usados para realizar estas migraciones fueron:

- [Excel fabricantes](../assets/documents/PROD_FABRICANTE.xlsx)
- [Excel categorías](../assets//documents/PROD_CATEGORIAS.xlsx)
- [Excel productos](../assets/documents/PRODUCTOS.xlsx)


# Tratamiento de datos
El tratamiento de datos se realizó mediante la herramienta de [Google Colaboratory](https://colab.research.google.com/), a continuación se detallaran los pasos que se realizaron para ello.

# Cargado de dataframes
Los documentos se subieron a una carpeta en google drive de modo que se pueda acceder a ellos mediante google colab, para el manejo de datos se usó pandas, esto se realizó de la siguiente manera:
```markdown
import pandas as pd
from google.colab import drive
drive.mount('/content/drive')

categorias = pd.read_excel('/content/drive/MyDrive/PathDocumento')
fabricantes = pd.read_excel('/content/drive/MyDrive/PathDocumento')
productos = pd.read_excel('/content/drive/MyDrive/PathDocumento')

```

# Dar formato a columnas
En siguiente código es para dar el formato correcto a columnas que toman "." por ",":
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
# Agregado de nuevas columnas
Se agregaron nuevas columnas de modo que los "Id" antíguos de las categorías y fabricantes conincidan con los nuevos "Id" que se crearán en el ERP y posteriormente se exportó en nuevo documento a un CSV, esto se realizó de la siguiente manera:

```markdown
# Crear un diccionario de código de fabricante y su índice de fila en el archivo de fabricantes
fabricantes_dict = fabricantes.reset_index().set_index('COD_FABRICANTE')['index'].add(1).to_dict()

# # Crear una nueva columna en el dataframe de productos
productos['INDICE_FABRICANTE'] = productos['FABRICANTE'].map(lambda x: fabricantes_dict.get(x, None)).astype(pd.Int64Dtype())

# Crear un diccionario de código de categoria y su índice de fila en el archivo de categorias
categorias_dict = categorias.reset_index().set_index('CODIGO')['index'].add(1).to_dict()

# Crear una nueva columna en el dataframe de productos
productos['INDICE_CATEGORIA'] = productos['GRUPO'].map(lambda x: categorias_dict.get(x, None)).astype(pd.Int64Dtype())

# Exportado de nuevo CSV
productos.to_csv('/content/drive/MyDrive/"PathExportacion".csv', index=False)

```

Proveedores

migraciones para proveedores

Clientes

migraciones para clientes

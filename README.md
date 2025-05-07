# 🚀 Descomprimir ZIPs Automáticamente con GitHub Actions

Este repositorio utiliza **GitHub Actions** para descomprimir automáticamente todos los archivos `.zip` que se suban o modifiquen en cualquier carpeta del repositorio.

## 📋 Índice
- [¿Qué hace esta acción?](#-qué-hace-esta-acción)
- [Cómo habilitar GitHub Actions](#️-cómo-habilitar-github-actions)
- [Agregar este flujo de trabajo](#-agregar-este-flujo-de-trabajo)
- [Cómo probarlo](#-cómo-probarlo)
- [Configuración personalizada](#-configuración-personalizada)
- [Solución de problemas](#-solución-de-problemas)
- [Contribuciones](#-contribuciones)

## ✅ ¿Qué hace esta acción?

Cada vez que haces un push con uno o más archivos `.zip`, el flujo de trabajo:

1. Detecta automáticamente cuáles archivos `.zip` son nuevos o modificados
2. Los descomprime en una carpeta con el mismo nombre (sin la extensión `.zip`)
3. Opcionalmente elimina el archivo `.zip` original (desactivado por defecto)
4. Hace commit y push de los archivos descomprimidos automáticamente

## ⚙️ Cómo habilitar GitHub Actions

1. Ve a tu repositorio en GitHub
2. Haz clic en la pestaña **"Actions"**
3. Si es tu primera acción:
   - Haz clic en el botón **"Set up a workflow yourself"**
4. Si ya tienes otras acciones:
   - Haz clic en el botón **"New workflow"**
   - Selecciona **"set up a workflow yourself"**

## 📂 Agregar este flujo de trabajo

1. Crea un nuevo archivo en la ruta `.github/workflows/auto-unzip.yml`
2. Copia y pega el siguiente código:

```yaml
name: Descomprimir Todos los ZIPs Subidos

on:
  push:
    paths:
      - '**.zip' # Se activa si cualquier archivo .zip se añade o modifica en cualquier ubicación

jobs:
  unzip_all_zips:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Necesario para hacer push de los cambios de vuelta al repositorio
    steps:
      - name: Checkout del repositorio
        uses: actions/checkout@v4
        with:
          # Necesitamos el historial completo para comparar los commits del push
          fetch-depth: 0

      - name: Encontrar y Descomprimir Archivos ZIP
        id: unzip_step
        run: |
          echo "Buscando archivos .zip nuevos o modificados..."
          # Identificar los archivos .zip que fueron añadidos o modificados en este push
          
          if [[ "${{ github.event.before }}" == "0000000000000000000000000000000000000000" ]]; then
            # Es una nueva rama o el primer push. Considera todos los .zip en el commit actual.
            echo "Nueva rama o primer push. Obteniendo todos los .zip del commit actual (${{ github.sha }})."
            zip_files_to_process=$(git ls-tree --full-tree -r --name-only ${{ github.sha }} | grep '\.zip$' || true)
          else
            # Es un push a una rama existente. Compara el estado antes y después del push.
            echo "Push a rama existente. Comparando commits ${{ github.event.before }} y ${{ github.sha }}."
            zip_files_to_process=$(git diff --name-only ${{ github.event.before }} ${{ github.sha }} | grep '\.zip$' || true)
          fi

          if [ -z "$zip_files_to_process" ]; then
            echo "No se encontraron archivos .zip nuevos o modificados para procesar en este push."
            echo "unzipped_something=false" >> $GITHUB_OUTPUT
            exit 0 # Salir si no hay zips que procesar
          fi

          echo "Archivos .zip a procesar:"
          echo "$zip_files_to_process" # Muestra la lista de archivos ZIP

          unzipped_files_list_for_commit=() # Para el mensaje de commit

          # Itera sobre cada archivo ZIP encontrado
          for zip_file in $zip_files_to_process; do
            # Verifica que el archivo realmente exista en el checkout (por si acaso)
            if [ -f "$zip_file" ]; then
              echo "Procesando archivo ZIP: $zip_file"
              
              # Determina el directorio padre y el nombre base del archivo ZIP (sin extensión)
              parent_dir=$(dirname "$zip_file")
              base_name_no_ext=$(basename "$zip_file" .zip)
              
              # Crea un directorio para la extracción con el nombre del ZIP (sin .zip)
              extraction_dir="$parent_dir/$base_name_no_ext"
              
              # Asegura que el directorio de extracción no sea solo "." si el zip está en la raíz
              if [[ "$parent_dir" == "." ]]; then
                extraction_dir="$base_name_no_ext"
              fi
              
              echo "Creando directorio de extracción: $extraction_dir"
              mkdir -p "$extraction_dir"
              
              echo "Descomprimiendo '$zip_file' en '$extraction_dir/'"
              # -q para modo silencioso, -o para sobrescribir archivos existentes sin preguntar
              unzip -q -o "$zip_file" -d "$extraction_dir"
              
              # Opcional: Eliminar el archivo ZIP original después de la descompresión
              # echo "Eliminando archivo ZIP original: $zip_file"
              # rm "$zip_file"
              
              unzipped_files_list_for_commit+=("$zip_file") # Añade a la lista para el mensaje de commit
            else
              echo "Advertencia: El archivo '$zip_file' listado en el diff no se encontró. Omitiendo."
            fi
          done

          if [ ${#unzipped_files_list_for_commit[@]} -gt 0 ]; then
            echo "unzipped_something=true" >> $GITHUB_OUTPUT
            # Convierte el array de archivos procesados a una cadena separada por comas
            processed_files_str=$(IFS=, ; echo "${unzipped_files_list_for_commit[*]}")
            echo "processed_zips=$processed_files_str" >> $GITHUB_OUTPUT
            echo "Archivos ZIP procesados: $processed_files_str"
          else
            echo "No se descomprimió ningún archivo ZIP nuevo o modificado."
            echo "unzipped_something=false" >> $GITHUB_OUTPUT
          fi

      - name: Configurar Git para el Commit
        # Solo se ejecuta si se descomprimió algo
        if: steps.unzip_step.outputs.unzipped_something == 'true'
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'

      - name: Hacer Commit de los Archivos Descomprimidos
        # Solo se ejecuta si se descomprimió algo
        if: steps.unzip_step.outputs.unzipped_something == 'true'
        run: |
          # Añade todos los cambios (archivos descomprimidos y ZIPs eliminados si se usa 'rm')
          git add .
          
          commit_message="Autounzip: se procesaron los archivos ZIP: ${{ steps.unzip_step.outputs.processed_zips }}"
          echo "Preparando commit con mensaje: $commit_message"
          
          # Verifica si hay cambios para hacer commit para evitar commits vacíos
          if git diff --staged --quiet; then
            echo "No hay cambios para hacer commit (posiblemente los archivos ya estaban extraídos y no hubo cambios)."
          else
            git commit -m "$commit_message"
            echo "Haciendo push de los cambios..."
            git push
          fi
```

3. Haz clic en **"Commit changes"** para guardar el archivo

## 🧪 Cómo probarlo

1. Sube un archivo `.zip` a cualquier carpeta del repositorio
2. Espera a que se ejecute el workflow (puedes seguir su progreso en la pestaña "Actions")
3. Verifica que se haya creado automáticamente:
   - Una nueva carpeta con el mismo nombre que el archivo ZIP (sin la extensión)
   - Un commit automático con los archivos descomprimidos

## ⚡ Configuración personalizada

Para personalizar el comportamiento de la acción:

- **Eliminar archivos ZIP originales**: Descomenta la línea `# rm "$zip_file"` en el script para eliminar los ZIP después de descomprimirlos
- **Cambiar el mensaje de commit**: Modifica la variable `commit_message` para personalizar el texto del commit automático
- **Aplicar a ciertos directorios**: Cambia el trigger `paths` si solo quieres activar la acción en ciertas carpetas

## 🔍 Solución de problemas

- **La acción no se ejecuta**: Verifica que tengas habilitadas las GitHub Actions en tu repositorio
- **Errores de permisos**: Asegúrate de que la línea `permissions: contents: write` esté presente
- **No se hace push**: Revisa los logs de la acción en la pestaña "Actions" para ver si hubo algún error

## 👥 Contribuciones

¡Las contribuciones son bienvenidas! Si encuentras algún problema o tienes alguna mejora, no dudes en:

1. Abrir un issue describiendo el problema o mejora
2. Enviar un pull request con los cambios propuestos


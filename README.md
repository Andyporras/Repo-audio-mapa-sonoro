# 🚀 Descomprimir ZIPs Automáticamente con GitHub Actions

Este repositorio utiliza **GitHub Actions** para descomprimir automáticamente todos los archivos `.zip` que se suban o modifiquen en cualquier carpeta del repositorio.

## ✅ ¿Qué hace esta acción?

Cada vez que haces un push con uno o más archivos `.zip`, el flujo de trabajo:

1. Detecta cuáles `.zip` son nuevos o modificados.
2. Los descomprime en una carpeta con el mismo nombre (sin la extensión `.zip`).
3. Opcionalmente (puedes habilitarlo), elimina el `.zip` original.
4. Hace commit y push de los archivos descomprimidos automáticamente.

---

## ⚙️ Cómo habilitar GitHub Actions

1. Ve a tu repositorio en GitHub.
2. Haz clic en la pestaña **"Actions"**.
3. Si no es la primera, haz clic en el botón **"New workflow"** y luego hace el paso 4
4. Si es la primera vez, haz clic en el botón **"Set up a workflow yourself"** para activar Actions.
5. ¡Listo! Ya puedes usar agregar el flujo de trabajo

---

## 📂 Agregar este flujo de trabajo

1. Una vez hecho lo pasado pegue este codigo y cambien el nombre del archivo a  `auto-unzip.yml`.
2. Ahora solo debe tocar el boton que dice **"Commit changes"**

```yaml
name: Descomprimir Todos los ZIPs Subidos

on:
  push:
    paths:
      - '**.zip'

jobs:
  unzip_all_zips:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout del repositorio
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Encontrar y Descomprimir Archivos ZIP
        id: unzip_step
        run: |
          echo "Buscando archivos .zip nuevos o modificados..."
          if [[ "${{ github.event.before }}" == "0000000000000000000000000000000000000000" ]]; then
            zip_files_to_process=$(git ls-tree --full-tree -r --name-only ${{ github.sha }} | grep '\.zip$' || true)
          else
            zip_files_to_process=$(git diff --name-only ${{ github.event.before }} ${{ github.sha }} | grep '\.zip$' || true)
          fi

          if [ -z "$zip_files_to_process" ]; then
            echo "unzipped_something=false" >> $GITHUB_OUTPUT
            exit 0
          fi

          unzipped_files_list_for_commit=()

          for zip_file in $zip_files_to_process; do
            if [ -f "$zip_file" ]; then
              parent_dir=$(dirname "$zip_file")
              base_name_no_ext=$(basename "$zip_file" .zip)
              extraction_dir="$parent_dir/$base_name_no_ext"
              if [[ "$parent_dir" == "." ]]; then
                extraction_dir="$base_name_no_ext"
              fi
              mkdir -p "$extraction_dir"
              unzip -q -o "$zip_file" -d "$extraction_dir"
              unzipped_files_list_for_commit+=("$zip_file")
            fi
          done

          if [ ${#unzipped_files_list_for_commit[@]} -gt 0 ]; then
            echo "unzipped_something=true" >> $GITHUB_OUTPUT
            processed_files_str=$(IFS=, ; echo "${unzipped_files_list_for_commit[*]}")
            echo "processed_zips=$processed_files_str" >> $GITHUB_OUTPUT
          else
            echo "unzipped_something=false" >> $GITHUB_OUTPUT
          fi

      - name: Configurar Git para el Commit
        if: steps.unzip_step.outputs.unzipped_something == 'true'
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'

      - name: Hacer Commit de los Archivos Descomprimidos
        if: steps.unzip_step.outputs.unzipped_something == 'true'
        run: |
          git add .
          commit_message="Autounzip: se procesaron los archivos ZIP: ${{ steps.unzip_step.outputs.processed_zips }}"
          if git diff --staged --quiet; then
            echo "No hay cambios para hacer commit."
          else
            git commit -m "$commit_message"
            git push



```

## 🧪 Cómo probarlo
1. Sube un archivo .zip en cualquier carpeta del repositorio y haz push.
2. Ve a la pestaña Actions y verás que se ejecuta el workflow.
3. Revisa que se haya creado una carpeta con el contenido del ZIP, y que se haya hecho commit automáticamente.

## 📝 Notas
* Este flujo no elimina los archivos .zip después de descomprimirlos. Si quieres eso, descomenta la línea rm "$zip_file" en el script.
* Ideal para proyectos donde se espera subir datos comprimidos y se necesita que estén descomprimidos automáticamente.

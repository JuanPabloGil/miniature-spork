Instalar Ruby y Rails:

Asegúrate de tener Ruby y Rails instalados.
Verifica la instalación:
```
ruby -v
rails -v
```
Crear un proyecto Rails nuevo (si no tienes uno):

```
rails new my_api_app --api
cd my_api_app
```
Configurar rutas:

Abre config/routes.rb y agrega una ruta que apunte a un controlador:
```
Rails.application.routes.draw do
  post '/run_service', to: 'services#run'
end
```
Crear un controlador para manejar el endpoint:

Genera el controlador con:
```
rails generate controller Services
```
En el archivo app/controllers/services_controller.rb define la acción run:
```
class ServicesController < ApplicationController
  def run
    result = MyService.new.call(params[:input])
    render json: { result: result }
  rescue StandardError => e
    render json: { error: e.message }, status: :internal_server_error
  end
end
```
Crear el script Ruby como un Service Object:

Crea un archivo en app/services/my_service.rb:
```
class MyService
  def initialize
    # Configura las variables o dependencias necesarias aquí
  end

  def call(input)
    # Tu lógica principal aquí
    "Hola, #{input}! Este es el resultado del script Ruby."
  end
end
```
Permitir parámetros en el controlador (opcional si usas parámetros):

Agrega un método privado en el controlador para filtrar parámetros:
```
private

def service_params
  params.require(:input)
end
```

Prueba el endpoint localmente:

Inicia el servidor Rails:
```
rails server
```
Envía una solicitud POST para probar el servicio (usando curl o Postman):
```
curl -X POST http://localhost:3000/run_service -d 'input=Mundo' -H "Content-Type: application/json"
```

Deberías obtener una respuesta JSON como:
```
{ "result": "Hola, Mundo! Este es el resultado del script Ruby." }
```

Manejo de errores y mejoras:
En el controlador, captura errores con bloques rescue y devuelve mensajes claros en formato JSON.
Agrega validaciones en el servicio si es necesario.



### Ejemplo de script (service)
```ruby
require 'google/cloud/storage'
require 'net/sftp'
require 'logger'

class UploadService
  SFTP_CONFIG = {
    host: '',
    port: '',
    username: '',
    password: ''
  }.freeze

  def initialize(bucket_name, log_file = 'upload_service.log')
    @bucket_name = bucket_name
    @logger = Logger.new(log_file, 'daily')
    @storage = Google::Cloud::Storage.new
  end

  def run
    log("Starting Upload Service...")
    loop do
      upload_latest_file_to_sftp
      sleep(3600) # Check every hour (adjust as needed)
    end
  rescue StandardError => e
    log("Service crashed: #{e.message}\n#{e.backtrace.join("\n")}")
  end

  private

  def upload_latest_file_to_sftp
    log("Fetching files from bucket: #{@bucket_name}")

    # 1. List files from Google Cloud Storage bucket
    bucket = @storage.bucket(@bucket_name)
    files = bucket.files

    if files.empty?
      log("No files found in the bucket.")
      return
    end

    # 2. Find the latest file
    latest_file = files.max_by(&:updated_at)
    log("Found latest file: #{latest_file.name}")

    # 3. Download the file locally
    temp_file_path = "/tmp/#{latest_file.name}"
    latest_file.download(temp_file_path)
    log("Downloaded #{latest_file.name} to #{temp_file_path}")

    # 4. Connect and upload via SFTP
    Net::SFTP.start(SFTP_CONFIG[:host], SFTP_CONFIG[:username], 
                    password: SFTP_CONFIG[:password], 
                    port: SFTP_CONFIG[:port]) do |sftp|
      remote_path = "home/#{latest_file.name}"
      sftp.upload!(temp_file_path, remote_path)
      log("Uploaded #{latest_file.name} to #{remote_path}")
    end
  rescue StandardError => e
    log("Error during upload: #{e.message}\n#{e.backtrace.join("\n")}")
  end

  def log(message)
    puts message
    @logger.info(message)
  end
end

```

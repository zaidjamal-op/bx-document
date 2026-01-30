📚 Actualización de la Documentación: Capas y Zonas Geográficas

Esta sección documenta los nuevos endpoints para la consulta de información geográfica de Capas y Zonas, implementados en el repositorio bx-srv-georf-geographical-data.

1. Consulta de Capas y Zonas (Layers & Zones)

Se han añadido dos nuevos endpoints al controlador LayerController para consultar metadatos de capas y sus geometrías asociadas, utilizando la base de datos Oracle.

1.1. Obtener Capas Geográficas

Permite obtener un listado de capas disponibles, filtradas por la base y el tipo de entrega/retiro.

Propiedad

Descripción

Ruta

GET /layers

Controlador

LayerController

Método

getLayers(baseId, type)

Response (200 OK)

application/json, Listado de LayerDto

Response (204 No Content)

La búsqueda no arrojó resultados.

Parámetros de Consulta (Query Params)

Parámetro

Tipo

Requerido

Descripción

Ejemplo

baseId

String

Sí

Código de la base (CSO, PSTN).

CSO000001

type

String

Sí

Tipo de capa: EN (Entrega) o RE (Retiro).

EN

Schema: LayerDto

Atributo

Tipo

Descripción

Ejemplo

layerCode

String

Código único de la capa.

CAPA-METRO

baseCode

String

Código de la base a la que pertenece la capa.

CSO000001

priority

Integer

Prioridad de visualización/uso de la capa.

1

layerType

String

Tipo de capa (EN / RE).

EN

layerName

String

Nombre descriptivo de la capa.

Cobertura Metropolitana

1.2. Obtener Zonas de una Capa Específica

Permite obtener las geometrías detalladas de todas las zonas asociadas a una capa, en formato GeoJSON FeatureCollection.

Propiedad

Descripción

Ruta

GET /layers/{layerId}/zones

Controlador

LayerController

Método

getZones(layerId)

Response (200 OK)

application/json, Objeto GeoJSON FeatureCollection

Response (204 No Content)

La capa existe, pero no tiene zonas definidas.

Parámetros de Ruta (Path Params)

Parámetro

Tipo

Requerido

Descripción

Ejemplo

layerId

String

Sí

Código único de la capa (layerCode).

CAPA-METRO

Schema: GeoJSON FeatureCollection

La respuesta es un objeto JSON que adhiere al estándar GeoJSON. La información geográfica (geometry y properties) se genera de forma optimizada en la base de datos a través de funciones espaciales de Oracle.

Atributo

Tipo

Descripción

type

String

Siempre "FeatureCollection".

features

Array<Feature>

Lista de objetos "Feature", donde cada uno representa una zona geográfica.

2. Configuración de Infraestructura y Secretos (Oracle)

Para el correcto funcionamiento de la capa de persistencia (LayerDao y LayerMapper), el servicio requiere las siguientes variables de ambiente para la conexión a Oracle.

2.1. Variables de Ambiente (Settings / ConfigMaps)

Variable

Uso

ORACLE_DS_URL

URL de conexión JDBC al datasource de Oracle.

BS_LOG_LEVEL

Nivel de logging de la aplicación (Ej: INFO, ERROR).

2.2. Secretos Requeridos (Secrets)

Las credenciales de acceso a la base de datos se inyectan a través de secretos de Kubernetes o del entorno local.

Secreto (Key)

Descripción

ORACLE_USERNAME

Nombre de usuario para la conexión a Oracle.

ORACLE_PASSWORD

Contraseña para la conexión a Oracle.

2.3. Configuración del Datasource (application-dev.yaml)

La configuración del datasource utiliza las variables anteriores e incluye parámetros de conexión HikariCP:

Parámetro

Configuración

spring.datasource.url

${ORACLE_DS_URL}

spring.datasource.username

${ORACLE_USERNAME}

spring.datasource.password

${ORACLE_PASSWORD}

hikari.maximum-pool-size

5

hikari.minimum-idle

1

hikari.idle-timeout

300000

hikari.max-lifetime

600000

3. Uso de Caching (Caffeine)

Se confirma que el servicio utiliza Caffeine para la configuración de caching. Aunque no se evidenció un cache específico para las nuevas capas/zonas en las imágenes, la configuración base está definida en:

Archivo

Key de Cache

application-k8s.yaml

cache-names: posts-and-bases, offices, warehouses-by-office

application-dev.yaml

cache-names: posts-and-bases, offices, warehouses-by-office

postman curls 

curl --location 'http://ejpumi.dev.blue.private/georf/srv/geographical-data/layers/SCLEN00007/zones'



curl --location 'http://ejpumi.dev.blue.private/georf/srv/geographical-data/layers?baseId=SCL&type=EN'




API Layers — type=EN | RE (Entrega / Retiro)

Repositorio: bx-srv-georf-geographical-data
Módulo / Carpeta: service
Controller dueño: LayersController
Base path: /layers

Propósito

Exponer capas geográficas y sus zonas asociadas, filtradas por:

Base / Posta → baseId → CAPA.PSTA_CDG

Tipo de capa → type → CAPA.CAPA_SWT_ENTREGA_RETIRO

EN = Entrega

RE = Retiro

Además, permite obtener las zonas/polígonos de una capa en formato GeoJSON FeatureCollection.

Endpoints

1. Obtener capas por base y tipo

GET

/layers?baseId={PSTA_CDG}&type={EN|RE}


Parámetros de query

Parámetro

Requerido

Validación

Significado

baseId

Sí

@NotBlank

Código de base/posta (CAPA.PSTA_CDG)

type

Sí

@NotBlank + `@Pattern("^(EN

RE)$")`

Importante:
type debe venir en mayúsculas (EN o RE).
Si llega en minúsculas (en), la validación falla con 400 antes de que el Service lo normalice.

Respuestas

HTTP

Descripción

200 OK

Arreglo JSON de LayerDto

404 Not Found

No existen capas (body null, comportamiento actual)

400 Bad Request

Parámetros faltantes, vacíos o type inválido

500 Internal Server Error

Error inesperado (handler global)

Ejemplo response 200

[
  {
    "layerCode": "123",
    "baseCode": "SCL",
    "priority": 1,
    "layerType": "EN",
    "layerName": "Capa entrega zona norte"
  }
]


Ejemplo response 404

HTTP 404
Body: null


Ejemplos cURL

Capas de entrega (EN)

curl --location \
'http://<host>/georf/srv/geographical-data/layers?baseId=SCL&type=EN' \
-H 'Authorization: Bearer <token>' \
-H 'x-api-key: <api-key>'


Capas de retiro (RE)

curl --location \
'http://<host>/georf/srv/geographical-data/layers?baseId=SCL&type=RE' \
-H 'Authorization: Bearer <token>' \
-H 'x-api-key: <api-key>'


2. Obtener zonas (polígonos) de una capa como GeoJSON

GET

/layers/{layerId}/zones


Parámetros de path

Parámetro

Requerido

Validación

Significado

layerId

Sí

@NotBlank

Código de capa (CAPA.CDG)

Respuestas

HTTP

Descripción

200 OK

FeatureCollection GeoJSON

400 Bad Request

layerId vacío

500 Internal Server Error

Error inesperado

Nota: Siempre retorna 200, incluso si no hay zonas (features: []).

Ejemplo response 200 (sin zonas)

{
  "type": "FeatureCollection",
  "features": []
}


Ejemplo response 200 (con zonas)

{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Polygon",
        "coordinates": [[[...]]]
      },
      "properties": {
        "zoneId": "456",
        "layerId": "123",
        "zoneName": "Zona A",
        "post": "SCL",
        "routeNumber": "10",
        "bussinessType": "01",
        "createdAt": "2026-01-21T10:11:12Z"
      },
      "id": "456"
    }
  ]
}


Ejemplo cURL

curl --location \
'http://<host>/georf/srv/geographical-data/layers/123/zones' \
-H 'Authorization: Bearer <token>' \
-H 'x-api-key: <api-key>'


Seguridad

El controller declara:

@SecurityRequirement(name = "BearerAuth")
@SecurityRequirement(name = "ApiKeyAuth")


Headers requeridos:

Authorization: Bearer <jwt>

x-api-key: <api-key>

Referencia: oas.yaml → components/securitySchemes
(Confirmar nombre exacto del header del API Key).

Flujo interno (request path)

Controller

Archivo

src/main/java/cl/bluex/geographical/service/interfaces/controller/LayersController.java


Responsabilidades

Valida parámetros requeridos

Fuerza type a EN | RE

Retorna 404 si no hay resultados (no 200 [])

@GetMapping("")
public ResponseEntity<List<LayerDto>> getLayersByBaseAndType(
  @RequestParam @NotBlank String baseId,
  @RequestParam @NotBlank @Pattern(regexp="^(EN|RE)$") String type
) {
  List<LayerDto> layers = layersService.getLayersByBaseAndType(baseId, type);
  if (CollectionUtils.isEmpty(layers)) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(null);
  }
  return ResponseEntity.ok(layers);
}


Service

Archivo

src/main/java/cl/bluex/geographical/service/application/service/LayersService.java


Responsabilidades

Normaliza inputs a mayúsculas

Mapea Layer → LayerDto

Construye FeatureCollection GeoJSON

Omite features vacías o JSON inválido (log de warning)

Retorna GeoJSON como String

@Transactional(readOnly = true)
public List<LayerDto> getLayersByBaseAndType(String baseId, String layerType) {
  String base = StringUtils.upperCase(baseId);
  String type = StringUtils.upperCase(layerType);
  List<Layer> layers = layersDao.getLayersByBaseAndLayerType(base, type);
  return layers.stream().map(LayerDto::fromEntity).collect(Collectors.toList());
}


DAO y Mapper (Oracle / MyBatis)

Archivos

LayersDao.java
LayersMapper.java


SQL — capas por base y tipo

SELECT
  CAPA.CAPA_CDG AS layerCode,
  CAPA.PSTA_CDG AS baseCode,
  CAPA.CAPA_PRIORIDAD AS priority,
  CAPA.CAPA_SWT_ENTREGA_RETIRO AS layerType,
  CAPA.CAPA_DESCRIPCION AS layerName
FROM CAPA
WHERE CAPA.PSTA_CDG = #{baseCode}
  AND CAPA.CAPA_SWT_ENTREGA_RETIRO = #{layerType}
  AND CAPA.CAPA_ESTADO = 'PR'
ORDER BY CAPA.CAPA_PRIORIDAD


Notas

Solo capas activas: CAPA_ESTADO = 'PR'

Ordenadas por prioridad (CAPA_PRIORIDAD)

SQL — zonas como GeoJSON Feature

Tabla: exgeogra.georef_zona

Geometría convertida con SDO_UTIL.TO_GEOJSON(...)

Cada fila retorna un Feature JSON (CLOB)

El Service envuelve todo en un FeatureCollection
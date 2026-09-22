# Collection Bruno - UrpiPro

Repositorio de una colección en formato OpenCollection para ejecutar pruebas de APIs de UrpiPro con Bruno. Reúne solicitudes de relación con clientes, originación de préstamos, servicios transversales y utilidades para localizar datos de prueba.

## Objetivo

La colección permite a QA ejercitar flujos y consultas de UrpiPro a través de las rutas organizadas como `apim`, `aks` y `apis-MiBanco`. Incluye solicitudes de autenticación y un script global que prepara tokens antes de ejecutar solicitudes identificadas como APIM, AKS o transversales.

No es una suite de pruebas automatizadas con aserciones declaradas: las solicitudes se ejecutan individualmente desde Bruno y algunos flujos registran resultados en la consola.

## Requisitos

- Bruno con soporte para colecciones OpenCollection/YAML.
- Acceso autorizado a las redes y APIs de los ambientes que se vayan a usar.
- Credenciales, suscripciones, encabezados y permisos que requieran las APIs. Solicítelos por el canal autorizado; no están documentados aquí.
- Certificado de cliente y su contraseña, entregados por el equipo responsable cuando correspondan.

## Estructura del repositorio

```text
collection_brunomi/
├── certs/                         # certificados locales; ignorados por Git
├── Urpipro_Apis/                  # colección OpenCollection
│   ├── apim/
│   │   ├── RC/                    # Customer Relationship
│   │   └── LOAN/                  # originación de préstamos
│   ├── aks/
│   │   ├── RC/
│   │   ├── LOAN/
│   │   └── Revisar/               # solicitudes de revisión/manuales
│   ├── apis-MiBanco/              # servicios transversales de UrpiPro
│   ├── auth/                      # obtención y preparación de tokens
│   ├── environments/              # QA y STG
│   ├── Get Test Data/             # búsquedas y CSV de datos de prueba
│   ├── client-cert-config.json    # configuración de certificado de cliente
│   └── opencollection.yml         # definición y configuración global
├── .gitignore
└── README.md
```

`opencollection.yml` define la colección `Urpipro_Apis`, la configuración de proxy heredada y el certificado de cliente configurado a nivel de colección. También conserva un archivo `opencollection.yml.backup`; no se identifica un propósito adicional para él desde la configuración activa.

Los archivos `folder.yml` representan carpetas de la colección y los demás YAML contienen solicitudes HTTP. No hay archivos `.bru` en el repositorio actual.

## Colecciones / APIs

### `auth`

Contiene `Token APIM`, `Token Front QA` y `Save Redis`. Las dos solicitudes de token guardan, mediante scripts `after-response`, los tokens obtenidos en variables de ejecución compartidas: `apim_access_token` y `ms_access_token`. `Save Redis` es una solicitud auxiliar usada por el script global de la colección.

### `apim`

Agrupa solicitudes que usan `{{apim_base_url}}`.

- `RC` (Customer Relationship): consultas de clientes —perfil básico y completo, portafolio y mora—, empleados, representante de ventas, *feature toggles* y limpieza de caché de perfil en Redis.
- `LOAN`: elegibilidad, simulación, tasas, creación, consulta, listado, cancelación y confirmación de solicitudes; cuentas de depósito y evaluación financiera. Incluye subcarpetas de `Pre aprobados` y `Segunda Linea`.

La solicitud `Eliminar datos en redis v1` usa `DELETE` para limpiar datos de perfil en caché. Ejecútala únicamente contra un cliente y ambiente autorizados.

### `aks`

Contiene rutas equivalentes o de validación para `RC` (clientes, empleados y `handshake`) y `LOAN`. La carpeta `Revisar` concentra solicitudes manuales de distintos flujos de préstamo, incluidas variantes BFF, ofertas, cierre en TOPAZ y segunda línea.

Varias solicitudes de esta área tienen URLs definidas directamente y otras hacen referencia a `{{aks_base_url}}`. Revise la URL y los datos de cada solicitud antes de ejecutarla, especialmente las operaciones `POST` y `PUT`.

### `apis-MiBanco`

Incluye servicios transversales de UrpiPro: tasas, productos, autonomía de ADN, préstamos en TOPAZ, información del cliente y línea de crédito disponible.

### `Get Test Data`

Incluye búsquedas de clientes sin correo, con campaña, con pre-solicitudes y con solicitudes. Todas consultan el perfil del cliente usando `{{dni}}`; sus scripts `after-response` inspeccionan la respuesta y escriben coincidencias en la consola de Bruno. No modifican el CSV ni generan archivos.

## Ambientes

Los ambientes versionados son:

| Ambiente | Variable declarada |
| --- | --- |
| `QA` | `apim_base_url` |
| `STG` | `apim_base_url` |

Abra `Urpipro_Apis` en Bruno y seleccione `QA` o `STG` en el selector de ambiente antes de ejecutar solicitudes de `apim`. Los valores se mantienen en `Urpipro_Apis/environments/` y no se reproducen en este documento.

La colección también referencia `aks_base_url`, pero esa variable no está declarada en los archivos de ambiente actuales. Si va a ejecutar una solicitud que la use, confirme con el equipo responsable cómo debe proporcionarse; no asuma ni agregue una URL. Otras variables que aparecen en solicitudes, como `applicationId`, `dni`, identificadores de cliente o de empleado, deben revisarse y sustituirse por datos de prueba autorizados cuando aplique.

## Autenticación y scripts compartidos

El `before-request` de `opencollection.yml` evita ejecutarse sobre las propias solicitudes de `auth`. Para el resto, identifica el tipo de URL e intenta preparar la autenticación:

- solicitudes transversales y AKS: ejecuta `Token Front QA` y luego `Save Redis`;
- solicitudes APIM: ejecuta `Token APIM`, `Token Front QA` y luego `Save Redis`.

Los tokens quedan como variables de ejecución de Bruno, no como valores de los archivos de ambiente. Los encabezados de las solicitudes también contienen parámetros de autorización, suscripción y correlación; use sólo valores autorizados y nunca los copie a documentación, incidencias o commits.

## Certificados

La configuración activa de la colección y `client-cert-config.json` apuntan a un certificado PKCS#12 ubicado en `certs/`, mediante una ruta relativa desde `Urpipro_Apis`. La configuración lo asocia con el dominio APIM indicado por la colección.

La carpeta `certs/` está ignorada por Git. Después de clonar, coloque allí el certificado que le entregue el equipo responsable y configure/importelo en Bruno conforme a la configuración de certificado de cliente de la colección. La contraseña debe obtenerse por un canal seguro: este README no publica contraseñas, claves ni contenido de certificados.

## Configuración inicial

1. Clone el repositorio y abra la carpeta `Urpipro_Apis/` como colección en Bruno.
2. Solicite y coloque el certificado de cliente requerido dentro de `certs/`, sin añadirlo al control de versiones.
3. En Bruno, valide la configuración de certificado de cliente de la colección y proporcione la contraseña sólo en el mecanismo seguro aprobado por su equipo.
4. Seleccione `QA` o `STG` y confirme que `apim_base_url` corresponde al ambiente autorizado.
5. Antes de usar rutas AKS con variables, confirme la fuente de `aks_base_url`, ya que no figura en los ambientes versionados.
6. Revise los valores de ruta, consulta, encabezados y cuerpo de la solicitud que ejecutará. Sustituya los identificadores de ejemplo por datos de prueba autorizados cuando sea necesario.
7. Ejecute una consulta no mutante adecuada al ambiente. La preparación de tokens se dispara desde el script global para las URL que reconoce; ante un error de autenticación, ejecute y revise las solicitudes de `auth` según los permisos disponibles.

## Uso de la colección

Navegue por el área funcional requerida y ejecute una solicitud desde Bruno. Los grupos más comunes son:

- Perfil, portafolio, mora y empleados en `apim/RC` o `aks/RC`.
- Ciclo de solicitud de préstamo en `apim/LOAN` y `aks/LOAN`: elegibilidad, simulación, creación, consulta, firma, confirmación y cancelación.
- Casos de preaprobados y segunda línea en sus subcarpetas correspondientes.
- Servicios de información y productos en `apis-MiBanco`.

No ejecute en bloque solicitudes que cambien estado (`POST`, `PUT` o `DELETE`) sin validar el ambiente, el cliente de prueba y el efecto esperado. Algunas solicitudes conservan identificadores o URLs directas en su definición; revíselas antes de enviarlas.

## Datos de prueba

`Urpipro_Apis/Get Test Data/clientes.csv` contiene una lista local de clientes para pruebas. Las cuatro solicitudes de búsqueda usan la variable `dni` y muestran en la consola sólo los casos que cumplen las condiciones definidas por sus scripts: ausencia de correo, campaña existente, pre-solicitud aceptada o solicitud en determinados estados.

El repositorio no declara un enlace automático entre el CSV y las solicitudes. Trate su contenido y las respuestas como datos sensibles; úselo exclusivamente en entornos autorizados y no lo copie a documentación, tickets o repositorios externos.

## Buenas prácticas

- Use el ambiente de Bruno en lugar de cambiar URLs de las solicitudes.
- Renueve o revise los tokens mediante el flujo `auth`; no los pegue en archivos versionados.
- Cambie identificadores de clientes, solicitudes y empleados por datos de prueba autorizados antes de ejecutar operaciones mutantes.
- Revise la consola de Bruno al usar `Get Test Data`, ya que sus scripts pueden registrar datos de respuesta.
- Mantenga certificados y archivos locales dentro de `certs/`, que ya está excluida en `.gitignore`.

## Seguridad

No suba al repositorio certificados, contraseñas, tokens, claves privadas, claves de suscripción, secretos de encabezados ni valores sensibles de ambientes. Tampoco incluya información personal o respuestas de prueba en documentación o evidencias fuera de los canales autorizados.

## Troubleshooting

- **No se resuelve `{{apim_base_url}}`:** seleccione `QA` o `STG` y valide la variable declarada en ese ambiente.
- **No se resuelve `{{aks_base_url}}`:** no existe en los archivos de ambiente actuales. Confirme la configuración requerida con el equipo responsable.
- **Error TLS o de certificado de cliente:** verifique que el certificado autorizado esté en `certs/` y que la configuración de Bruno use la ruta relativa indicada por la colección. Solicite la contraseña por un canal seguro.
- **401/403 o falla al obtener token:** confirme acceso a red, permisos, credenciales y encabezados autorizados. Revise la salida de las solicitudes de `auth` sin exponer sus valores.
- **La búsqueda de datos no muestra coincidencias:** compruebe el valor de `dni`, el ambiente seleccionado y los criterios exactos de cada script `after-response`.
- **Una operación modifica datos inesperadamente:** detenga el flujo y valide el ambiente, la URL y los identificadores; la colección contiene solicitudes `POST`, `PUT` y `DELETE`.

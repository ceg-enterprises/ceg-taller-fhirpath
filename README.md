# Taller FHIRPath - Connectathon HL7 Chile 2026

> **Taller practico sobre FHIRPath:** Aprende a usar expresiones FHIRPath para crear invariantes de validacion y SearchParameters personalizados en FHIR.

---

## Que es FHIRPath?

**FHIRPath** es un lenguaje de navegacion y extraccion de datos disenado especificamente para recursos FHIR. Piensa en el como "XPath para FHIR" - te permite:

- **Navegar** por la estructura de un recurso FHIR
- **Filtrar** elementos basandose en condiciones
- **Extraer** valores especificos
- **Validar** que los datos cumplan ciertas reglas

### Ejemplo basico

```fhirpath
Patient.name.where(use = 'official').family
```

Esta expresion:
1. Parte del recurso `Patient`
2. Accede a todos los elementos `name`
3. Filtra solo los que tienen `use = 'official'`
4. Retorna el valor de `family`

### Por que es importante?

FHIRPath se usa en dos lugares criticos de FHIR:

| Uso | Descripcion | Ejemplo |
|-----|-------------|---------|
| **Invariantes** | Reglas de validacion en StructureDefinitions | "El RUT debe tener formato valido" |
| **SearchParameters** | Definir como buscar recursos | "Buscar pacientes por su RUT" |

---

## Requisitos Previos

Antes de comenzar, asegurate de tener:

- [ ] **Docker Desktop** instalado y corriendo ([Descargar](https://www.docker.com/products/docker-desktop))
- [ ] **Navegador web moderno** (Chrome, Firefox, Edge)
- [ ] **(Opcional)** VS Code con extension [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)

### Verificar Docker

```bash
docker --version
# Deberia mostrar: Docker version 24.x.x o superior
```

---

## Inicio Rapido

### Paso 1: Clonar o descargar el repositorio

```bash
git clone https://github.com/ceg-enterprises/ceg-taller-fhirpath.git
cd ceg-taller-fhirpath
```

### Paso 2: Levantar el servidor HAPI FHIR

```bash
docker-compose up -d
```

> **Nota:** La primera vez puede tomar 2-3 minutos mientras descarga la imagen. Espera aproximadamente 60 segundos adicionales a que el servidor inicie completamente.

### Paso 3: Verificar que el servidor funciona

Abre en tu navegador: **http://localhost:8081/fhir/metadata**

Deberias ver un documento JSON grande (el CapabilityStatement del servidor).

### Paso 4: Abrir el panel interactivo del taller

Abre el archivo `demo.html` en tu navegador:
- **Windows:** Doble click en el archivo, o arrastralo al navegador
- **Mac/Linux:** `open demo.html` o `xdg-open demo.html`

El panel mostrara **"Conectado"** cuando detecte el servidor FHIR.

---

## Estructura del Taller

El taller esta dividido en **10 pasos** organizados en dos secciones:

### Parte 1: Invariantes (Pasos 1-6)

Aprenderemos como FHIRPath se usa en **constraints** (restricciones) dentro de un StructureDefinition para validar que los recursos cumplan ciertas reglas.

### Parte 2: SearchParameters (Pasos 7-10)

Aprenderemos como FHIRPath se usa para definir **parametros de busqueda personalizados** que el servidor FHIR no conoce por defecto.

---

## Parte 1: Invariantes FHIRPath

### Concepto Clave

Un **invariante** es una regla de validacion definida en un StructureDefinition. Tiene dos partes importantes:

```json
{
  "key": "cl-rut-formato",
  "severity": "warning",  // o "error"
  "human": "El RUT debe tener formato valido",
  "expression": "identifier.where(system='http://regcivil.cl/run').all(value.matches('^[0-9]+-[0-9Kk]$'))"
}
```

| Propiedad | Descripcion |
|-----------|-------------|
| `key` | Identificador unico del invariante |
| `severity` | **warning** = solo avisa, **error** = bloquea |
| `human` | Mensaje legible para humanos |
| `expression` | La expresion FHIRPath que debe evaluarse a `true` |

### Los Tres Invariantes del Taller

#### 1. cl-rut-formato (Formato de RUT Chileno)

**Proposito:** Validar que el RUT tenga el formato correcto (digitos-digito verificador)

| Version | Severidad | Expresion FHIRPath |
|---------|-----------|-------------------|
| v1 (Permisivo) | warning | `value.matches('^[0-9]+-[0-9Kk]$')` |
| v2 (Estricto) | **error** | `value.matches('^[0-9]{7,8}-[0-9Kk]$')` |

**Diferencia:** La v2 exige exactamente 7-8 digitos antes del guion (un RUT real).

#### 2. cl-contacto-emergencia (Contacto para Adultos Mayores)

**Proposito:** Los pacientes mayores de 65 anos deben tener un contacto de emergencia registrado.

| Version | Severidad | Expresion FHIRPath |
|---------|-----------|-------------------|
| v1 | warning | `contact.exists() or birthDate.empty()` |
| v2 | **error** | `birthDate.empty() or (edad < 65) or contact.exists()` |

**Expresion v2 completa:**
```fhirpath
birthDate.empty() or
(today().toString().substring(0,4).toInteger() - birthDate.toString().substring(0,4).toInteger() < 65) or
contact.exists()
```

**Explicacion:** Calcula la edad restando el ano de nacimiento del ano actual.

#### 3. cl-telefono-movil (Telefono Movil Chileno)

**Proposito:** El paciente debe tener un telefono movil con formato chileno (+56).

| Version | Severidad | Expresion FHIRPath |
|---------|-----------|-------------------|
| v1 | warning | Si existe movil, debe empezar con +56 |
| v2 | **error** | DEBE existir movil Y empezar con +56 |

**Expresion v2:**
```fhirpath
telecom.where(system='phone' and use='mobile').exists() and
telecom.where(system='phone' and use='mobile').all(value.startsWith('+56'))
```

---

### Ejercicios Paso a Paso

#### Paso 1: Cargar Perfil v1 (Permisivo)

**Que hacemos:** Enviamos un StructureDefinition con 3 invariantes configurados como `severity: warning`.

**Por que:** Establecemos las reglas base. Como son warnings, los recursos que no cumplan igual seran aceptados (solo veran advertencias).

**Metodo:** `POST /StructureDefinition`

---

#### Paso 2: Validar Juan Gonzalez

**Que hacemos:** Validamos un paciente que cumple TODAS las reglas.

**Datos del paciente:**
- RUT: `12345678-9` (formato correcto, 8 digitos)
- Edad: 34 anos (no requiere contacto)
- Telefono: `+56912345678` (movil chileno)

**Resultado esperado:** Validacion exitosa sin warnings ni errores.

**Metodo:** `POST /Patient/$validate`

---

#### Paso 3: Validar Maria Silva

**Que hacemos:** Validamos un paciente que NO cumple ninguna regla.

**Datos del paciente:**
- RUT: `123-4` (muy corto, solo 3 digitos)
- Edad: 69 anos, SIN contacto de emergencia
- Sin telefono movil registrado

**Resultado esperado:** Validacion pasa pero con 3 WARNINGS (porque v1 usa `severity: warning`).

**Metodo:** `POST /Patient/$validate`

**Punto clave:** Maria "pasa" la validacion porque los warnings no bloquean.

---

#### Paso 4: Cargar Perfil v2 (Estricto)

**Que hacemos:** Actualizamos el StructureDefinition cambiando todos los `severity: warning` a `severity: error`.

**Por que:** Ahora las mismas reglas seran OBLIGATORIAS. Si un recurso no cumple, la validacion FALLA.

**Cambios en las expresiones:**
- cl-rut-formato: Ahora exige 7-8 digitos
- cl-contacto-emergencia: Ahora calcula la edad real
- cl-telefono-movil: Ahora EXIGE tener movil

**Metodo:** `PUT /StructureDefinition/PacienteChilenoDemo`

---

#### Paso 5: Validar Juan Gonzalez (con Perfil v2)

**Que hacemos:** Validamos a Juan nuevamente con las reglas estrictas.

**Resultado esperado:** PASA - Juan cumple todas las reglas, incluso las mas estrictas.

**Metodo:** `POST /Patient/$validate`

---

#### Paso 6: Validar Maria Silva (FALLA)

**Que hacemos:** Validamos a Maria con las reglas estrictas.

**Resultado esperado:** FALLA con 3 ERRORES:
1. `cl-rut-formato`: RUT muy corto (solo 3 digitos, necesita 7-8)
2. `cl-contacto-emergencia`: Tiene 69 anos y no tiene contacto
3. `cl-telefono-movil`: No tiene telefono movil

**Metodo:** `POST /Patient/$validate`

**Punto clave:** El MISMO recurso que antes "pasaba" (con warnings), ahora FALLA porque cambiamos la severidad de las reglas.

---

## Parte 2: SearchParameters FHIRPath

### Concepto Clave

Un **SearchParameter** le ensena al servidor FHIR como buscar recursos usando un parametro personalizado.

```json
{
  "resourceType": "SearchParameter",
  "code": "rut",
  "base": ["Patient"],
  "type": "token",
  "expression": "Patient.identifier.where(system='http://regcivil.cl/run')"
}
```

| Propiedad | Descripcion |
|-----------|-------------|
| `code` | Nombre del parametro (se usara en la URL) |
| `base` | Recursos donde aplica (Patient, Observation, etc.) |
| `type` | Tipo de busqueda (token, string, date, reference, etc.) |
| `expression` | FHIRPath que extrae el valor a indexar |

### Ejercicios Paso a Paso

#### Paso 7: Crear Pacientes de Prueba

**Que hacemos:** Creamos 3 pacientes en el servidor para poder buscarlos despues.

**Pacientes:**
| Nombre | RUT |
|--------|-----|
| Juan Gonzalez | 12345678-9 |
| Maria Silva | 123-4 |
| Pedro Lagos | 98765432-1 |

**Metodo:** `POST /Patient` (x3)

---

#### Paso 8: Buscar por RUT (sin SearchParameter)

**Que hacemos:** Intentamos buscar un paciente por su RUT.

**URL:** `GET /Patient?rut=12345678-9`

**Resultado esperado:** El servidor NO conoce el parametro "rut", asi que:
- Retorna TODOS los pacientes (ignora el parametro), o
- Retorna un error indicando parametro desconocido

**Punto clave:** FHIR no sabe buscar por RUT porque no es un parametro estandar.

---

#### Paso 9: Cargar SearchParameter RUT

**Que hacemos:** Registramos un SearchParameter que le ensena al servidor como encontrar el RUT.

**Expresion FHIRPath:**
```fhirpath
Patient.identifier.where(system='http://regcivil.cl/run')
```

**Explicacion:**
1. `Patient.identifier` - Accede a todos los identificadores
2. `.where(system='http://regcivil.cl/run')` - Filtra solo los que tienen system de RUT chileno

**Metodo:** `POST /SearchParameter`

> **Nota:** Algunos servidores requieren reindexar despues de crear un SearchParameter. HAPI FHIR lo hace automaticamente.

---

#### Paso 10: Buscar por RUT (con SearchParameter)

**Que hacemos:** Repetimos la misma busqueda del Paso 8.

**URL:** `GET /Patient?rut=12345678-9`

**Resultado esperado:** Ahora el servidor encuentra a Juan Gonzalez.

**Punto clave:** El servidor ahora sabe usar la expresion FHIRPath para extraer e indexar los RUTs.

---

## Pacientes de Prueba (Resumen)

| Paciente | RUT | Edad | Movil | Contacto | Resultado v1 | Resultado v2 |
|----------|-----|------|-------|----------|--------------|--------------|
| Juan Gonzalez | 12345678-9 | 34 | +56912345678 | - | PASA | PASA |
| Maria Silva | 123-4 | 69 | - | - | PASA (warnings) | **FALLA** |
| Pedro Lagos | 98765432-1 | 39 | +56998765432 | Si | PASA | PASA |

---

## Expresiones FHIRPath Utiles

### Navegacion basica
```fhirpath
Patient.name                     // Todos los nombres
Patient.name.first()             // Primer nombre
Patient.name.family              // Todos los apellidos
Patient.telecom.where(system='phone')  // Solo telefonos
```

### Filtros
```fhirpath
.where(condition)                // Filtra elementos
.exists()                        // Verifica si existe al menos uno
.empty()                         // Verifica si esta vacio
.all(condition)                  // Todos deben cumplir
```

### Comparaciones
```fhirpath
value = 'texto'                  // Igualdad
value.startsWith('+56')          // Empieza con
value.matches('^[0-9]+$')        // Expresion regular
value.toInteger() > 65           // Conversion y comparacion
```

### Operadores logicos
```fhirpath
condition1 and condition2        // Ambas deben ser true
condition1 or condition2         // Al menos una debe ser true
condition1.not()                 // Negacion
```

---

## Comandos Docker

```bash
# Iniciar el servidor
docker-compose up -d

# Ver logs en tiempo real
docker-compose logs -f

# Detener el servidor (mantiene datos)
docker-compose down

# Detener y eliminar todos los datos
docker-compose down -v

# Reiniciar el servidor
docker-compose restart
```

---

## Troubleshooting

### El servidor no responde

```bash
# Verificar que el contenedor este corriendo
docker ps

# Ver logs para errores
docker-compose logs hapi-fhir
```

### El panel muestra "Sin conexion"

1. Verifica que Docker este corriendo
2. Verifica que el puerto 8081 no este en uso por otra aplicacion
3. Espera 60 segundos despues de iniciar el servidor

### La validacion no detecta los errores

HAPI FHIR cachea los StructureDefinitions. Soluciones:
```bash
# Opcion 1: Reiniciar
docker-compose restart

# Opcion 2: Eliminar datos y reiniciar
docker-compose down -v
docker-compose up -d
```

### El SearchParameter no funciona

Despues de crear un SearchParameter, algunos servidores necesitan reindexar. En HAPI FHIR:
1. El reindexado es automatico pero puede tomar unos segundos
2. Intenta la busqueda nuevamente despues de 10-15 segundos

---

## Estructura del Proyecto

```
ceg-taller-fhirpath/
├── docker-compose.yml              # Configuracion del servidor HAPI FHIR
├── demo.html                       # Panel interactivo del taller
├── README.md                       # Esta guia
├── logo-ceg-icon.svg              # Logo de CEG Consultores
├── resources/
│   ├── profiles/
│   │   ├── StructureDefinition-PacienteChilenoDemo-v1.json
│   │   └── StructureDefinition-PacienteChilenoDemo-v2.json
│   ├── searchparameters/
│   │   └── SearchParameter-patient-rut.json
│   └── examples/
│       ├── Patient-juan-gonzalez.json
│       ├── Patient-maria-silva.json
│       └── Patient-pedro-lagos.json
└── api/
    └── requests.http               # Requests para VS Code REST Client
```

---

## Recursos Adicionales

### Documentacion Oficial
- [Especificacion FHIRPath](https://hl7.org/fhirpath/) - Documentacion completa del lenguaje
- [FHIRPath en FHIR](https://build.fhir.org/fhirpath.html) - Uso de FHIRPath en el contexto de FHIR
- [Invariantes en FHIR](https://build.fhir.org/elementdefinition-definitions.html#ElementDefinition.constraint) - Como definir constraints

### Herramientas
- [HAPI FHIR](https://hapifhir.io/) - Servidor FHIR usado en este taller
- [FHIRPath Lab](https://fhirpath-lab.azurewebsites.net/) - Prueba expresiones FHIRPath online
- [Simplifier](https://simplifier.net/) - Plataforma para publicar perfiles FHIR

### HL7 Chile
- [CL Core IG](https://build.fhir.org/ig/HL7Chile/clcore_ig/) - Guia de implementacion de Chile
- [HL7 Chile](https://hl7chile.cl/) - Capitulo chileno de HL7

---

## Creditos

Taller desarrollado por **CEG Consultores SpA** para el Connectathon HL7 Chile 2026.

Para consultas o soporte: contacto@ceg.cl

---

*Ultima actualizacion: Enero 2026*

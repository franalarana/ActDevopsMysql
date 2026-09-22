# 🧪 Guía del Estudiante — Pruebas Unitarias, JaCoCo y Pipeline CI
### Asignatura DevOps | Actividad 2.3 — Asegurando calidad en nuestro código

> **¿Qué vas a lograr?**
> Escribir pruebas unitarias para la API REST de Estudiantes, medir la cobertura de código
> con JaCoCo y automatizar todo en un pipeline de GitHub Actions que guarde el reporte
> de cobertura como artefacto. No necesitas levantar contenedores para ejecutar los tests
> — Maven los corre directamente en tu máquina.

---

## 🖥️ Paso 0 — Verifica tu entorno local

Antes de comenzar, confirma que tienes las herramientas necesarias:

| Herramienta | ¿Cómo verificar? | Versión mínima |
|---|---|---|
| **Java JDK** | `java -version` | 17 |
| **Maven** | `mvn -version` | 3.8+ |
| **Git** | `git --version` | cualquiera |
| **Docker Desktop** o **Podman** | `docker --version` / `podman --version` | cualquiera |

> ⚠️ **Para los tests unitarios solo necesitas Java y Maven.** Docker/Podman se usa más adelante
> para el pipeline completo y para levantar el sistema (como en la actividad anterior).

---

## 📦 Paso 1 — Obtén el proyecto

Clona o descomprime el proyecto en tu máquina.

```text
bdget/
├── .github/
│   └── workflows/
│       └── main.yml          ← pipeline de GitHub Actions (lo modificarás aquí)
├── src/
│   ├── main/java/...         ← código fuente de la aplicación
│   └── test/java/...         ← pruebas unitarias (aquí trabajarás)
├── pom.xml                   ← dependencias y configuración de JaCoCo
└── Dockerfile
```

Abre una terminal y entra a la raíz del proyecto:

```bash
cd bdget
```

> ⚠️ **Todos los comandos Maven de esta guía se ejecutan desde la raíz del proyecto (donde está `pom.xml`).**

---

## 🔍 Paso 2 — Entiende la estructura de pruebas existente

El proyecto ya tiene tres clases de test. Ábrelas y lee cada una antes de ejecutar nada.

```text
src/test/java/com/example/bdget/
├── BdgetApplicationTests.java          ← test de arranque del contexto
├── model/
│   └── StudentModelTest.java           ← prueba del modelo (getters/setters)
├── service/
│   └── StudentServiceImplTest.java     ← prueba del servicio con Mockito
└── controller/
    └── StudentControllerTest.java      ← prueba del controlador con MockMvc
```

### ¿Qué tipo de prueba es cada una?

| Clase | Tipo | Herramienta | ¿Levanta contexto Spring? |
|---|---|---|---|
| `StudentModelTest` | Prueba unitaria pura | JUnit 5 | ❌ No |
| `StudentServiceImplTest` | Prueba unitaria con mocks | JUnit 5 + Mockito | ❌ No |
| `StudentControllerTest` | Prueba de capa web | MockMvc + Mockito | ✅ Parcial (`@WebMvcTest`) |

> 💡 **Concepto clave — Prueba unitaria vs. de integración:**
> Una prueba *unitaria* aísla un componente y reemplaza sus dependencias con *mocks*.
> Una prueba de *integración* levanta el sistema completo y conecta componentes reales.
> Esta actividad se enfoca en pruebas unitarias para detectar errores rápido y sin base de datos.

---

## 🔍 Paso 3 — Lee el pom.xml: JaCoCo ya está configurado

Abre `pom.xml` y localiza el plugin de JaCoCo:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <!-- Instrumenta el bytecode ANTES de ejecutar tests -->
        <execution>
            <goals>
                <goal>prepare-agent</goal>   <!-- se ejecuta en fase: initialize -->
            </goals>
        </execution>
        <!-- Genera el reporte HTML DESPUÉS de ejecutar tests -->
        <execution>
            <id>report</id>
            <phase>test</phase>             <!-- se dispara al final de mvn test -->
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

**¿Qué hace JaCoCo?**

JaCoCo instrumenta el bytecode compilado de tu aplicación. Cuando los tests se ejecutan,
registra qué líneas, ramas e instrucciones fueron alcanzadas. Al terminar, genera un reporte
HTML visual en `target/site/jacoco/index.html`.

---

## 🧪 Paso 4 — Ejecuta los tests localmente

### Ejecutar todos los tests y generar cobertura

```bash
mvn test
```

Maven ejecutará las 3 clases de test y, gracias a la configuración de JaCoCo en el `pom.xml`,
generará el reporte automáticamente al finalizar.

Salida esperada (resumen final):

```text
[INFO] Results:
[INFO]
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] --- jacoco-maven-plugin:0.8.10:report (report) ---
[INFO] Loading execution data file .../target/jacoco.exec
[INFO] Analyzed bundle 'bdget' with 5 classes
[INFO] BUILD SUCCESS
```

> ⏱️ La primera vez tarda ~1-2 minutos porque Maven descarga dependencias.
> Las siguientes veces es casi instantáneo.

---

## 📊 Paso 5 — Abre el reporte de cobertura

El reporte HTML queda en:

```text
target/site/jacoco/index.html
```

Ábrelo con tu navegador:

### Con Podman o Docker (si usas WSL/Linux)

```bash
# Abre directamente desde la terminal (macOS)
open target/site/jacoco/index.html

# En Linux
xdg-open target/site/jacoco/index.html
```

### En Windows (PowerShell)

```bash
start target\site\jacoco\index.html
```

### ¿Qué verás en el reporte?

El reporte muestra una tabla con cada paquete y clase:

```text
Package                         | Missed  | Coverage
--------------------------------|---------|----------
com.example.bdget.model         |    0    |  100%  ✅
com.example.bdget.service       |    2    |   85%  🟡
com.example.bdget.controller    |    0    |  100%  ✅
com.example.bdget.repository    |   N/A   |   N/A  (interface)
```

**Interpretación del color en JaCoCo:**

| Color | Significado |
|---|---|
| 🟢 Verde | Línea/rama ejecutada por al menos un test |
| 🔴 Rojo | Línea/rama NO ejecutada — posible gap de cobertura |
| 🟡 Amarillo | Rama parcialmente cubierta (ej: solo el `if`, no el `else`) |

---

## 🔎 Paso 6 — Analiza las pruebas existentes

### `StudentModelTest.java` — Prueba del modelo

```java
@Test
void testGettersAndSetters() {
    Student student = new Student();
    student.setId(1L);
    student.setName("John");
    assertEquals(1L, student.getId());    // verifica getter id
    assertEquals("John", student.getName()); // verifica getter name
}
```

Esta prueba verifica que los getters y setters del modelo `Student` funcionen correctamente.
Es una prueba unitaria pura: no usa Spring, no usa base de datos.

---

### `StudentServiceImplTest.java` — Prueba del servicio

```java
@ExtendWith(MockitoExtension.class)  // activa Mockito
class StudentServiceImplTest {

    @Mock
    private StudentRepository repository;  // MOCK: simula la BD

    @InjectMocks
    private StudentServiceImpl service;    // objeto real bajo prueba

    @Test
    void testUpdateStudentExists() {
        when(repository.existsById(1L)).thenReturn(true);   // simula que el ID existe
        when(repository.save(student)).thenReturn(student);
        Student result = service.updateStudent(1L, student);
        assertEquals(student, result);
        verify(repository).save(student);  // verifica que se llamó a save()
    }

    @Test
    void testUpdateStudentNotExists() {
        when(repository.existsById(1L)).thenReturn(false);  // simula que NO existe
        assertNull(service.updateStudent(1L, student));     // debe retornar null
        verify(repository, never()).save(any());             // NO debe llamar a save()
    }
}
```

> 💡 **`@Mock` vs `@InjectMocks`:**
> `@Mock` crea un objeto falso que simula el comportamiento que tú definas.
> `@InjectMocks` crea el objeto real e inyecta automáticamente los mocks en él.
> Así probamos `StudentServiceImpl` sin tocar la base de datos.

---

### `StudentControllerTest.java` — Prueba del controlador

```java
@WebMvcTest(StudentController.class)  // levanta SOLO la capa web
class StudentControllerTest {

    @Autowired
    private MockMvc mockMvc;    // simula peticiones HTTP sin servidor real

    @MockBean
    private StudentService service;  // mock del servicio (no se usa BD)

    @Test
    void testGetAllStudents() throws Exception {
        when(service.getAllStudents()).thenReturn(Arrays.asList(student));

        mockMvc.perform(get("/students"))        // GET /students
               .andExpect(status().isOk())       // espera HTTP 200
               .andExpect(content().json(...));  // verifica el JSON
    }
}
```

> 💡 **`@WebMvcTest` vs `@SpringBootTest`:**
> `@WebMvcTest` carga solo los beans de la capa web (controladores, filtros).
> No levanta JPA ni la base de datos. Es más rápido y apropiado para pruebas unitarias del controlador.

---

## ✏️ Paso 7 — Agrega tus propias pruebas

Para mejorar la cobertura, puedes agregar casos que el código existente no cubre.
Por ejemplo, el caso en que `getStudentById` no encuentra el estudiante:

Abre `src/test/java/com/example/bdget/service/StudentServiceImplTest.java` y agrega:

```java
@Test
void testGetStudentByIdNotFound() {
    when(repository.findById(99L)).thenReturn(Optional.empty());
    Optional<Student> result = service.getStudentById(99L);
    assertTrue(result.isEmpty());
}
```

O en `StudentControllerTest.java`, el caso de estudiante no encontrado (404):

```java
@Test
void testGetStudentByIdNotFound() throws Exception {
    when(service.getStudentById(99L)).thenReturn(Optional.empty());
    mockMvc.perform(get("/students/99"))
           .andExpect(status().isNotFound());
}
```

Después de agregar tests, vuelve a ejecutar:

```bash
mvn test
```

Y abre nuevamente `target/site/jacoco/index.html` para ver cómo mejoró la cobertura.

---

## 🔄 Paso 8 — Lee el pipeline de GitHub Actions

Abre `.github/workflows/main.yml` y analiza cada paso:

```yaml
name: CI — Build, Test y Push a DockerHub

on:
  push:
    branches: [main]      # se dispara al hacer push a main
  pull_request:
    branches: [main]      # también en pull requests

jobs:
  build-test-push:
    runs-on: ubuntu-latest

    # ── Servicio MySQL para tests de integración (opcional) ──
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: bdget_db
          MYSQL_USER: bdget_user
          MYSQL_PASSWORD: bdget_pass
        options: --health-cmd="mysqladmin ping ..." --health-retries=5

    steps:
      # 1. Descarga el código
      - uses: actions/checkout@v4

      # 2. Instala Java 17 con caché de Maven
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven          # ← guarda ~/.m2 entre ejecuciones (más rápido)

      # 3. Ejecuta los tests Y genera el reporte JaCoCo
      - name: Ejecutar tests
        run: mvn test
        env:
          SPRING_DATASOURCE_URL: jdbc:mysql://localhost:3306/bdget_db?...
          SPRING_DATASOURCE_USERNAME: bdget_user
          SPRING_DATASOURCE_PASSWORD: bdget_pass

      # 4. Genera el reporte explícitamente (por si acaso)
      - name: Generar reporte de cobertura
        run: mvn jacoco:report

      # 5. Sube target/site/jacoco/ como artefacto descargable
      - name: Subir reporte de cobertura
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/

      # 6-9. Compila el .jar, hace login en DockerHub y publica la imagen
      - run: mvn package -DskipTests
      - uses: docker/login-action@v3
      - run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/bdget-app:latest .
      - run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/bdget-app:latest
```

**Puntos clave para entender:**

| Paso | ¿Para qué sirve? |
|---|---|
| `actions/checkout@v4` | Descarga el código del repositorio en el runner |
| `actions/setup-java@v4` + `cache: maven` | Instala Java 17 y reutiliza dependencias descargadas |
| `mvn test` | Corre todos los tests + genera `jacoco.exec` |
| `mvn jacoco:report` | Convierte `jacoco.exec` → HTML en `target/site/jacoco/` |
| `upload-artifact@v4` | Guarda el HTML como artefacto descargable en GitHub |
| `mvn package -DskipTests` | Compila el `.jar` sin re-ejecutar tests |

---

## 🔑 Paso 9 — Configura los Secrets en GitHub

El pipeline necesita credenciales de DockerHub guardadas como Secrets en GitHub.

1. Ve a tu repositorio en GitHub
2. Click en **Settings** → **Secrets and variables** → **Actions**
3. Click en **"New repository secret"** y agrega:

| Nombre del Secret | Valor |
|---|---|
| `DOCKERHUB_USERNAME` | Tu nombre de usuario en DockerHub |
| `DOCKERHUB_TOKEN` | Token de acceso de DockerHub (no tu contraseña) |

**¿Cómo crear un token de DockerHub?**

```text
1. Ingresa a https://hub.docker.com
2. Click en tu usuario → Account Settings → Security
3. Click en "New Access Token"
4. Ponle un nombre (ej: "github-actions") y copia el token generado
5. Pégalo en GitHub como el secret DOCKERHUB_TOKEN
```

> ⚠️ Un token de acceso es más seguro que tu contraseña: puedes revocarlo sin cambiar tu contraseña.
> Nunca escribas credenciales directamente en el YAML del pipeline.

---

## 🚀 Paso 10 — Ejecuta el pipeline en GitHub Actions

1. Asegúrate de tener todos los cambios commiteados:

```bash
git add .
git commit -m "feat: agrego pruebas unitarias y pipeline con JaCoCo"
git push origin main
```

2. Ve a tu repositorio en GitHub → pestaña **Actions**
3. Verás el workflow **"CI — Build, Test y Push a DockerHub"** ejecutándose
4. Click en el nombre del workflow para ver el detalle de cada paso

### ¿Qué verás en la ejecución?

```text
✅ Checkout del repositorio          ~5s
✅ Configurar Java 17                ~15s
✅ Ejecutar tests                    ~60s  (12 tests, 0 fallos)
✅ Generar reporte de cobertura      ~5s
✅ Subir reporte de cobertura        ~5s   → aparece en "Artifacts"
✅ Build del proyecto                ~30s
✅ Login en DockerHub                ~5s
✅ Build de la imagen Docker         ~120s
✅ Push de la imagen a DockerHub     ~30s
```

---

## 📥 Paso 11 — Descarga y analiza el artefacto de cobertura

Una vez que el pipeline termine exitosamente:

1. Ve a la pestaña **Actions** en GitHub
2. Click en la ejecución del workflow
3. Baja hasta la sección **Artifacts**
4. Descarga **`jacoco-report`**
5. Descomprime el `.zip` y abre `index.html` en tu navegador

Verás exactamente el mismo reporte que generaste localmente, pero guardado permanentemente
en GitHub como evidencia de la ejecución del pipeline.

---

## 🛑 Solución de problemas frecuentes

### Los tests fallan con error de conexión a la base de datos

Las pruebas unitarias no necesitan MySQL — pero si falla el contexto de Spring, revisa:

```bash
# Ejecuta solo los tests unitarios (sin levantar contexto Spring completo)
mvn test -Dtest=StudentModelTest,StudentServiceImplTest
```

---

### `mvn test` falla con "compilation error"

Verifica que estás usando Java 17:

```bash
java -version
# Debe mostrar: openjdk version "17.x.x"

# Si tienes múltiples versiones de Java (macOS con SDKMAN o Homebrew):
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
mvn test
```

---

### El reporte JaCoCo no se genera

Si `target/site/jacoco/` no existe después de `mvn test`, fuerza la generación:

```bash
mvn jacoco:report
```

---

### El pipeline falla en "Push de la imagen a DockerHub"

Verifica que los secrets `DOCKERHUB_USERNAME` y `DOCKERHUB_TOKEN` estén configurados
correctamente en GitHub (Settings → Secrets → Actions).

---

### El pipeline falla en "Ejecutar tests" por MySQL

El pipeline usa un servicio MySQL de GitHub Actions. Si los tests unitarios
no necesitan base de datos, puedes omitir el bloque `services:` del YAML
o asegúrate de que la URL en `env:` apunte a `localhost:3306`.

---

## 📊 Resumen de comandos Maven

| Acción | Comando |
|---|---|
| Ejecutar todos los tests + cobertura | `mvn test` |
| Ejecutar solo una clase de test | `mvn test -Dtest=NombreClase` |
| Generar reporte JaCoCo manualmente | `mvn jacoco:report` |
| Compilar sin tests | `mvn package -DskipTests` |
| Limpiar archivos compilados | `mvn clean` |
| Todo desde cero | `mvn clean test` |
| Ver reporte local | `open target/site/jacoco/index.html` |

---

## ✅ Lista de verificación — entregables

Al terminar la actividad debes tener evidencia de cada punto:

- [ ] Captura de `mvn test` con **0 fallos** y el número de tests ejecutados
- [ ] Captura del reporte `index.html` de JaCoCo abierto en el navegador (porcentaje de cobertura visible)
- [ ] Captura del reporte de JaCoCo mostrando líneas cubiertas (verde) y no cubiertas (rojo) en al menos una clase
- [ ] Captura del pipeline en GitHub Actions con **todos los pasos en verde** ✅
- [ ] Captura de la sección **Artifacts** en GitHub Actions mostrando el artefacto `jacoco-report` descargable
- [ ] Captura del reporte de cobertura descargado desde el artefacto de GitHub Actions

---

*Guía del Estudiante — Asignatura DevOps | Actividad 2.3*


---

## 📎 Anexo — Uso de Podman en lugar de Docker

> Este anexo aplica si usas **Podman** como motor de contenedores en lugar de Docker Desktop.
> Podman es compatible con los mismos comandos e imágenes, pero requiere ajustes en el pipeline
> y en los comandos locales.

---

### 🖥️ Diferencias locales (tu máquina)

Podman es un reemplazo directo de Docker. Casi todos los comandos son idénticos:

| Docker | Podman equivalente |
|---|---|
| `docker build -t imagen .` | `podman build -t imagen .` |
| `docker run -p 8080:8080 imagen` | `podman run -p 8080:8080 imagen` |
| `docker push usuario/imagen` | `podman push usuario/imagen` |
| `docker login` | `podman login docker.io` |
| `docker compose up` | `podman compose up` |

---

### 🔑 Paso 9 (Podman) — Los secrets siguen siendo de DockerHub

Podman puede publicar imágenes en **DockerHub** exactamente igual que Docker.
Los secrets de GitHub no cambian:

| Nombre del Secret | Valor |
|---|---|
| `DOCKERHUB_USERNAME` | Tu nombre de usuario en DockerHub |
| `DOCKERHUB_TOKEN` | Token de acceso de DockerHub |

> La diferencia es **solo en tu máquina local** — en el pipeline de GitHub Actions
> el runner usa Docker por defecto, así que el YAML **no necesita cambiar**.

---

### 🔄 Paso 8 (Podman) — Pipeline sin cambios

El runner de GitHub Actions (`ubuntu-latest`) tiene Docker instalado de forma nativa.
El `main.yml` original funciona **tal cual** aunque tú uses Podman localmente:

```yaml
# Estos pasos funcionan sin modificación en el runner de GitHub Actions:
- run: mvn package -DskipTests
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
- run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/bdget-app:latest .
- run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/bdget-app:latest
```

---

### 🖥️ Comandos locales con Podman

Para hacer pruebas en tu máquina antes del push, usa Podman:

```bash
# 1. Compilar el .jar
mvn package -DskipTests

# 2. Construir la imagen con Podman
podman build -t bdget-app:latest .

# 3. Probar la imagen localmente
podman run -p 8080:8080 bdget-app:latest

# 4. Login en DockerHub desde Podman
podman login docker.io

# 5. Etiquetar y publicar la imagen
podman tag bdget-app:latest docker.io/TU_USUARIO/bdget-app:latest
podman push docker.io/TU_USUARIO/bdget-app:latest
```

---

### 🗄️ Levantar MySQL con Podman (desarrollo local)

Si necesitas MySQL local para pruebas de integración:

```bash
# Levantar MySQL con Podman
podman run -d \
  --name mysql-bdget \
  -e MYSQL_DATABASE=bdget_db \
  -e MYSQL_USER=bdget_user \
  -e MYSQL_PASSWORD=bdget_pass \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:8.0

# Verificar que está corriendo
podman ps

# Detener el contenedor
podman stop mysql-bdget
```

Con `podman compose` (si lo tienes instalado):

```bash
podman compose up -d
podman compose down
```

---

### 🛑 Problemas frecuentes con Podman

#### `podman: command not found`
```bash
# macOS con Homebrew
brew install podman
podman machine init
podman machine start
```

#### `Error: short-name resolution` al hacer pull de imágenes
```bash
# Especifica el registro completo
podman pull docker.io/library/mysql:8.0
```

#### `podman compose` no está disponible
```bash
# Instalar podman-compose
brew install podman-compose
# o con pip
pip3 install podman-compose
```

#### El pipeline falla en el paso de Docker aunque uses Podman localmente
El runner de GitHub Actions usa Docker, no Podman. Si el pipeline falla, el problema
no es Podman — revisa los secrets `DOCKERHUB_USERNAME` y `DOCKERHUB_TOKEN` en
**Settings → Secrets and variables → Actions**.

---

*Anexo Podman — Guía del Estudiante | Actividad 2.3*

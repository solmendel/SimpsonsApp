# SimpsonsApp — Análisis de errores

**Alumno:** Sol Mendelsohn  
**Materia:** Desarrollo de Aplicaciones I — UADE FAIN  
**Parcial:** 2do Parcial · 1er Cuatrimestre 2026


---
ERRORES

## Error 1 — `Episode.kt` · Líneas 13-15

**Categoría:** Sintaxis inválida — no compila

```kotlin
init {
    return Episode; //NO BORRAR
}
```
**¿Qué está mal?**  
El bloque `init` existe únicamente dentro de clases. Acá está suelto a nivel de archivo, lo cual es directamente inválido en Kotlin y hace que la app no compile. Encima, aunque estuviera dentro de una clase, los bloques `init` no retornan valores — eso tampoco tiene sentido.

**¿Cómo se corrige?**  
Eliminar las líneas 13, 14 y 15 completas. El archivo debería terminar en el cierre del paréntesis de la `data class`.

---

## Error 2 — `EpisodeRemoteMediator.kt` · Líneas 105-110

**Categoría:** Definición duplicada — no compila

```kotlin
interface SimpsonsApi {
    @GET("https://thesimpsonsapi.com/api/episodes")
    suspend fun getEpisodes(
        @Query("page") page: Int
    ): EpisodesResponse
}
```

**¿Qué está mal?**  
Hay dos problemas en uno. Primero, la interfaz `SimpsonsApi` está definida dentro del archivo del `EpisodeRemoteMediator`, que no es su lugar — debería vivir en su propio archivo separado. Segundo, la URL completa está hardcodeada directamente en el `@GET`, cuando la práctica correcta es poner solo el path relativo (`"episodes"`) y configurar la base URL en Retrofit, que es quien la centraliza.

**¿Cómo se corrige?**  
Mover `SimpsonsApi` a su propio archivo `SimpsonsApi.kt` dentro del paquete `data/remote`, y cambiar el `@GET` para que use solo el path: `@GET("episodes")`. La base URL va en `DataModule.kt`.

---

## Error 3 — `DataModule.kt` · Líneas 34-38

**Categoría:** Configuración crítica de Retrofit — crash en runtime

```kotlin
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

**¿Qué está mal?**  
Retrofit se construye sin `.baseUrl(...)`. Esto no es un error silencioso — Retrofit lanza una `IllegalStateException: Base URL required` en el momento en que Hilt intenta inicializar las dependencias, es decir, ni bien abre la app. Lo confirmamos en el Logcat: la app crasheaba inmediatamente al abrirse.

**¿Cómo se corrige?**  
Agregar `.baseUrl("https://thesimpsonsapi.com/api/")` antes del `.addConverterFactory(...)`.

---

## Error 4 — `EpisodeRepository.kt` · Línea 8

**Categoría:** Convención de nomenclatura — snake_case en Kotlin

```kotlin
fun get_episodes(): Flow<PagingData<Episode>>
```

**¿Qué está mal?**  
Kotlin usa camelCase para todo: variables, funciones, parámetros. El nombre `get_episodes()` sigue la convención de Python o C, que no corresponde acá. Además genera una inconsistencia grave: la implementación (`EpisodeRepositoryImpl`) declara `getEpisodes()` con el nombre correcto, pero la interfaz dice `get_episodes()`, rompiendo el contrato entre ambas.

**¿Cómo se corrige?**  
Renombrar a `getEpisodes()` tanto en la interfaz como en el UseCase que la llama (`GetEpisodesUseCase.kt` línea 13).

---

## Error 5 — `AppNavigation.kt` · Línea 10

**Categoría:** Import inválido

```kotlin
import androidx.compose.*
```

**¿Qué está mal?**  
`androidx.compose` no existe como paquete raíz importable con wildcard. Es un import que no resuelve nada y genera ambigüedad innecesaria. Todo lo que la app necesita de Compose ya está importado correctamente en las líneas anteriores del mismo archivo.

**¿Cómo se corrige?**  
Eliminar esa línea directamente.

---

## Error 6 — `MainScreenViewModel.kt` · Línea 12

**Categoría:** Arquitectura — ViewModel sin anotaciones de Hilt

```kotlin
class MainScreenViewModel(dataRepository: DataRepository) : ViewModel() {
```

**¿Qué está mal?**  
Toda la app usa Hilt como sistema de inyección de dependencias. Todos los ViewModels del proyecto tienen `@HiltViewModel` y `@Inject constructor`. Este ViewModel recibe una dependencia por constructor pero no tiene ninguna de esas anotaciones, lo que hace que Hilt no pueda instanciarlo correctamente cuando se llama a `hiltViewModel()` desde la UI.

**¿Cómo se corrige?**

```kotlin
@HiltViewModel
class MainScreenViewModel @Inject constructor(
    dataRepository: DataRepository
) : ViewModel() {
```

---

## Error 7 — `MainScreen.kt` · Líneas 51-53

**Categoría:** Arquitectura MVVM — lógica de negocio dentro de un Composable

```kotlin
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}
```

**¿Qué está mal?**  
Esta lógica está directamente en el cuerpo del Composable, fuera de cualquier efecto controlado. En MVVM, los Composables solo renderizan — no toman decisiones sobre cuándo llamar al ViewModel ni ejecutan side effects libremente. Además, ejecutar código con efectos secundarios en el cuerpo de un Composable puede provocar que se llame en cada recomposición, generando bucles infinitos.

**¿Cómo se corrige?**  
Envolver la lógica en un `LaunchedEffect` que controle cuándo se ejecuta:

```kotlin
LaunchedEffect(episodes.loadState.refresh) {
    if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
        viewModel.refreshSeasons()
    }
}
```

---

## Error 8 — `EpisodeEntity.kt` · Línea 7

**Categoría:** Room — Entity definida como `class` en lugar de `data class`

```kotlin
class EpisodeEntity(
```

**¿Qué está mal?**  
Las entidades de Room deben ser `data class`. Al usar `class` común, Room no puede generar correctamente las comparaciones de igualdad entre objetos ni el método `copy()`, que son fundamentales para que el sistema detecte cambios en la base de datos y actualice la UI reactivamente a través de `Flow`.

**¿Cómo se corrige?**

```kotlin
data class EpisodeEntity(
```

---

## Error 9 — `MainScreen.kt` · Línea 119

**Categoría:** UI/UX — lista principal con scroll horizontal

```kotlin
LazyRow(
    state = listState,
    modifier = Modifier.fillMaxSize()
) {
```

**¿Qué está mal?**  
`LazyRow` genera una lista con scroll horizontal. La lista principal de episodios, que muestra título, sinopsis, fecha y temporada, debería scrollearse verticalmente. Esto lo pudimos confirmar visualmente al correr la app: las tarjetas aparecían una al lado de la otra en vez de apilarse hacia abajo. `LazyRow` es apropiado para carruseles o filas de elementos pequeños, no para el contenido principal de una pantalla.

**¿Cómo se corrige?**

```kotlin
LazyColumn(
    state = listState,
    modifier = Modifier.fillMaxSize()
) {
```

---

## Error 10 — `DetailScreen.kt` · Línea 59

**Categoría:** Seguridad — uso del operador `!!` con riesgo de NullPointerException

```kotlin
EpisodeCard(
    episode = episode!!,
```

**¿Qué está mal?**  
El operador `!!` fuerza un cast non-null y lanza `NullPointerException` si el valor es null. El problema es que justo arriba, en la línea 51, ya se hizo la verificación `if (episode != null)`. En Kotlin, el patrón correcto para este caso es usar `?.let` que captura el valor en una variable local inmutable y garantiza que no puede ser null dentro del bloque, sin importar condiciones de carrera.

**¿Cómo se corrige?**

```kotlin
episode?.let { ep ->
    EpisodeCard(
        episode = ep,
        onClick = { },
        isDetailMode = true
    )
}
```

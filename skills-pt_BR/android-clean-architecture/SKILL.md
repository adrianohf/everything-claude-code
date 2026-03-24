---
name: android-clean-architecture
description: Padrões de Clean Architecture para projetos Android e Kotlin Multiplatform — estrutura de módulos, regras de dependência, UseCases, Repositories e padrões de camada de dados.
origin: ECC
---

# Android Clean Architecture

Padrões de Clean Architecture para projetos Android e KMP. Abrange limites de módulo, inversão de dependência, padrões UseCase/Repository e design de camada de dados com Room, SQLDelight e Ktor.

## Quando Ativar

- Estruturação de módulos de projeto Android ou KMP
- Implementação de UseCases, Repositories ou DataSources
- Projetando o fluxo de dados entre camadas (domain, data, presentation)
- Configurando injeção de dependência com Koin ou Hilt
- Trabalhando com Room, SQLDelight ou Ktor em uma arquitetura em camadas

## Estrutura de Módulos

### Layout Recomendado

```
project/
├── app/                  # Ponto de entrada Android, fiação de DI, classe Application
├── core/                 # Utilitários compartilhados, classes base, tipos de erro
├── domain/               # UseCases, modelos de domínio, interfaces de repositório (Kotlin puro)
├── data/                 # Implementações de repositório, DataSources, DB, rede
├── presentation/         # Telas, ViewModels, modelos de UI, navegação
├── design-system/        # Componentes Compose reutilizáveis, tema, tipografia
└── feature/              # Módulos de funcionalidade (opcional, para projetos maiores)
    ├── auth/
    ├── settings/
    └── profile/
```

### Regras de Dependência

```
app → presentation, domain, data, core
presentation → domain, design-system, core
data → domain, core
domain → core (ou nenhuma dependência)
core → (nada)
```

**Crítico**: `domain` NUNCA deve depender de `data`, `presentation` ou qualquer framework. Ele contém apenas Kotlin puro.

## Camada de Domínio (Domain Layer)

### Padrão UseCase

Cada UseCase representa uma operação de negócio. Use `operator fun invoke` para locais de chamada limpos:

```kotlin
class GetItemsByCategoryUseCase(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(category: String): Result<List<Item>> {
        return repository.getItemsByCategory(category)
    }
}

// UseCase baseado em Flow para streams reativas
class ObserveUserProgressUseCase(
    private val repository: UserRepository
) {
    operator fun invoke(userId: String): Flow<UserProgress> {
        return repository.observeProgress(userId)
    }
}
```

### Modelos de Domínio

Modelos de domínio são classes de dados Kotlin simples — sem anotações de framework:

```kotlin
data class Item(
    val id: String,
    val title: String,
    val description: String,
    val tags: List<String>,
    val status: Status,
    val category: String
)

enum class Status { DRAFT, ACTIVE, ARCHIVED }
```

### Interfaces de Repositório

Definidas no domínio, implementadas nos dados:

```kotlin
interface ItemRepository {
    suspend fun getItemsByCategory(category: String): Result<List<Item>>
    suspend fun saveItem(item: Item): Result<Unit>
    fun observeItems(): Flow<List<Item>>
}
```

## Camada de Dados (Data Layer)

### Implementação do Repositório

Coordena entre fontes de dados locais e remotas:

```kotlin
class ItemRepositoryImpl(
    private val localDataSource: ItemLocalDataSource,
    private val remoteDataSource: ItemRemoteDataSource
) : ItemRepository {

    override suspend fun getItemsByCategory(category: String): Result<List<Item>> {
        return runCatching {
            val remote = remoteDataSource.fetchItems(category)
            localDataSource.insertItems(remote.map { it.toEntity() })
            localDataSource.getItemsByCategory(category).map { it.toDomain() }
        }
    }

    override suspend fun saveItem(item: Item): Result<Unit> {
        return runCatching {
            localDataSource.insertItems(listOf(item.toEntity()))
        }
    }

    override fun observeItems(): Flow<List<Item>> {
        return localDataSource.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

### Padrão Mapper

Mantenha os mappers como funções de extensão próximas aos modelos de dados:

```kotlin
// Na camada de dados
fun ItemEntity.toDomain() = Item(
    id = id,
    title = title,
    description = description,
    tags = tags.split("|"),
    status = Status.valueOf(status),
    category = category
)

fun ItemDto.toEntity() = ItemEntity(
    id = id,
    title = title,
    description = description,
    tags = tags.joinToString("|"),
    status = status,
    category = category
)
```

### Banco de Dados Room (Android)

```kotlin
@Entity(tableName = "items")
data class ItemEntity(
    @PrimaryKey val id: String,
    val title: String,
    val description: String,
    val tags: String,
    val status: String,
    val category: String
)

@Dao
interface ItemDao {
    @Query("SELECT * FROM items WHERE category = :category")
    suspend fun getByCategory(category: String): List<ItemEntity>

    @Upsert
    suspend fun upsert(items: List<ItemEntity>)

    @Query("SELECT * FROM items")
    fun observeAll(): Flow<List<ItemEntity>>
}
```

### SQLDelight (KMP)

```sql
-- Item.sq
CREATE TABLE ItemEntity (
    id TEXT NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    tags TEXT NOT NULL,
    status TEXT NOT NULL,
    category TEXT NOT NULL
);

getByCategory:
SELECT * FROM ItemEntity WHERE category = ?;

upsert:
INSERT OR REPLACE INTO ItemEntity (id, title, description, tags, status, category)
VALUES (?, ?, ?, ?, ?, ?);

observeAll:
SELECT * FROM ItemEntity;
```

### Cliente de Rede Ktor (KMP)

```kotlin
class ItemRemoteDataSource(private val client: HttpClient) {

    suspend fun fetchItems(category: String): List<ItemDto> {
        return client.get("api/items") {
            parameter("category", category)
        }.body()
    }
}

// Configuração do HttpClient com negociação de conteúdo
val httpClient = HttpClient {
    install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    install(Logging) { level = LogLevel.HEADERS }
    defaultRequest { url("https://api.example.com/") }
}
```

## Injeção de Dependência

### Koin (Amigável a KMP)

```kotlin
// Módulo de domínio
val domainModule = module {
    factory { GetItemsByCategoryUseCase(get()) }
    factory { ObserveUserProgressUseCase(get()) }
}

// Módulo de dados
val dataModule = module {
    single<ItemRepository> { ItemRepositoryImpl(get(), get()) }
    single { ItemLocalDataSource(get()) }
    single { ItemRemoteDataSource(get()) }
}

// Módulo de apresentação
val presentationModule = module {
    viewModelOf(::ItemListViewModel)
    viewModelOf(::DashboardViewModel)
}
```

### Hilt (Apenas Android)

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindItemRepository(impl: ItemRepositoryImpl): ItemRepository
}

@HiltViewModel
class ItemListViewModel @Inject constructor(
    private val getItems: GetItemsByCategoryUseCase
) : ViewModel()
```

## Tratamento de Erros

### Padrão Result/Try

Use `Result<T>` ou um tipo selado personalizado para propagação de erros:

```kotlin
sealed interface Try<out T> {
    data class Success<T>(val value: T) : Try<T>
    data class Failure(val error: AppError) : Try<Nothing>
}

sealed interface AppError {
    data class Network(val message: String) : AppError
    data class Database(val message: String) : AppError
    data object Unauthorized : AppError
}

// Na ViewModel — mapear para o estado da UI
viewModelScope.launch {
    when (val result = getItems(category)) {
        is Try.Success -> _state.update { it.copy(items = result.value, isLoading = false) }
        is Try.Failure -> _state.update { it.copy(error = result.error.toMessage(), isLoading = false) }
    }
}
```

## Plugins de Convenção (Gradle)

Para projetos KMP, use plugins de convenção para reduzir a duplicação de arquivos de build:

```kotlin
// build-logic/src/main/kotlin/kmp-library.gradle.kts
plugins {
    id("org.jetbrains.kotlin.multiplatform")
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()
    sourceSets {
        commonMain.dependencies { /* dependências compartilhadas */ }
        commonTest.dependencies { implementation(kotlin("test")) }
    }
}
```

Aplicar nos módulos:

```kotlin
// domain/build.gradle.kts
plugins { id("kmp-library") }
```

## Anti-padrões a Evitar

- Importar classes de framework Android em `domain` — mantenha-o em Kotlin puro
- Expor entidades de banco de dados ou DTOs para a camada de UI — sempre mapeie para modelos de domínio
- Colocar lógica de negócio em ViewModels — extraia para UseCases
- Usar `GlobalScope` ou corrotinas não estruturadas — use `viewModelScope` ou concorrência estruturada
- Implementações de repositório inchadas — divida em DataSources focados
- Dependências de módulo circulares — se A depende de B, B não deve depender de A

## Referências

Veja a skill: `compose-multiplatform-patterns` para padrões de UI.
Veja a skill: `kotlin-coroutines-flows` para padrões assíncronos.

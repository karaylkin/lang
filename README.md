# Языки написания спецификаций
## Тема ВКР:  Разработка облачного хранилища файлов c поиском на основе интеграции нейромоделей
## Диаграмма вариантов использования (use case)
```
@startuml
left to right direction

actor "Пользователь" as User
actor "Администратор" as Admin

rectangle "Облачное хранилище" {
  together {
    usecase "Регистрация / Вход" as UC1
    usecase "Загрузка файла" as UC2
    usecase "Семантический поиск" as UC3
    usecase "Просмотр / Скачивание" as UC4
    usecase "Удаление файла" as UC5
    usecase "Мониторинг системы" as UC6
    usecase "Управление пользователями" as UC7
  }
  together {
    usecase "Вычисление эмбеддинга" as UC8
    usecase "Поиск в БД" as UC9
    usecase "Сохранение вектора в БД" as UC10
  }
}

User --> UC1
User --> UC2
User --> UC3
User --> UC4
User --> UC5

Admin --> UC6
Admin --> UC7

UC2 .> UC8 : trigger
UC3 .> UC9 : include
UC8 .> UC10 : include

@enduml
```
<img width="483" height="826" alt="image" src="https://github.com/user-attachments/assets/808e07f6-e352-44c5-ab24-dcc41b2454b2" />

## Диаграмма классов (classes)
```
@startuml CloudStoragePhotos

skinparam classAttributeIconSize 0
skinparam class {
  BackgroundColor White
  BorderColor #555
  ArrowColor #555
  FontSize 13
}

class User {
  - id: UUID
  - email: String
  - passwordHash: String
  + register(): void
  + login(): Token
  + uploadPhoto(): Photo
}

class Photo {
  - id: UUID
  - filename: String
  - mimeType: String
  - storageUrl: String
  - ownerId: UUID
  - takenAt: LocalDateTime
  - latitude: Double?
  - longitude: Double?
  - createdAt: Instant
  + upload(): URL
  + delete(): void
  + getEmbedding(): EmbeddingVector
}

class EmbeddingVector {
  - vector: float[]
  - modelId: String
  - photoId: UUID
  - dimensions: Int
  + compute(): float[]
  + save(): void
  + similarity(other: EmbeddingVector): float
}

class NeuralModel {
  - modelId: String
  - type: ModelType
  - endpoint: URL
  + encodeImage(bytes: byte[]): float[]
  + encodeText(text: String): float[]
  + isAvailable(): Boolean
}

class SearchQuery {
  - text: String?
  - dateFrom: LocalDateTime?
  - dateTo: LocalDateTime?
  - latitude: Double?
  - longitude: Double?
  - radiusKm: Double?
}

class SearchEngine {
  - threshold: Float
  - topK: Int
  + search(query: SearchQuery): List~Photo~
  + indexPhoto(photo: Photo): void
  + rerank(results: List~Photo~): List~Photo~
}

class VectorDB {
  - collection: String
  - dimensions: Int
  - host: String
  + upsert(id: UUID, vector: float[], payload: Map): void
  + query(vector: float[], filters: Map, k: Int): List~UUID~
  + delete(id: UUID): void
  + count(): Long
}

class StorageService {
  - bucketUrl: String
  - provider: String
  + store(photo: Photo): URL
  + remove(photoId: UUID): void
  + getPresignedUrl(photoId: UUID): URL
  + exists(photoId: UUID): Boolean
}

class ImageProcessor {
  - maxSizeMb: Int
  - allowedTypes: List~String~
  + extractExif(bytes: byte[]): ExifData
  + resize(bytes: byte[]): byte[]
  + normalize(bytes: byte[]): byte[]
  + validate(bytes: byte[]): Boolean
}

class AuthService {
  - jwtSecret: String
  - tokenTtl: Int
  + generateToken(user: User): JWT
  + validate(token: JWT): Boolean
  + refresh(token: JWT): JWT
}

class ExifData {
  - takenAt: LocalDateTime?
  - latitude: Double?
  - longitude: Double?
  - cameraMake: String?
  - cameraModel: String?
}

enum ModelType {
  CLIP
  VIT
  CUSTOM
}

' Relationships
User "1" o-- "*" Photo : владеет
Photo "1" --> "1" EmbeddingVector
ImageProcessor --> ExifData : извлекает
ImageProcessor ..> Photo : обрабатывает
ImageProcessor ..> NeuralModel : использует
EmbeddingVector ..> NeuralModel : вычисляется через
EmbeddingVector ..> VectorDB : хранится в
SearchQuery --> SearchEngine : передаётся в
SearchEngine ..> VectorDB : запрашивает
SearchEngine ..> NeuralModel : кодирует через
Photo ..> StorageService : хранится в
User ..> AuthService : аутентифицируется через
NeuralModel --> ModelType

@enduml
```
<img width="831" height="933" alt="image" src="https://github.com/user-attachments/assets/78700ef7-17aa-49b2-bb3f-0dc4af91da16" />

## Диаграмма последовательности (sequence)

```
@startuml PhotoSystem

skinparam style strictuml
skinparam monochrome true
skinparam shadowing false
skinparam sequence {
  ArrowColor #000
  LifeLineBorderColor #000
  LifeLineBackgroundColor #fff
  ParticipantBorderColor #000
  ParticipantBackgroundColor #fff
  ParticipantFontSize 13
  ArrowFontSize 12
}
skinparam ActorBorderColor #000
skinparam ActorBackgroundColor #fff

actor "User" as U
participant "API" as A
participant "Обработчик изображений" as IP
participant "Нейромодель" as NM
database "Хранилище S3" as S
database "VectorDB" as VDB
database "PostgreSQL" as DB

== Загрузка фото ==

U -> A : Загрузить фото
activate A

A -> IP : Проверить файл
activate IP
IP --> A : Данные EXIF
A -> IP : Изменить размер и нормализовать
IP --> A : Готовые байты
deactivate IP

A -> NM : Закодировать изображение
activate NM
NM --> A : Вектор
deactivate NM

A -> S : Сохранить оригинал
activate S
S --> A : Ссылка на файл
deactivate S

A -> VDB : Добавить вектор с метаданными
note right
  Дата съёмки
  Координаты
end note
activate VDB
VDB --> A : Успешно
deactivate VDB

A -> DB : Записать метаданные
note right
  Имя, ссылка на S3,
  владелец, дата,
  координаты
end note

A --> U : Фото загружено
deactivate A

== Поиск фото ==

U -> A : Найти фото
activate A

A -> NM : Закодировать запрос
activate NM
NM --> A : Вектор запроса
deactivate NM

A -> VDB : Поиск с фильтрами
note right
  Дата съёмки
  Геозона
end note
activate VDB
VDB --> A : UUID найденных фото
deactivate VDB

A -> DB : Получить метаданные
activate DB
DB --> A : Данные фото
deactivate DB

A -> S : Сгенерировать ссылки
activate S
S --> A : Пресайнед URL
deactivate S

A --> U : Результаты поиска
deactivate A

@enduml
```
<img width="831" height="802" alt="image" src="https://github.com/user-attachments/assets/575a5af3-abec-48e3-b57f-b34f28f8b486" />

## Диаграмма диаграмма состояний (state)
```
@startuml PhotoSystemStates

skinparam monochrome true
skinparam shadowing false
skinparam state {
  BorderColor #000
  BackgroundColor #fff
  FontSize 13
  ArrowColor #000
}

[*] --> Неавторизован

Неавторизован --> Авторизован : вход в систему
Авторизован --> Неавторизован : выход из системы

state "Загрузка фото" as Upload {
  [*] --> ВыборФото
  ВыборФото --> Проверка : файл выбран
  Проверка --> Обработка : файл валиден
  Проверка --> Ошибка : файл невалиден
  Ошибка --> ВыборФото : повторить
  Обработка --> ВычислениеВектора : EXIF извлечён\nразмер изменён
  ВычислениеВектора --> СохранениеФайла : вектор получен
  СохранениеФайла --> СохранениеВектора : файл сохранён в S3
  СохранениеВектора --> СохранениеМетаданных : вектор сохранён в VectorDB
  СохранениеМетаданных --> [*] : метаданные записаны
}

state "Поиск фото" as Search {
  [*] --> ВводЗапроса
  ВводЗапроса --> УстановкаФильтров : запрос введён
  УстановкаФильтров --> КодированиеЗапроса : фильтры применены
  КодированиеЗапроса --> ПоискВекторов : вектор получен
  ПоискВекторов --> ПолучениеМетаданных : UUID найдены
  ПоискВекторов --> НетРезультатов : совпадений нет
  НетРезультатов --> ВводЗапроса : изменить запрос
  ПолучениеМетаданных --> ГенерацияСсылок : метаданные получены
  ГенерацияСсылок --> ПросмотрРезультатов : ссылки готовы
  ПросмотрРезультатов --> [*] : выйти
}

Авторизован --> Upload : загрузить фото
Авторизован --> Search : найти фото
Upload --> Авторизован : завершено
Search --> Авторизован : завершено

@enduml

```
<img width="831" height="1163" alt="image" src="https://github.com/user-attachments/assets/6536318b-10c3-4a58-b852-a996a8e52616" />

## Диаграмма диаграмма деятельности (activity)
```
@startuml FullLifecycle

skinparam monochrome true
skinparam shadowing false
skinparam activity {
  BorderColor #000
  BackgroundColor #fff
  FontSize 13
  ArrowColor #000
  BarColor #000
}

|Пользователь|
start

:Открыть приложение;

|Система|
:Проверить токен;

if (токен валиден?) then (нет)
  |Пользователь|
  :Регистрация / Вход;
  |Система|
  :Создать JWT токен;
endif

|Пользователь|
:Главный экран;

fork
  :Загрузить фото;
  |Система|
  :Проверить файл;
  if (файл валиден?) then (нет)
    |Пользователь|
    :Сообщение об ошибке;
    stop
  endif
  |Система|
  :Извлечь EXIF;
  :Изменить размер\nи нормализовать;
  :Вычислить вектор\nчерез нейромодель;
  fork
    |Хранилища|
    :Сохранить файл в S3;
  fork again
    :Сохранить вектор\nв VectorDB;
  fork again
    :Записать метаданные\nв PostgreSQL;
  end fork
  |Пользователь|
  :Фото в галерее;

fork again
  :Найти фото;
  |Пользователь|
  :Ввести текстовый запрос;
  :Указать фильтры\n(дата, геозона);
  |Система|
  :Закодировать запрос\nчерез нейромодель;
  :Поиск в VectorDB\nс фильтрами;
  if (результаты найдены?) then (нет)
    |Пользователь|
    :Нет результатов;
    stop
  endif
  |Система|
  :Получить метаданные\nиз PostgreSQL;
  :Сгенерировать\nпресайнед URL;
  |Пользователь|
  :Просмотр результатов;
end fork

stop

@enduml
```
<img width="831" height="792" alt="image" src="https://github.com/user-attachments/assets/44a1b67e-27cd-4658-9127-6b8ba6c2a6f6" />
<img width="1359" height="1192" alt="image" src="https://github.com/user-attachments/assets/ebcc5335-fbbf-4a2e-b259-a5180faf52ef" />



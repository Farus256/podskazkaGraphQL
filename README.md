# 📚 GraphQL Шпаргалка для проекта AbsoluteCinema

## 🎯 Что такое GraphQL простыми словами?

**GraphQL** — это язык запросов для API и среда выполнения для выполнения этих запросов.

### Простая аналогия:
- **REST API** = меню в ресторане: вы заказываете готовое блюдо (например, "борщ"), и получаете всё, что входит в это блюдо, даже если вам нужна только сметана
- **GraphQL** = шведский стол: вы сами выбираете, что именно вам нужно, и получаете только это

### Основные преимущества:
1. **Запрашиваешь только то, что нужно** — не получаешь лишних данных
2. **Один эндпоинт** — вместо множества REST endpoints (`/api/movies`, `/api/movies/1`, `/api/movies/1/actors`) один `/graphql`
3. **Сильная типизация** — сервер точно знает, какие данные может вернуть
4. **Интроспекция** — можно узнать, какие запросы доступны (автодополнение в IDE)

### Основные понятия:
- **Query (Запрос)** — получение данных (аналог GET в REST)
- **Mutation (Мутация)** — изменение данных (аналог POST/PUT/DELETE в REST)
- **Subscription (Подписка)** — получение данных в реальном времени (WebSocket)
- **Schema (Схема)** — описание всех доступных данных и операций
- **Type (Тип)** — описание структуры данных

---

## 🏗️ Архитектура GraphQL в проекте

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   Frontend      │         │   GraphQL        │         │   Backend       │
│   (React)       │────────▶│   Server         │────────▶│   (C# .NET)     │
│   Apollo Client │         │   (HotChocolate) │         │   Services      │
└─────────────────┘         └──────────────────┘         └─────────────────┘
```

**Поток данных:**
1. React компонент создает GraphQL запрос
2. Apollo Client отправляет запрос на `/graphql` endpoint
3. HotChocolate обрабатывает запрос
4. Вызывается соответствующий resolver (метод в Query классе)
5. Resolver использует сервисы для получения данных из БД
6. Данные возвращаются клиенту в формате, который он запросил

---

## 🔧 Backend: Как работает GraphQL на сервере

### Используемые технологии:
- **HotChocolate** — GraphQL сервер для .NET
- **HotChocolate.Data** — расширения для фильтрации, сортировки, проекций
- **HotChocolate.Types** — типизация GraphQL схемы

### Структура файлов:

```
AbsoluteCinema.WebAPI/
├── GraphQL/
│   ├── GraphQLSchema.cs          # Настройка GraphQL сервера
│   ├── Queries/
│   │   └── MovieQuery.cs         # Запросы (Query)
│   ├── Mutations/                # Мутации (пока пусто)
│   ├── Types/
│   │   ├── MovieType.cs          # Тип Movie для GraphQL
│   │   └── MovieResponseType.cs  # Тип ответа (не используется)
│   └── Middleware/
│       └── RequestLoggingMiddleware.cs  # Логирование запросов
```

---

## 📝 Код Backend с объяснениями

### 1. Регистрация GraphQL в Program.cs

```csharp
// AbsoluteCinema.WebAPI/Program.cs

using HotChocolate.AspNetCore;
using AbsoluteCinema.WebAPI.GraphQL;

// ... другие using

var builder = WebApplication.CreateBuilder(args);

// Добавляем GraphQL схему в DI контейнер
builder.Services.AddGraphQLSchema();

// ... остальная настройка

var app = builder.Build();

// ... middleware настройка

// Маппинг GraphQL endpoint
// По умолчанию доступен на: /graphql
// Также доступен GraphQL Playground на: /graphql/ui (в development)
app.MapGraphQL();

app.Run();
```

**Что происходит:**
- `AddGraphQLSchema()` регистрирует GraphQL сервер в DI контейнере
- `MapGraphQL()` создает endpoint `/graphql` для обработки запросов
- В development режиме также доступен GraphQL Playground (интерактивная IDE) на `/graphql/ui`

---

### 2. Настройка GraphQL Schema

```csharp
// AbsoluteCinema.WebAPI/GraphQL/GraphQLSchema.cs

using Microsoft.Extensions.DependencyInjection;
using AbsoluteCinema.WebAPI.GraphQL.Middleware;
using AbsoluteCinema.WebAPI.GraphQL.Queries;
using AbsoluteCinema.WebAPI.GraphQL.Types;

namespace AbsoluteCinema.WebAPI.GraphQL
{
    public static class GraphQLSchema
    {
        public static IServiceCollection AddGraphQLSchema(this IServiceCollection services)
        {
            services
                .AddGraphQLServer()                    // Создаем GraphQL сервер
                .AddQueryType<MovieQuery>()            // Регистрируем Query тип (запросы)
                .AddType<MovieType>()                  // Регистрируем тип Movie
                .AddFiltering()                        // Включаем фильтрацию (where)
                .AddSorting()                          // Включаем сортировку (orderBy)
                .AddProjections()                       // Включаем проекции (select)
                .UseRequest<RequestLoggingMiddleware>(); // Добавляем middleware для логирования

            return services;
        }
    }
}
```

**Что происходит:**
- `AddGraphQLServer()` — создает GraphQL сервер
- `AddQueryType<MovieQuery>()` — регистрирует класс с запросами
- `AddType<MovieType>()` — регистрирует кастомный тип данных
- `AddFiltering()` — позволяет фильтровать данные в запросе
- `AddSorting()` — позволяет сортировать данные
- `AddProjections()` — оптимизирует запросы к БД (загружает только нужные поля)
- `UseRequest<RequestLoggingMiddleware>()` — добавляет middleware для логирования

---

### 3. Определение Query (Запросов)

```csharp
// AbsoluteCinema.WebAPI/GraphQL/Queries/MovieQuery.cs

using AbsoluteCinema.Application.Contracts;
using HotChocolate.Types;
using Microsoft.Extensions.Logging;

namespace AbsoluteCinema.WebAPI.GraphQL.Queries
{
    public class MovieQuery
    {
        // Запрос для получения всех фильмов
        [GraphQLName("movies")]  // Имя в GraphQL схеме
        [GraphQLDescription("Gets all movies with included related data")]  // Описание
        public async Task<IEnumerable<MovieDto>> GetMovies(
            [Service] IMovieService movieService,  // Dependency Injection
            [Service] ILogger<MovieQuery> logger)
        {
            try
            {
                // Вызываем сервис для получения данных
                return await movieService.GetAllMoviesWithIncludeAsync();
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error while getting movies");
                throw;
            }
        }

        // Запрос для получения фильма по ID
        [GraphQLName("movie")]  // Имя в GraphQL схеме
        [GraphQLDescription("Gets a movie by its ID")]  // Описание
        public async Task<MovieDto?> GetMovieById(
            [GraphQLName("id")] int id,  // Параметр запроса
            [Service] IMovieService movieService,
            [Service] ILogger<MovieQuery> logger)
        {
            try
            {
                return await movieService.GetMovieByIdAsync(id);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, $"Error while getting movie with ID {id}");
                throw;
            }
        }
    }
}
```

**Что происходит:**
- `MovieQuery` — класс, содержащий все Query операции
- Каждый метод становится доступным как GraphQL запрос
- `[GraphQLName("movies")]` — задает имя в GraphQL схеме (по умолчанию было бы `getMovies`)
- `[Service]` — атрибут для Dependency Injection (HotChocolate автоматически внедряет сервисы)
- Методы могут быть async и возвращать `Task<T>`

**Пример GraphQL запроса:**
```graphql
query {
  movies {
    id
    title
    description
  }
}
```

**Или с параметром:**
```graphql
query {
  movie(id: 1) {
    id
    title
    score
  }
}
```

---

### 4. Определение типов (Types)

```csharp
// AbsoluteCinema.WebAPI/GraphQL/Types/MovieType.cs

using AbsoluteCinema.Application.DTO.Entities;
using HotChocolate.Types;

namespace AbsoluteCinema.WebAPI.GraphQL.Types
{
    // Наследуемся от ObjectType<T>, где T - это DTO класс
    public class MovieType : ObjectType<MovieDto>
    {
        protected override void Configure(IObjectTypeDescriptor<MovieDto> descriptor)
        {
            // Общее описание типа
            descriptor.Description("Represents a movie in the cinema");
            
            // Настройка каждого поля
            descriptor
                .Field(m => m.Id)
                .Description("The unique identifier of the movie");
                
            descriptor
                .Field(m => m.Title)
                .Description("The title of the movie");
                
            // Переименование поля (в C# Discription, в GraphQL description)
            descriptor
                .Field(m => m.Discription)
                .Name("description")  // Переименовываем для GraphQL
                .Description("The description of the movie");
                
            descriptor
                .Field(m => m.ReleaseDate)
                .Description("The release date of the movie");
                
            descriptor
                .Field(m => m.Score)
                .Description("The score/rating of the movie");

            descriptor
                .Field(m => m.Adult)
                .Description("Whether the movie is marked as adult");

            descriptor
                .Field(m => m.PosterPath)
                .Description("Poster path");

            descriptor
                .Field(m => m.Language)
                .Description("Movie language");

            descriptor
                .Field(m => m.TrailerPath)
                .Description("Trailer path");

            // Вложенные объекты (связи)
            descriptor
                .Field(m => m.MovieGenres)
                .Description("Genres linked to the movie");

            descriptor
                .Field(m => m.MovieActors)
                .Description("Actors linked to the movie");
        }
    }
}
```

**Что происходит:**
- `MovieType` наследуется от `ObjectType<MovieDto>` — это говорит HotChocolate, как преобразовать C# DTO в GraphQL тип
- `Configure()` метод позволяет настроить каждое поле:
  - Добавить описание
  - Переименовать поле
  - Скрыть/показать поля
  - Настроить резолверы для вложенных объектов

**Зачем это нужно:**
- Контроль над тем, какие поля доступны в GraphQL
- Возможность переименовать поля (например, `Discription` → `description`)
- Добавление документации для автодополнения в GraphQL Playground

---

### 5. Middleware для логирования

```csharp
// AbsoluteCinema.WebAPI/GraphQL/Middleware/RequestLoggingMiddleware.cs

using System.Diagnostics;
using HotChocolate.Execution;
using Microsoft.Extensions.Logging;
using RequestDelegate = HotChocolate.Execution.RequestDelegate;

namespace AbsoluteCinema.WebAPI.GraphQL.Middleware
{
    public class RequestLoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<RequestLoggingMiddleware> _logger;

        public RequestLoggingMiddleware(
            RequestDelegate next,
            ILogger<RequestLoggingMiddleware> logger)
        {
            _next = next ?? throw new ArgumentNullException(nameof(next));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }

        public async ValueTask InvokeAsync(IRequestContext context)
        {
            var start = Stopwatch.GetTimestamp();
            try
            {
                // Вызываем следующий middleware в цепочке
                await _next(context);
            }
            finally
            {
                // Логируем время выполнения запроса
                var elapsed = Stopwatch.GetElapsedTime(start);

                if (context.Operation is not null)
                {
                    _logger.LogInformation(
                        "GraphQL operation '{OperationName}' completed in {ElapsedMilliseconds}ms",
                        context.Operation.Name ?? "anonymous",
                        elapsed.TotalMilliseconds);
                }
            }
        }
    }
}
```

**Что происходит:**
- Middleware перехватывает каждый GraphQL запрос
- Замеряет время выполнения
- Логирует имя операции и время выполнения

**Пример лога:**
```
GraphQL operation 'GetMovies' completed in 45.2ms
```

---

## 🎨 Frontend: Как работает GraphQL на клиенте

### Используемые технологии:
- **Apollo Client** — клиент для работы с GraphQL
- **@apollo/client** — основной пакет
- **graphql** — парсер GraphQL запросов

### Структура файлов:

```
AbsoluteCinema_Frontend/src/
├── apollo/
│   ├── client.ts              # Настройка Apollo Client
│   └── ApolloProviderWrapper.tsx  # Provider для React
├── hooks/
│   └── useGraphQLQuery.js     # Кастомный хук для запросов
└── pages/
    └── GraphQLMoviesPage.jsx  # Пример использования
```

---

## 📝 Код Frontend с объяснениями

### 1. Настройка Apollo Client

```typescript
// AbsoluteCinema_Frontend/src/apollo/client.ts

import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';
import { APP_CONFIG } from '../env';

// 1. Создаем HTTP ссылку на GraphQL endpoint
const httpLink = createHttpLink({
  uri: `${APP_CONFIG.HUB_URL}/graphql`, // URL GraphQL сервера
});

// 2. Создаем middleware для добавления токена авторизации
const authLink = setContext((_, { headers }) => {
  // Получаем токен из localStorage
  const token = localStorage.getItem('token');
  
  // Возвращаем заголовки с токеном
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
    }
  };
});

// 3. Создаем экземпляр Apollo Client
export const client = new ApolloClient({
  link: authLink.concat(httpLink),  // Цепочка: сначала auth, потом http
  cache: new InMemoryCache(),        // Кэш для хранения данных
  defaultOptions: {
    watchQuery: {
      fetchPolicy: 'cache-first',    // Сначала проверяем кэш
      errorPolicy: 'all',            // Показываем все ошибки
    },
    query: {
      fetchPolicy: 'network-only',   // Всегда запрашиваем с сервера
      errorPolicy: 'all',
    },
    mutate: {
      errorPolicy: 'all',
    },
  },
});
```

**Что происходит:**
- `httpLink` — указывает, куда отправлять GraphQL запросы
- `authLink` — автоматически добавляет JWT токен в заголовки каждого запроса
- `authLink.concat(httpLink)` — создает цепочку: сначала добавляется токен, потом отправляется запрос
- `InMemoryCache` — кэширует результаты запросов в памяти браузера
- `fetchPolicy` — политика кэширования:
  - `cache-first` — сначала проверяет кэш, если нет — запрашивает с сервера
  - `network-only` — всегда запрашивает с сервера, игнорируя кэш
  - `cache-and-network` — запрашивает с сервера, но показывает данные из кэша, если они есть

---

### 2. Apollo Provider для React

```typescript
// AbsoluteCinema_Frontend/src/apollo/ApolloProviderWrapper.tsx

import { ApolloProvider } from '@apollo/client/react';
import { client } from './client';
import { ReactNode } from 'react';

type Props = {
  children: ReactNode;
};

export const ApolloProviderWrapper = ({ children }: Props) => {
  return <ApolloProvider client={client}>{children}</ApolloProvider>;
};
```

**Что происходит:**
- `ApolloProvider` — React Context Provider, который делает Apollo Client доступным во всех дочерних компонентах
- Обернув приложение в `ApolloProviderWrapper`, все компоненты могут использовать GraphQL хуки

**Использование в main.jsx:**
```jsx
// AbsoluteCinema_Frontend/src/main.jsx

import { ApolloProviderWrapper } from './apollo/ApolloProviderWrapper';

createRoot(document.getElementById('root')).render(
    <ApolloProviderWrapper>
        <MovieProvider>
            <RouteHandler />
        </MovieProvider>
    </ApolloProviderWrapper>
)
```

---

### 3. Кастомный хук для запросов

```javascript
// AbsoluteCinema_Frontend/src/hooks/useGraphQLQuery.js

import { useQuery } from '@apollo/client/react';

export const useGraphQLQuery = (query, options = {}) => {
  const { data, loading, error, refetch } = useQuery(query, {
    ...options,
    fetchPolicy: options.fetchPolicy || 'cache-and-network',
  });

  return { data, loading, error, refetch };
};
```

**Что происходит:**
- Обертка над стандартным `useQuery` из Apollo Client
- Упрощает использование: не нужно каждый раз указывать `fetchPolicy`
- Возвращает:
  - `data` — данные из запроса
  - `loading` — флаг загрузки
  - `error` — ошибка, если есть
  - `refetch` — функция для повторного запроса

---

### 4. Пример использования в компоненте

```jsx
// AbsoluteCinema_Frontend/src/pages/GraphQLMoviesPage.jsx

import React from 'react';
import { gql } from '@apollo/client';
import { useGraphQLQuery } from '../hooks/useGraphQLQuery';

// 1. Определяем GraphQL запрос
const GET_MOVIES = gql`
  query GetMovies {
    movies {
      id
      title
      description
      releaseDate
      score
      adult
      posterPath
      language
      trailerPath
    }
  }
`;

const GraphQLMoviesPage = () => {
  // 2. Используем хук для выполнения запроса
  const { data, loading, error } = useGraphQLQuery(GET_MOVIES);

  // 3. Обрабатываем состояния загрузки и ошибки
  if (loading) return <div className="text-center my-5">Loading movies...</div>;
  if (error) return <div className="alert alert-danger">Error loading movies: {error.message}</div>;

  // 4. Извлекаем данные
  const movies = data?.movies || [];

  // 5. Рендерим компонент
  return (
    <div className="container mt-4">
      <h2 className="mb-4">Movies (GraphQL)</h2>
      
      {movies.length === 0 ? (
        <div className="alert alert-info">No movies found.</div>
      ) : (
        <div className="row">
          {movies.map((movie) => (
            <div key={movie.id} className="col-md-4 mb-4">
              <div className="card h-100">
                <div className="card-body">
                  <h5 className="card-title">{movie.title}</h5>
                  <h6 className="card-subtitle mb-2 text-muted">
                    {movie.releaseDate ? new Date(movie.releaseDate).getFullYear() : 'N/A'} • 
                    Score: {movie.score || 'N/A'}
                  </h6>
                  <p className="card-text">
                    {movie.description?.length > 150 
                      ? `${movie.description.substring(0, 150)}...` 
                      : movie.description}
                  </p>
                </div>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default GraphQLMoviesPage;
```

**Что происходит:**
1. **Определение запроса** — `gql` парсит GraphQL строку в объект запроса
2. **Выполнение запроса** — `useGraphQLQuery` автоматически выполняет запрос при монтировании компонента
3. **Состояния**:
   - `loading === true` — запрос выполняется
   - `error !== undefined` — произошла ошибка
   - `data` — данные получены
4. **Рендеринг** — компонент автоматически перерендерится, когда данные загрузятся

---

## 🔄 Полный цикл запроса

### Пример: Получение списка фильмов

**1. Frontend создает запрос:**
```graphql
query GetMovies {
  movies {
    id
    title
    description
  }
}
```

**2. Apollo Client отправляет HTTP POST запрос:**
```
POST /graphql
Content-Type: application/json
Authorization: Bearer <token>

{
  "query": "query GetMovies { movies { id title description } }"
}
```

**3. HotChocolate получает запрос:**
- Парсит GraphQL запрос
- Определяет, что нужно вызвать `movies` query
- Находит метод `GetMovies()` в классе `MovieQuery`

**4. Выполняется resolver:**
```csharp
public async Task<IEnumerable<MovieDto>> GetMovies(...)
{
    return await movieService.GetAllMoviesWithIncludeAsync();
}
```

**5. Сервис получает данные из БД:**
- Entity Framework выполняет SQL запрос
- Данные маппятся в `MovieDto`

**6. HotChocolate фильтрует поля:**
- Клиент запросил только `id`, `title`, `description`
- HotChocolate возвращает только эти поля (благодаря проекциям)

**7. Ответ отправляется клиенту:**
```json
{
  "data": {
    "movies": [
      {
        "id": 1,
        "title": "Inception",
        "description": "A mind-bending thriller..."
      },
      {
        "id": 2,
        "title": "The Matrix",
        "description": "A computer hacker learns..."
      }
    ]
  }
}
```

**8. Apollo Client:**
- Кэширует данные в `InMemoryCache`
- Обновляет компонент с новыми данными
- Компонент перерендеривается

---

## 🎓 Основы GraphQL синтаксиса

### Query (Запрос данных)

**Простой запрос:**
```graphql
query {
  movies {
    id
    title
  }
}
```

**Запрос с именем:**
```graphql
query GetMovies {
  movies {
    id
    title
  }
}
```

**Запрос с параметрами:**
```graphql
query GetMovie {
  movie(id: 1) {
    id
    title
    description
  }
}
```

**Запрос с переменными:**
```graphql
query GetMovie($movieId: Int!) {
  movie(id: $movieId) {
    id
    title
  }
}
```

**Вложенные объекты:**
```graphql
query {
  movies {
    id
    title
    movieGenres {
      id
      name
    }
    movieActors {
      id
      name
    }
  }
}
```

**Алиасы (если нужно запросить одно поле несколько раз):**
```graphql
query {
  firstMovie: movie(id: 1) {
    title
  }
  secondMovie: movie(id: 2) {
    title
  }
}
```

**Фрагменты (переиспользуемые части запроса):**
```graphql
fragment MovieInfo on Movie {
  id
  title
  description
  score
}

query {
  movies {
    ...MovieInfo
    releaseDate
  }
}
```

---

### Mutation (Изменение данных)

**Пример мутации (если бы она была реализована):**
```graphql
mutation CreateMovie($input: CreateMovieInput!) {
  createMovie(input: $input) {
    id
    title
  }
}
```

**На бекенде это выглядело бы так:**
```csharp
public class MovieMutation
{
    public async Task<MovieDto> CreateMovie(
        CreateMovieInput input,
        [Service] IMovieService movieService)
    {
        return await movieService.CreateMovieAsync(input);
    }
}
```

---

## 🛠️ Полезные возможности HotChocolate

### Фильтрация (AddFiltering)

```graphql
query {
  movies(where: { score: { gt: 7 } }) {
    id
    title
    score
  }
}
```

### Сортировка (AddSorting)

```graphql
query {
  movies(order: { score: DESC }) {
    id
    title
    score
  }
}
```

### Пагинация

```graphql
query {
  movies(first: 10, skip: 20) {
    id
    title
  }
}
```

---

## 🔍 GraphQL Playground

После запуска приложения в development режиме, доступен GraphQL Playground:

**URL:** `http://localhost:5299/graphql/ui`

**Возможности:**
- Интерактивное выполнение запросов
- Автодополнение схемы
- Документация API
- История запросов
- Визуализация схемы

**Пример использования:**
1. Откройте `/graphql/ui` в браузере
2. В левой панели введите запрос:
```graphql
query {
  movies {
    id
    title
    score
  }
}
```
3. Нажмите кнопку "Play"
4. Увидите результат в правой панели

---

## 📊 Сравнение REST vs GraphQL

### REST API:
```javascript
// Нужно сделать 3 запроса
const movies = await fetch('/api/movies');
const movie = await fetch('/api/movies/1');
const actors = await fetch('/api/movies/1/actors');
```

### GraphQL:
```graphql
# Один запрос получает всё
query {
  movies {
    id
    title
  }
  movie(id: 1) {
    id
    title
    movieActors {
      id
      name
    }
  }
}
```

**Преимущества GraphQL:**
- ✅ Меньше запросов к серверу
- ✅ Запрашиваешь только нужные поля
- ✅ Сильная типизация
- ✅ Автодополнение в IDE

**Недостатки GraphQL:**
- ❌ Сложнее кэширование на уровне HTTP
- ❌ Может быть сложнее для простых CRUD операций
- ❌ Требует больше настройки на бекенде

---

## 🎯 Резюме

### Backend (C# .NET):
1. **GraphQLSchema.cs** — регистрирует GraphQL сервер
2. **MovieQuery.cs** — определяет запросы (Query)
3. **MovieType.cs** — определяет типы данных
4. **RequestLoggingMiddleware.cs** — логирует запросы
5. **Program.cs** — подключает GraphQL к приложению

### Frontend (React):
1. **client.ts** — настраивает Apollo Client
2. **ApolloProviderWrapper.tsx** — оборачивает приложение
3. **useGraphQLQuery.js** — кастомный хук для запросов
4. **GraphQLMoviesPage.jsx** — пример использования

### Ключевые концепции:
- **Query** — получение данных
- **Type** — описание структуры данных
- **Resolver** — метод, который возвращает данные
- **Schema** — описание всех доступных операций

---

## 📚 Дополнительные ресурсы

- [GraphQL Official Docs](https://graphql.org/learn/)
- [HotChocolate Docs](https://chillicream.com/docs/hotchocolate)
- [Apollo Client Docs](https://www.apollographql.com/docs/react/)

---

**Создано для проекта AbsoluteCinema** 🎬


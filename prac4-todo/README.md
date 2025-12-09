# Практическое занятие №4 Маршрутизация с chi (альтернатива — gorilla/mux). Создание небольшого CRUD-сервиса «Список задач».

## Цели работы
1.	Освоить базовую маршрутизацию HTTP-запросов в Go на примере роутера chi.
2.	Научиться строить REST-маршруты и обрабатывать методы GET/POST/PUT/DELETE.
3.	Реализовать небольшой CRUD-сервис «ToDo» (без БД, хранение в памяти).
4.	Добавить простое middleware (логирование, CORS).
5.	Научиться тестировать API запросами через curl/Postman/HTTPie.


## Структура проекта

```
prac4-todo/
├── main.go                 
├── go.mod                  
├── go.sum                  
├── internal/
│   └── task/
│       ├── handler.go
│       ├── model.go        
│       └── repo.go         
└── pkg/
    └── middleware/
        ├── cors.go         
        └── logger.go       
```

## Фрагменты кода роутера, middleware, обработчиков

[main.go line 18](./pz4-todo/main.go)
```GO
r := chi.NewRouter()
	r.Use(chimw.RequestID)
	r.Use(chimw.Recoverer)
	r.Use(myMW.Logger)
	r.Use(myMW.SimpleCORS)

	r.Get("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("OK"))
	})

	r.Route("/api", func(api chi.Router) {
		api.Mount("/tasks", h.Routes())
	})
```

[logger.go](./pkg/middleware/logger.go)
```GO
package middleware

import (
	"log"
	"net/http"
	"time"
)

func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
	})
}
```

[handler.go](./internal/task/handler.go)

- Роутинг line 19
```GO
func (h *Handler) Routes() chi.Router {
	r := chi.NewRouter()
	r.Get("/", h.list)         // GET /tasks
	r.Post("/", h.create)       // POST /tasks
	r.Get("/{id}", h.get)       // GET /tasks/{id}
	r.Put("/{id}", h.update)    // PUT /tasks/{id}
	r.Delete("/{id}", h.delete) // DELETE /tasks/{id}
	return r
}
```

-  Обработка ошибок 404 line 33
```GO
func (h *Handler) get(w http.ResponseWriter, r *http.Request) {
	id, bad := parseID(w, r)
	if bad {
		return
	}
	t, err := h.repo.Get(id)
	if err != nil {
		httpError(w, http.StatusNotFound, err.Error())  // ← 404 ошибка
		return
	}
	writeJSON(w, http.StatusOK, t)
}
```

-  Обработка ошибок 400 line 50
```GO
func (h *Handler) create(w http.ResponseWriter, r *http.Request) {
	var req createReq
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil || req.Title == "" {
		httpError(w, http.StatusBadRequest, "invalid json: require non-empty title")  
		return
	}
	t := h.repo.Create(req.Title)
	writeJSON(w, http.StatusCreated, t)  
}
```

- Вспомогательные функции (JSON и ошибки) line 107
```GO
func writeJSON(w http.ResponseWriter, code int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(code)
	_ = json.NewEncoder(w).Encode(v)
}

func httpError(w http.ResponseWriter, code int, msg string) {
	writeJSON(w, code, map[string]string{"error": msg})
}
```
## Результаты тестирования

- ` curl -Method POST -Uri "http://localhost:8080/api/tasks" -Headers @{"Content-Type"="application/json"} -Body '{"title":"Выучить chi"}'` тестирование создания задачи

image.png

- `curl -Method GET -Uri "http://localhost:8080/api/tasks"` тестирование получения данных 



- ` curl -Method GET -Uri "http://localhost:8080/api/tasks/1"` тестирование получения данных по ID



- ` curl -Method PUT -Uri "http://localhost:8080/api/tasks/1" -Headers @{"Content-Type"="application/json"} -Body '{"title":"Learn chi deeper","done":true}'` тестирование обновления



- ` curl -Method DELETE -Uri "http://localhost:8080/api/tasks/1"` тестирование удаления



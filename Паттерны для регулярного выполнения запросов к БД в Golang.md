# Паттерны для регулярного выполнения запросов к БД в Golang

Для вашей задачи (регулярное выполнение запросов к БД с сохранением результатов) лучше всего подойдут следующие паттерны:

## 1. Worker Pool + Scheduled Tasks (Пул воркеров с планировщиком)

**Лучший выбор для большинства случаев** - сочетание пула воркеров и планировщика:

```go
type DBWorker struct {
    db      *sql.DB
    jobs    chan QueryTask
    stop    chan struct{}
    wg      sync.WaitGroup
}

type QueryTask struct {
    Query      string
    Parameters []interface{}
    ResultDest interface{}
}

func NewDBWorker(db *sql.DB, workerCount int) *DBWorker {
    w := &DBWorker{
        db:   db,
        jobs: make(chan QueryTask, 100),
        stop: make(chan struct{}),
    }
    
    for i := 0; i < workerCount; i++ {
        w.wg.Add(1)
        go w.worker()
    }
    
    return w
}

func (w *DBWorker) worker() {
    defer w.wg.Done()
    for {
        select {
        case task := <-w.jobs:
            // Выполнение запроса
            rows, err := w.db.Query(task.Query, task.Parameters...)
            if err != nil {
                log.Printf("Query failed: %v", err)
                continue
            }
            
            // Обработка результатов
            // ... (сохранение в task.ResultDest)
            
            // Сохранение результатов в БД
            _, err = w.db.Exec("INSERT INTO query_results (...) VALUES (...)", ...)
            if err != nil {
                log.Printf("Failed to save results: %v", err)
            }
            
        case <-w.stop:
            return
        }
    }
}

func (w *DBWorker) Schedule(task QueryTask, interval time.Duration) {
    ticker := time.NewTicker(interval)
    go func() {
        for {
            select {
            case <-ticker.C:
                w.jobs <- task
            case <-w.stop:
                ticker.Stop()
                return
            }
        }
    }()
}

func (w *DBWorker) Stop() {
    close(w.stop)
    w.wg.Wait()
}
```

## 2. Ticker-based Service (Сервис на основе тикера)

**Простое решение для периодических задач**:

```go
type DBPoller struct {
    db        *sql.DB
    interval  time.Duration
    stop      chan struct{}
    query     string
    saveQuery string
}

func NewDBPoller(db *sql.DB, interval time.Duration, query, saveQuery string) *DBPoller {
    return &DBPoller{
        db:        db,
        interval:  interval,
        stop:      make(chan struct{}),
        query:     query,
        saveQuery: saveQuery,
    }
}

func (p *DBPoller) Start() {
    ticker := time.NewTicker(p.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // Выполнение запроса
            rows, err := p.db.Query(p.query)
            if err != nil {
                log.Printf("Query failed: %v", err)
                continue
            }
            
            // Обработка результатов
            var results []interface{}
            // ... (сканирование rows в results)
            
            // Сохранение результатов
            _, err = p.db.Exec(p.saveQuery, results...)
            if err != nil {
                log.Printf("Failed to save results: %v", err)
            }
            
        case <-p.stop:
            return
        }
    }
}

func (p *DBPoller) Stop() {
    close(p.stop)
}
```

## 3. Event-driven with Debouncing (Событийный подход с дебаунсингом)

**Хорош при нерегулярных, но частых обновлениях**:

```go
type DBDebouncedPoller struct {
    db         *sql.DB
    request    chan struct{}
    debounce   time.Duration
    lastUpdate time.Time
    stop       chan struct{}
    query      string
    saveQuery  string
}

func NewDBDebouncedPoller(db *sql.DB, debounce time.Duration, query, saveQuery string) *DBDebouncedPoller {
    return &DBDebouncedPoller{
        db:        db,
        request:   make(chan struct{}, 100),
        debounce:  debounce,
        stop:      make(chan struct{}),
        query:     query,
        saveQuery: saveQuery,
    }
}

func (p *DBDebouncedPoller) Trigger() {
    p.request <- struct{}{}
}

func (p *DBDebouncedPoller) Start() {
    for {
        select {
        case <-p.request:
            if time.Since(p.lastUpdate) < p.debounce {
                continue
            }
            
            // Выполнение запроса и сохранение
            p.executeAndSave()
            p.lastUpdate = time.Now()
            
        case <-p.stop:
            return
        }
    }
}

func (p *DBDebouncedPoller) executeAndSave() {
    // Реализация выполнения запроса и сохранения
}
```

## Рекомендации по выбору:

1. **Для строго периодических задач** (раз в минуту/час/день) - используйте **Ticker-based Service**
2. **Для переменной нагрузки с параллельной обработкой** - **Worker Pool + Scheduler**
3. **Для событийно-управляемых систем** - **Event-driven с дебаунсингом**

Дополнительные улучшения:
- Добавьте Circuit Breaker для обработки ошибок БД
- Реализуйте retry-логику для временных сбоев
- Используйте connection pooling для работы с БД
- Добавьте метрики и логирование выполнения запросов

Пример интеграции с промышленным решением (используя `github.com/go-co-op/gocron`):

```go
func setupScheduledJobs(db *sql.DB) {
    s := gocron.NewScheduler(time.UTC)
    
    // Ежедневно в 3:00
    _, _ = s.Every(1).Day().At("03:00").Do(func() {
        executeAndSaveDailyReport(db)
    })
    
    // Каждые 5 минут
    _, _ = s.Every(5).Minutes().Do(func() {
        executeAndSaveRegularQuery(db)
    })
    
    s.StartAsync()
}
```
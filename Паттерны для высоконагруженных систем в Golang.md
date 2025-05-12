# Паттерны для высоконагруженных систем в Golang

Для разработки высоконагруженных систем в Go используются специальные паттерны, которые помогают эффективно распределять ресурсы, минимизировать блокировки и масштабировать систему. Вот основные из них:

## 1. Worker Pool (Пул воркеров)

Организация пула горутин для обработки задач:

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Println("worker", id, "processing job", j)
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // Запускаем 3 воркера
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }
    
    // Отправляем задания
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Получаем результаты
    for a := 1; a <= 9; a++ {
        <-results
    }
}
```

## 2. Fan-out/Fan-in

Распределение задач между множеством воркеров и сбор результатов:

```go
func producer(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)
    
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }
    
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

func main() {
    in := producer(1, 2, 3, 4)
    
    // Fan-out
    c1 := square(in)
    c2 := square(in)
    
    // Fan-in
    for n := range merge(c1, c2) {
        fmt.Println(n)
    }
}
```

## 3. Circuit Breaker (Автоматический выключатель)

Защита системы от каскадных ошибок:

```go
type CircuitBreaker struct {
    maxFailures int
    failures    int
    cooldown    time.Duration
    lastFailure time.Time
    mux         sync.Mutex
}

func (cb *CircuitBreaker) Execute(f func() error) error {
    cb.mux.Lock()
    
    if cb.failures >= cb.maxFailures {
        if time.Since(cb.lastFailure) < cb.cooldown {
            cb.mux.Unlock()
            return fmt.Errorf("circuit breaker is open")
        }
        // Переход в half-open состояние
    }
    cb.mux.Unlock()
    
    err := f()
    
    cb.mux.Lock()
    defer cb.mux.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        return err
    }
    
    // Сброс при успешном выполнении
    if cb.failures > 0 {
        cb.failures = 0
    }
    return nil
}
```

## 4. Rate Limiter (Ограничитель запросов)

Контроль частоты выполнения операций:

```go
type RateLimiter struct {
    limit    int
    interval time.Duration
    tokens   int
    lastTime time.Time
    mux      sync.Mutex
}

func NewRateLimiter(limit int, interval time.Duration) *RateLimiter {
    return &RateLimiter{
        limit:    limit,
        interval: interval,
        tokens:   limit,
        lastTime: time.Now(),
    }
}

func (rl *RateLimiter) Allow() bool {
    rl.mux.Lock()
    defer rl.mux.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(rl.lastTime)
    
    // Пополняем токены
    refill := int(elapsed / rl.interval)
    if refill > 0 {
        rl.tokens = min(rl.tokens+refill, rl.limit)
        rl.lastTime = now
    }
    
    if rl.tokens > 0 {
        rl.tokens--
        return true
    }
    return false
}
```

## 5. Sharded Map (Шардированная мапа)

Распределение нагрузки по нескольким мапам с отдельными блокировками:

```go
type ShardedMap struct {
    shards []*shard
    hasher func(string) uint32
}

type shard struct {
    items map[string]interface{}
    sync.RWMutex
}

func NewShardedMap(shardsNum int) *ShardedMap {
    shards := make([]*shard, shardsNum)
    for i := 0; i < shardsNum; i++ {
        shards[i] = &shard{items: make(map[string]interface{})}
    }
    return &ShardedMap{
        shards: shards,
        hasher: fnv.New32(),
    }
}

func (m *ShardedMap) getShard(key string) *shard {
    h := m.hasher.Sum32(key)
    return m.shards[h%uint32(len(m.shards))]
}

func (m *ShardedMap) Set(key string, value interface{}) {
    shard := m.getShard(key)
    shard.Lock()
    shard.items[key] = value
    shard.Unlock()
}

func (m *ShardedMap) Get(key string) (interface{}, bool) {
    shard := m.getShard(key)
    shard.RLock()
    val, ok := shard.items[key]
    shard.RUnlock()
    return val, ok
}
```

## 6. Connection Pool (Пул соединений)

Эффективное управление соединениями с БД или другими сервисами:

```go
type ConnPool struct {
    pool    chan net.Conn
    factory func() (net.Conn, error)
    mu      sync.Mutex
}

func NewConnPool(size int, factory func() (net.Conn, error)) *ConnPool {
    return &ConnPool{
        pool:    make(chan net.Conn, size),
        factory: factory,
    }
}

func (p *ConnPool) Get() (net.Conn, error) {
    select {
    case conn := <-p.pool:
        return conn, nil
    default:
        return p.factory()
    }
}

func (p *ConnPool) Put(conn net.Conn) {
    select {
    case p.pool <- conn:
    default:
        conn.Close()
    }
}
```

## 7. Batching (Пакетная обработка)

Группировка запросов для уменьшения накладных расходов:

```go
type Batcher struct {
    size     int
    timeout  time.Duration
    input    chan Item
    output   chan []Item
    stopOnce sync.Once
    stop     chan struct{}
}

func NewBatcher(size int, timeout time.Duration) *Batcher {
    return &Batcher{
        size:    size,
        timeout: timeout,
        input:   make(chan Item),
        output:  make(chan []Item),
        stop:    make(chan struct{}),
    }
}

func (b *Batcher) Run() {
    batch := make([]Item, 0, b.size)
    timer := time.NewTimer(b.timeout)
    
    for {
        select {
        case item := <-b.input:
            batch = append(batch, item)
            if len(batch) >= b.size {
                b.output <- batch
                batch = make([]Item, 0, b.size)
                timer.Reset(b.timeout)
            }
        case <-timer.C:
            if len(batch) > 0 {
                b.output <- batch
                batch = make([]Item, 0, b.size)
            }
            timer.Reset(b.timeout)
        case <-b.stop:
            if len(batch) > 0 {
                b.output <- batch
            }
            close(b.output)
            return
        }
    }
}
```

## 8. Caching (Кэширование)

Эффективное кэширование данных с учетом TTL:

```go
type Cache struct {
    items   map[string]Item
    ttl     time.Duration
    mu      sync.RWMutex
    stop    chan struct{}
}

func NewCache(ttl time.Duration, cleanupInterval time.Duration) *Cache {
    c := &Cache{
        items: make(map[string]Item),
        ttl:   ttl,
        stop:  make(chan struct{}),
    }
    
    go c.cleanup(cleanupInterval)
    return c
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = Item{
        Value:    value,
        ExpireAt: time.Now().Add(c.ttl),
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    item, found := c.items[key]
    if !found || time.Now().After(item.ExpireAt) {
        return nil, false
    }
    return item.Value, true
}

func (c *Cache) cleanup(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            c.mu.Lock()
            for key, item := range c.items {
                if time.Now().After(item.ExpireAt) {
                    delete(c.items, key)
                }
            }
            c.mu.Unlock()
        case <-c.stop:
            return
        }
    }
}
```

Эти паттерны помогают создавать высоконагруженные системы на Go, обеспечивая эффективное использование ресурсов, масштабируемость и отказоустойчивость.
# creational
## singleton
usecase
1. A class has one and only instance.
2. It provide global access to the instance of the class

![singleton](https://refactoring.guru/images/patterns/content/singleton/singleton-2x.png?id=accb2cc7594f7a491ce01dddf0d2f876)

[example](https://github.com/deliveryhero/pd-subscription/blob/master/app/infrastructure/config/specfile/api.go#L15)

Singleton fits the need when app's need to read app's configuration in different processes. **LoadAPIAppConfig** is the function that always load the same app config for **subscription's backend**. But be careful, it's not singleton. Why?

```golang
func LoadAPIAppConfig() (*APIAppConfig, *datadog.EnvConfig, error) {
	// Structure of specfile
	specFileConfig := new(apiSpecFileConfig)
	dataDogConfig := new(datadog.EnvConfig)
	if err := loadConfig(specFileConfig, dataDogConfig); err != nil {
		return nil, nil, err
	}
	return config, dataDogConfig, nil
}
```

## Factory
usecase
1. extend internal property of an object
2. decouple creation from the client code
3. create different types of instances with ease

![factory](https://assets.alexandria.raywenderlich.com/books/des/images/c370c4f82221dae3e9260291d995b3e4e8ba5bd3904522bdacb68d35562729b7/original.png)

[example](https://github.com/deliveryhero/pd-subscription/blob/master/app/infrastructure/cache/invalidator.go#L32-L70)

User's subscription is cached for better performance. However, subscription needs to be evicted from cached for certain events in order to achieve data consistency. The **Action** does the job. Look at how Action prepares messages from event by type assertion. Imagine more types of events to come in the future, this pattern handle this need gracefully.


# structural
## Bridge
usecase

1. organize classes that has the same functionality
2. swap the underlying class at run time.

![bridge](https://refactoring.guru/images/patterns/content/bridge/bridge-2-en-2x.png?id=bbd64c96e6711636356944b3564ad67e)


[example](https://github.com/deliveryhero/pd-subscription/blob/master/app/usecase/subscription/service.go#L155)

Subscription's API needs to retrieve subscription status for user. This is where **Repository** interface comes in. It **bridges** the needs of client to interact with different data store. Imagine in the future you want to migrate subscriber's data into different data store. You don't have to rewrite everything in the client's code. You only change implemntation of repository.
```golang
type SubscriptionRepository interface {
	GetLatestSubscription(ctx context.Context, globalEntityID string, customerCode string) (*entities.Subscription, error)
}


type SubscriptionRepository struct {
	db          *sql.DB
	loggerEntry *logrus.Entry
}

// GetLatestSubscription behaves like GetSubscriptionHistory but only takes the latest subscription
func (sr *SubscriptionRepository) GetLatestSubscription(ctx context.Context, globalEntityID string, customerCode string) (*entities.Subscription, error) {
	qr := GetQuerier(ctx, sr.db)

	// querying the subscription data order by its created_at column in descending order, hence ensuring the very first element is the latest and current subscription
	s, err := sr.scanSubscription(qr.QueryRowContext(ctx, selectSubscription+" WHERE  customer_code = ? ORDER BY `created_at` DESC LIMIT 1", customerCode))

	// No results
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, subscriptionRepositoryErr{errors.Wrapf(err, "Error while reading subscription data with Global entity Id: %s and Customer Code: %s", globalEntityID, customerCode)}
	}

	return &s, nil
}

func NewAPIService(
	subscriptionRepository pdsubscription.SubscriptionRepository,
) *Service {
	return &Service{
		subscriptionRepository:               subscriptionRepository,
	}
}
```

## facade
facade let client interacts with dozens of components with an interface. It's similar to orchestration.
https://github.com/kubernetes/kubernetes/blob/f6e163fe27f2d698af5309aa946515ee62d98e40/pkg/kubelet/pod/pod_manager.go#L102

https://refactoring.guru/design-patterns/facade


## proxy
usecase
1. Control access to the service
2. Proxy has the same interface as servce.

https://github.com/deliveryhero/pd-subscription/blob/master/app/infrastructure/cache/subscription_repository_cached.go#L84

## decorator
usecase
1. adds additional behavior to an object
2. object's internal is not changed

![decorator](https://refactoring.guru/images/patterns/content/decorator/decorator-comic-1-2x.png?id=ba869f621b6e0ea173fdc2b535fc7eed)

[example](https://github.com/deliveryhero/pd-subscription/blob/master/app/presentation/http/middleware/logger_middleware.go#L54)

It's desire to log info about http request/response. Using a decorator save us from writing the logs everywhere in the handler function.

```golang
// Wrap will be called in every request
func (m *LogMiddleware) Wrap(next http.Handler) http.Handler {
	if m.fn == nil {
		panic("Using LogMiddleware without set a log function (See: SetLogger or SetLoggerFunc)")
	}

	fn := func(w http.ResponseWriter, req *http.Request) {
		started := time.Now()

		lr := &LogResponse{
			ResponseWriter: w,
			req:            req,
		}
		next.ServeHTTP(lr, req)

		lr.Elapsed = time.Since(started)

		logReq := &LogRequest{
			Request:    *req,
			Response:   lr,
			RemoteAddr: getRemoteAddr(req),
		}

		m.fn(logReq)
	}

	return http.HandlerFunc(fn)
}
```

# behavioral
## memento
usecase
1. want to create a snapshot of the object
2. not allow to access to object's field

![memento](https://refactoring.guru/images/patterns/diagrams/memento/problem1-en-2x.png)

example

We want to keep x of **Position** private. Only **Sanpshot** can access the position x, and **Manager** keep track of position. Why do we use Manager class over stack?
```golang
package main

import "fmt"

type Snapshot struct {
	State int
}

type Position struct {
	x int
}

func (p *Position) CreateSnapshot() Snapshot {
	return Snapshot{
		State: p.x,
	}
}

func (p *Position) Undo(snapshop Snapshot) {
	p.x = snapshop.State
	fmt.Printf("at %d\n", p.x)
}

func (p *Position) Move(step int) {
	p.x = p.x + step
	fmt.Printf("at %d\n", p.x)
}

type Manager struct {
	snapshots []Snapshot
}

func (m *Manager) Append(snapshot Snapshot) {
	m.snapshots = append(m.snapshots, snapshot)
}

func (m *Manager) Pop() Snapshot {
	L := len(m.snapshots)
	last := m.snapshots[L-1]
	m.snapshots = m.snapshots[:L-1]
	return last
}

func (m *Manager) Len() int {
	return len(m.snapshots)
}

func main() {
	steps := []int{1, 2, 3}
	p := Position{0}
	manager := Manager{snapshots: make([]Snapshot, 0)}
	for _, step := range steps {
		manager.Append(p.CreateSnapshot())
		p.Move(step)
	}
	for manager.Len() > 0 {
		snapshot := manager.Pop()
		p.Undo(snapshot)
	}
    // will print
    // at 1
    // at 3
    // at 6
    // at 3
    // at 1
    // at 0
}

```


## strategy
usecase
1. Change algorithm at run time
2. Most of the method are the same; only few method are different.
3. Implementation is not important to business logic.

example

```golang
package main

import "fmt"

type EvictionAlgo interface {
    evict(c *Cache)
}

type Fifo struct {
}

func (l *Fifo) evict(c *Cache) {
    fmt.Println("Evicting by fifo strtegy")
}

type Lru struct {
}

func (l *Lru) evict(c *Cache) {
    fmt.Println("Evicting by lru strtegy")
}

type Cache struct {
    storage      map[string]string
    evictionAlgo EvictionAlgo
    capacity     int
    maxCapacity  int
}

func initCache(e EvictionAlgo) *Cache {
    storage := make(map[string]string)
    return &Cache{
        storage:      storage,
        evictionAlgo: e,
        capacity:     0,
        maxCapacity:  2,
    }
}

func (c *Cache) setEvictionAlgo(e EvictionAlgo) {
    c.evictionAlgo = e
}

func (c *Cache) add(key, value string) {
    if c.capacity == c.maxCapacity {
        c.evict()
    }
    c.capacity++
    c.storage[key] = value
}

func (c *Cache) get(key string) {
    delete(c.storage, key)
}

func (c *Cache) evict() {
    c.evictionAlgo.evict(c)
    c.capacity--
}

func main() {
    lru := &Lru{}
    cache := initCache(lru)
    cache.add("a", "1")
    cache.add("b", "2")
    cache.add("c", "3")
    cache.add("d", "4")

    fifo := &Fifo{}
    cache.setEvictionAlgo(fifo)
    cache.add("e", "5")

}
```
## observer
usecase
1. change of state in one object will affect the other（publisher vs subscriber)
2. type of the object is not known beforehand.

![observer](https://refactoring.guru/images/patterns/content/observer/observer-comic-1-en-2x.png?id=8e89674eb83b01e82203987e600ba59e)

example

In pd-subscription, observer pattern is used to monitor internal events. For example. when user cancels its subscription, an **subscription-cancel-event** will be **dispatched** to an **cancelled-receipt-email action**.
This way, we don't have to implement email action in the code where subscription is cancelled.

[where events are delivered](https://github.com/deliveryhero/pd-subscription/blob/master/app/usecase/subscription/service.go#L1306)

[where events are consumed by subscriber](https://github.com/deliveryhero/pd-subscription/blob/master/app/infrastructure/dispatcher/dispatcher.go#L52)

[where events are subscribed by customer](https://github.com/deliveryhero/pd-subscription/blob/master/cmd/api/main.go#L209)

```golang
package main

import "fmt"

type Event string

type Action func(data any)

type Dispatcher struct {
	listeners map[Event][]Action
}

func (d *Dispatcher) Register(event Event, action func(data any)) {
	d.listeners[event] = append(d.listeners[event], action)
}

func (d *Dispatcher) Dispatch(event Event, data any) {
	if actions, ok := d.listeners[event]; ok {
		for _, action := range actions {
			action(data)
		}
	}
}

const (
	hello     = "hello"
	howAreYou = "how are you"
)

func main() {
	dispatcher := Dispatcher{listeners: make(map[Event][]Action)}
	dispatcher.Register(hello, func(data any) { fmt.Println(data) })
	dispatcher.Register(howAreYou, func(data any) { fmt.Println(data) })
	fmt.Printf("event %s publish\n", hello)
	dispatcher.Dispatch(hello, "hello!")
	fmt.Printf("event %s publish\n", howAreYou)
	dispatcher.Dispatch(howAreYou, "I am fine.")
}
```

## iterator
1. traverse a collection of items
2. hide complex data structure
3. isolate traversal from business logic

![iterator](https://refactoring.guru/images/patterns/content/iterator/iterator-en-2x.png?id=2a85705e8e5fab257802b2ce36d6d236)

[example](https://refactoring.guru/design-patterns/iterator/go/example)

```golang
type UserIterator struct {
    index int
    users []*User
}

func (u *UserIterator) hasNext() bool {
    if u.index < len(u.users) {
        return true
    }
    return false

}

func (u *UserIterator) getNext() *User {
    if u.hasNext() {
        user := u.users[u.index]
        u.index++
        return user
    }
    return nil
}

type UserCollection struct {
    users []*User
}

func (u *UserCollection) createIterator() Iterator {
    return &UserIterator{
        users: u.users,
    }
}

type User struct {
    name string
    age  int
}

func main() {

    user1 := &User{
        name: "a",
        age:  30,
    }
    user2 := &User{
        name: "b",
        age:  20,
    }

    userCollection := &UserCollection{
        users: []*User{user1, user2},
    }

    iterator := userCollection.createIterator()

    for iterator.hasNext() {
        user := iterator.getNext()
        fmt.Printf("User is %+v\n", user)
    }
}
```

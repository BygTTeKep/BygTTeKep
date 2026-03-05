Это статический анализатор который помогает компилятору понять куда положить переменную, в кучу или стек
точка входа https://go.dev/src/cmd/compile/internal/gc/main.go#:~:text=escape.Funcs(typecheck.Target.Funcs)
Сначало строится ориентированный взвешенный граф(ориентированный значит есть направления, взвешенный значит есть вес ребра) где
- узлы - это переменные
- ребра- присвоения между переменными, вес считается как кол-во операций разыменования - кол-во операций адресаций
пример
```
p =q (вес 0)
p = *q (вес 1)
p = &q (вес -1)
p = **q(вес 2)
```
потом проходимся по графу ища такие TODO

escape структура
```go
type escape struct {
	*batch //указатель на батч функций всего пакета, т.к. golang компилирует по покетно, а EA анализирует пакет
	curfn *ir.Func указатель на текущую функцию
	
	labels map[*types.Sym]labelState // метки функции
	
	// Глубина вложености циклов, увеличивается внутри каждого for 
	// и в каждой метки goto
	loopDepth int 
}
```
сам batch
```go
type batch struct {
	allLocs []*location
	closures []closure
	reassingOracles map[*ir.Func]*ir.ReassingOracles
	heapLoc location
	mutatorLoc location
	calleeLoc location
	blanckLoc location
}
```
location - это абстрактное представление где хранится переменная
ReassingOracles - позволяет эффективно опеределять переназначения переменных

основное место
```go
func Funcs(all []*ir.Func) {
	// Make a cache of ir.ReassignOracles. The cache is lazily populated.
	// TODO(thepudds): consider adding a field on ir.Func instead. We might also be able
	// to use that field elsewhere, like in walk. See discussion in https://go.dev/cl/688075.
	reassignOracles := make(map[*ir.Func]*ir.ReassignOracle)

	ir.VisitFuncsBottomUp(all, func(list []*ir.Func, recursive bool) {
		Batch(list, reassignOracles)
	})
}
```
visitFunc вызывает функцию analyze(2-ой параметр) для каждой функции в списке, снизу вверх по графу
```go
func Batch(fns []*ir.Func, reassignOracles map[*ir.Func]*ir.ReassignOracle) {
	var b batch
	b.heapLoc.attrs = attrEscapes | attrPersists | attrMutates | attrCalls
	b.mutatorLoc.attrs = attrMutates
	b.calleeLoc.attrs = attrCalls
	b.reassignOracles = reassignOracles

	// Construct data-flow graph from syntax trees.
	for _, fn := range fns {
		if base.Flag.W > 1 {
			s := fmt.Sprintf("\nbefore escape %v", fn)
			ir.Dump(s, fn)
		}
		b.initFunc(fn)
	}
	for _, fn := range fns {
		if !fn.IsClosure() {
			b.walkFunc(fn)
		}
	}

	// We've walked the function bodies, so we've seen everywhere a
	// variable might be reassigned or have its address taken. Now we
	// can decide whether closures should capture their free variables
	// by value or reference.
	for _, closure := range b.closures {
		b.flowClosure(closure.k, closure.clo)
	}
	b.closures = nil

	for _, loc := range b.allLocs {
		// Try to replace some non-constant expressions with literals.
		b.rewriteWithLiterals(loc.n, loc.curfn)

		// Check if the node must be heap allocated for certain reasons
		// such as OMAKESLICE for a large slice.
		if why := HeapAllocReason(loc.n); why != "" {
			b.flow(b.heapHole().addr(loc.n, why), loc)
		}
	}

	b.walkAll()
	b.finish(fns)
}
```
base.Flag.W это просто флаг для дебага
fn.IsClosure() - функция проверяющая что функция литерал(анонимная функция) захватила переменную вне своего окружения
пример
```go
func f() func() {
    x := 10
    return func() {
        println(x)
    }
}
```
fn.Dcl - содержит список oname узлов
	PPARAM - входные параметры функции
	PPARAMOUT - именованые возвращаемые значения
	PAUTO - локальные переменные функции
	
attrEscapes указывает нужно ли выделять память в куче для переменной
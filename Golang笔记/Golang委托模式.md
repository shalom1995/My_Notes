# Golang 委托模式

委托模式

```golang
type Widget struct {
	X, Y int
}

type Label struct {
	Widget        // Embedding (delegation)
	Text   string // Aggregation
	X int         // Override
}

func (label Label) Paint() {
	// [0xc4200141e0] - Label.Paint("State")
	fmt.Printf("[%p] - Label.Paint(%q)\n",
		&label, label.Text)
}


type Button struct {
	Label // Embedding (delegation)
}

func NewButton(x, y int, text string) Button {
	return Button{Label{Widget{x, y}, text, x}}
}
func (button Button) Paint() { // Override
	fmt.Printf("[%p] - Button.Paint(%q)\n",
		&button, button.Text)
}
func (button Button) Click() {
	fmt.Printf("[%p] - Button.Click()\n", &button)
}



type ListBox struct {
	Widget          // Embedding (delegation)
	Texts  []string // Aggregation
	Index  int      // Aggregation
}
func (listBox ListBox) Paint() {
	fmt.Printf("[%p] - ListBox.Paint(%q)\n",
		&listBox, listBox.Texts)
}
func (listBox ListBox) Click() {
	fmt.Printf("[%p] - ListBox.Click()\n", &listBox)
}


type Painter interface {
	Paint()
}

type Clicker interface {
	Click()
}

func Tem() {

	label := Label{Widget{10, 10}, "State", 100}

	// X=100, Y=10, Text=State, Widget.X=10
	fmt.Printf("X=%d, Y=%d, Text=%s Widget.X=%d\n",
		label.X, label.Y, label.Text,
		label.Widget.X)
	fmt.Println()
	// {Widget:{X:10 Y:10} Text:State X:100}
	// {{10 10} State 100}
	fmt.Printf("%+v\n%v\n", label, label)

	label.Paint()


	button1 := Button{Label{Widget{10, 70}, "OK", 10}}
	button2 := NewButton(50, 70, "Cancel")
	listBox := ListBox{Widget{10, 40},
		[]string{"AL", "AK", "AZ", "AR"}, 0}

	fmt.Println()
	//[0xc4200142d0] - Label.Paint("State")
	//[0xc420014300] - ListBox.Paint(["AL" "AK" "AZ" "AR"])
	//[0xc420014330] - Button.Paint("OK")
	//[0xc420014360] - Button.Paint("Cancel")
	for _, painter := range []Painter{label, listBox, button1, button2} {
		painter.Paint()
	}

	fmt.Println()
	//[0xc420014450] - ListBox.Click()
	//[0xc420014480] - Button.Click()
	//[0xc4200144b0] - Button.Click()
	for _, widget := range []interface{}{label, listBox, button1, button2} {
		if clicker, ok := widget.(Clicker); ok {
			clicker.Click()
		}
	}
}
```

Undo示例

```golang
type IntSet struct {
	data map[int]bool
}

func NewIntSet() IntSet {
	return IntSet{make(map[int]bool)}
}

func (set *IntSet) Add(x int) {
	set.data[x] = true
}

func (set *IntSet) Delete(x int) {
	delete(set.data, x)
}

func (set *IntSet) Contains(x int) bool {
	return set.data[x]
}

func (set *IntSet) String() string { // Satisfies fmt.Stringer interface
	if len(set.data) == 0 {
		return "{}"
	}
	ints := make([]int, 0, len(set.data))
	for i := range set.data {
		ints = append(ints, i)
	}
	sort.Ints(ints)
	parts := make([]string, 0, len(ints))
	for _, i := range ints {
		parts = append(parts, fmt.Sprint(i))
	}
	return "{" + strings.Join(parts, ",") + "}"
}

type UndoableIntSet struct { // Poor style
	IntSet    // Embedding (delegation)
	functions []func()
}

func NewUndoableIntSet() UndoableIntSet {
	return UndoableIntSet{NewIntSet(), nil}
}

func (set *UndoableIntSet) Add(x int) { // Override
	if !set.Contains(x) {
		set.data[x] = true
		set.functions = append(set.functions, func() { set.Delete(x) })
	} else {
		set.functions = append(set.functions, nil)
	}
}

func (set *UndoableIntSet) Delete(x int) { // Override
	if set.Contains(x) {
		delete(set.data, x)
		set.functions = append(set.functions, func() { set.Add(x) })
	} else {
		set.functions = append(set.functions, nil)
	}
}

func (set *UndoableIntSet) Undo() error {
	if len(set.functions) == 0 {
		return errors.New("No functions to undo")
	}
	index := len(set.functions) - 1
	if function := set.functions[index]; function != nil {
		function()
		set.functions[index] = nil // Free closure for garbage collection
	}
	set.functions = set.functions[:index]
	return nil
}

func useIntSet() {

	/*ints := NewIntSet()
	for _, i := range []int{1, 3, 5, 7} {
		ints.Add(i)
		fmt.Println(ints)
	}
	for _, i := range []int{1, 2, 3, 4, 5, 6, 7} {
		fmt.Print(i, ints.Contains(i), " ")
		ints.Delete(i)
		fmt.Println(ints)
	}*/



	ints := NewUndoableIntSet()
	for _, i := range []int{1, 3, 5, 7} {
		ints.Add(i)
		fmt.Println(ints)
	}
	for _, i := range []int{1, 2, 3, 4, 5, 6, 7} {
		fmt.Println(i, ints.Contains(i), " ")
		ints.Delete(i)
		fmt.Println(ints)
	}
	fmt.Println()
	for {
		if err := ints.Undo(); err != nil {
			break
		}
		fmt.Println(ints)
	}
}
```

进化版Undo

```golang
type Undo []func()

func (undo *Undo) Add(function func()) {
	*undo = append(*undo, function)
}

func (undo *Undo) Undo() error {
	functions := *undo
	if len(functions) == 0 {
		return errors.New("No functions to undo")
	}
	index := len(functions) - 1
	if function := functions[index]; function != nil {
		function()
		functions[index] = nil // Free closure for garbage collection
	}
	*undo = functions[:index]
	return nil
}


type IntSet struct {
	data map[int]bool
	undo Undo
}

func NewIntSet() IntSet {
	return IntSet{data: make(map[int]bool)}
}

func (set *IntSet) Add(x int) {
	if !set.Contains(x) {
		set.data[x] = true
		set.undo.Add(func() { set.Delete(x) })
	} else {
		set.undo.Add(nil)
	}
}

func (set *IntSet) Delete(x int) {
	if set.Contains(x) {
		delete(set.data, x)
		set.undo.Add(func() { set.Add(x) })
	} else {
		set.undo.Add(nil)
	}
}

func (set *IntSet) Undo() error {
	return set.undo.Undo()
}

func (set *IntSet) Contains(x int) bool {
	return set.data[x]
}
```
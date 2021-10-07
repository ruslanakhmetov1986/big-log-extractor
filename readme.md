# Парсинг большего текстого файла в GO

Источник: https://medium.com/swlh/processing-16gb-file-in-seconds-go-lang-3982c235dfa2

Любая компьютерная система в современном мире ежедневно генерирует очень большое количество журналов или данных. По мере роста системы невозможно сохранять данные отладки в базе данных, поскольку они неизменны и будут использоваться только для целей аналитики и устранения ошибок. Поэтому организации обычно хранят его в файлах, которые находятся в хранилище локальных дисков.

*Мы собираемся извлечь журналы из файла .txt или .log размером 16 ГБ, содержащего миллионы строк, используя Golang.*

## Погнали…!

Давайте сначала откроем файл. Мы будем использовать стандартный Go ***os.File\*** для любого файлового ввода-вывода.

```go
f, err := os.Open(fileName)
 if err != nil {
   fmt.Println("cannot able to read the file", err)
   return
 }
// UPDATE: close after checking error
defer file.Close()  //Do not forget to close the file
```

После открытия файла у нас есть два варианта, чтобы продолжить

1. Читайте файл построчно, это помогает снизить нагрузку на память, но требует больше времени при вводе-выводе.
2. Прочтите сразу весь файл в память и обработайте файл, что потребует больше памяти, но значительно увеличит время.

Поскольку у нас слишком большой размер файла, то есть 16 ГБ, мы не можем загрузить в память весь файл. Но и первый вариант нам не подходит, так как мы хотим обработать файл за секунды.

Но знаете что, есть третий вариант. Вуаля…! Вместо того, чтобы загружать весь файл в память, мы будем загружать файл по ***частям\*** , используя ***bufio.NewReader (),\*** доступный *в Go.*

```go
r := bufio.NewReader(f)
for {
buf := make([]byte,4*1024) //the chunk size
n, err := r.Read(buf) //loading chunk into buffer
   buf = buf[:n]
if n == 0 {
   
     if err != nil {
       fmt.Println(err)
       break
     }
     if err == io.EOF {
       break
     }
     return err
  }
}
```

Когда у нас есть чанк, мы создадим ветвь потока, то есть подпрограмму Go, чтобы обрабатывать каждый чанк одновременно с другими чанками. Приведенный выше код будет изменен на -

```go
//sync pools to reuse the memory and decrease the preassure on //Garbage Collector
linesPool := sync.Pool{New: func() interface{} {
        lines := make([]byte, 500*1024)
        return lines
}}
stringPool := sync.Pool{New: func() interface{} {
          lines := ""
          return lines
}}
slicePool := sync.Pool{New: func() interface{} {
           lines := make([]string, 100)
           return lines
}}
r := bufio.NewReader(f)
var wg sync.WaitGroup //wait group to keep track off all threads
for {
     
     buf := linesPool.Get().([]byte)
     n, err := r.Read(buf)
     buf = buf[:n]
if n == 0 {
        if err != nil {
            fmt.Println(err)
            break
        }
        if err == io.EOF {
            break
        }
        return err
     }
nextUntillNewline, err := r.ReadBytes('\n')//read entire line
     
     if err != io.EOF {
         buf = append(buf, nextUntillNewline...)
     }
     
     wg.Add(1)
     go func() { 
      
        //process each chunk concurrently
        //start -> log start time, end -> log end time
        
        ProcessChunk(buf, &linesPool, &stringPool, &slicePool,     start, end)
wg.Done()
     
     }()
}
wg.Wait()
}
```

В приведенном выше коде представлены две новые оптимизации: -

1. **Sync.Pool** мощный пул экземпляров , которые могут быть повторно использованы , чтобы уменьшить давление на сборщик мусора. Мы будем восстанавливать память, выделенную для различных срезов. Это помогает нам снизить потребление памяти и значительно ускорить нашу работу.
2. В **Go Подпрограмма** , который помогает нам обрабатывать буферный кусок одновременно , что значительно увеличивает скорость обработки.

Теперь давайте реализуем функцию **ProcessChunk** , которая будет обрабатывать строки журнала, которые имеют формат

```
2020-01-31T20:12:38.1234Z, Some Field, Other Field, And so on, Till new line,...\n
```

Мы будем извлекать журналы на основе отметки времени, указанной в командной строке.

```go
func ProcessChunk(chunk []byte, linesPool *sync.Pool, stringPool *sync.Pool, slicePool *sync.Pool, start time.Time, end time.Time) {
//another wait group to process every chunk further                             
      var wg2 sync.WaitGroup
logs := stringPool.Get().(string)
logs = string(chunk)
linesPool.Put(chunk) //put back the chunk in pool
//split the string by "\n", so that we have slice of logs
      logsSlice := strings.Split(logs, "\n")
stringPool.Put(logs) //put back the string pool
chunkSize := 100 //process the bunch of 100 logs in thread
n := len(logsSlice)
noOfThread := n / chunkSize
if n%chunkSize != 0 { //check for overflow 
         noOfThread++
      }
length := len(logsSlice)
//traverse the chunk
     for i := 0; i < length; i += chunkSize {
         
         wg2.Add(1)
//process each chunk in saperate chunk
         go func(s int, e int) {
            for i:= s; i<e;i++{
               text := logsSlice[i]
if len(text) == 0 {
                  continue
               }
           
            logParts := strings.SplitN(text, ",", 2)
            logCreationTimeString := logParts[0]
            logCreationTime, err := time.Parse("2006-01-  02T15:04:05.0000Z", logCreationTimeString)
if err != nil {
                 fmt.Printf("\n Could not able to parse the time :%s       for log : %v", logCreationTimeString, text)
                 return
            }
// check if log's timestamp is inbetween our desired period
          if logCreationTime.After(start) && logCreationTime.Before(end) {
          
            fmt.Println(text)
           }
        }
        textSlice = nil
        wg2.Done()
     
     }(i*chunkSize, int(math.Min(float64((i+1)*chunkSize), float64(len(logsSlice)))))
   //passing the indexes for processing
}  
   wg2.Wait() //wait for a chunk to finish
   logsSlice = nil
}
```

Приведенный выше код протестирован с использованием файла журнала объемом 16 ГБ.

Время, необходимое для извлечения журналов, составляет около 25 секунд.



## Полный код



```go
//wertyuio

func main() {

	s := time.Now()
	args := os.Args[1:]
	if len(args) != 6 { // for format  LogExtractor.exe -f "From Time" -t "To Time" -i "Log file directory location"
		fmt.Println("Please give proper command line arguments")
		return
	}
	startTimeArg := args[1]
	finishTimeArg := args[3]
	fileName := args[5]

	file, err := os.Open(fileName)
	
	if err != nil {
		fmt.Println("cannot able to read the file", err)
		return
	}
	
	defer file.Close() //close after checking err
	
	queryStartTime, err := time.Parse("2006-01-02T15:04:05.0000Z", startTimeArg)
	if err != nil {
		fmt.Println("Could not able to parse the start time", startTimeArg)
		return
	}

	queryFinishTime, err := time.Parse("2006-01-02T15:04:05.0000Z", finishTimeArg)
	if err != nil {
		fmt.Println("Could not able to parse the finish time", finishTimeArg)
		return
	}

	filestat, err := file.Stat()
	if err != nil {
		fmt.Println("Could not able to get the file stat")
		return
	}

	fileSize := filestat.Size()
	offset := fileSize - 1
	lastLineSize := 0

	for {
		b := make([]byte, 1)
		n, err := file.ReadAt(b, offset)
		if err != nil {
			fmt.Println("Error reading file ", err)
			break
		}
		char := string(b[0])
		if char == "\n" {
			break
		}
		offset--
		lastLineSize += n
	}

	lastLine := make([]byte, lastLineSize)
	_, err = file.ReadAt(lastLine, offset+1)

	if err != nil {
		fmt.Println("Could not able to read last line with offset", offset, "and lastline size", lastLineSize)
		return
	}

	logSlice := strings.SplitN(string(lastLine), ",", 2)
	logCreationTimeString := logSlice[0]

	lastLogCreationTime, err := time.Parse("2006-01-02T15:04:05.0000Z", logCreationTimeString)
	if err != nil {
		fmt.Println("can not able to parse time : ", err)
	}

	if lastLogCreationTime.After(queryStartTime) && lastLogCreationTime.Before(queryFinishTime) {
		Process(file, queryStartTime, queryFinishTime)
	}

	fmt.Println("\nTime taken - ", time.Since(s))
}

func Process(f *os.File, start time.Time, end time.Time) error {

	linesPool := sync.Pool{New: func() interface{} {
		lines := make([]byte, 250*1024)
		return lines
	}}

	stringPool := sync.Pool{New: func() interface{} {
		lines := ""
		return lines
	}}

	r := bufio.NewReader(f)

	var wg sync.WaitGroup

	for {
		buf := linesPool.Get().([]byte)

		n, err := r.Read(buf)
		buf = buf[:n]

		if n == 0 {
			if err != nil {
				fmt.Println(err)
				break
			}
			if err == io.EOF {
				break
			}
			return err
		}

		nextUntillNewline, err := r.ReadBytes('\n')

		if err != io.EOF {
			buf = append(buf, nextUntillNewline...)
		}

		wg.Add(1)
		go func() {
			ProcessChunk(buf, &linesPool, &stringPool, start, end)
			wg.Done()
		}()

	}

	wg.Wait()
	return nil
}

func ProcessChunk(chunk []byte, linesPool *sync.Pool, stringPool *sync.Pool, start time.Time, end time.Time) {

	var wg2 sync.WaitGroup

	logs := stringPool.Get().(string)
	logs = string(chunk)

	linesPool.Put(chunk)

	logsSlice := strings.Split(logs, "\n")

	stringPool.Put(logs)

	chunkSize := 300
	n := len(logsSlice)
	noOfThread := n / chunkSize

	if n%chunkSize != 0 {
		noOfThread++
	}

	for i := 0; i < (noOfThread); i++ {

		wg2.Add(1)
		go func(s int, e int) {
			defer wg2.Done() //to avaoid deadlocks
			for i := s; i < e; i++ {
				text := logsSlice[i]
				if len(text) == 0 {
					continue
				}
				logSlice := strings.SplitN(text, ",", 2)
				logCreationTimeString := logSlice[0]

				logCreationTime, err := time.Parse("2006-01-02T15:04:05.0000Z", logCreationTimeString)
				if err != nil {
					fmt.Printf("\n Could not able to parse the time :%s for log : %v", logCreationTimeString, text)
					return
				}

				if logCreationTime.After(start) && logCreationTime.Before(end) {
					//fmt.Println(text)
				}
			}
			

		}(i*chunkSize, int(math.Min(float64((i+1)*chunkSize), float64(len(logsSlice)))))
	}

	wg2.Wait()
	logsSlice = nil
}
```


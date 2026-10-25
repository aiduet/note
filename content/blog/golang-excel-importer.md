+++
title = "A small Go tool to import Excel"
date = "2021-05-12"
slug = "golang-excel-importer"
tags = ["notes"]
+++

Need: import Excel files into MongoDB and MySQL in batches.

Work was in Go, so I wrote this in Go too. Colleagues can change it.

I called it `ecc`. Layout:

```
.
|-- cmd
|   `-- ecc.go
|-- configs
|   |-- cfg.go
|   `-- cfg.yaml
|-- data
|-- internal
|   `-- importing
|-- pkg
|   |-- files
|   |-- mongo
|   `-- mysql
|-- tools
|    `-- print.go
|-- go.mod
|-- go.sum
|-- LICENSE
|-- README.en.md
`-- README.md
```

### ecc.go

The CLI uses [github.com/urfave/cli/v2](https://github.com/urfave/cli/v2).

Main flag: `dir` — the folder of Excel files.

```
func main() {

	var err error
	var model string
	dir := DirPath
	app := &cli.App{
		Name:  "Ecc",
		Usage: "Ecc is a tools for batch processing of excel data",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:        "model",
				Aliases:     []string{"m"},
				Usage:       "The model of searching",
				Value:       "model",
				Destination: &model,
			},
			&cli.StringFlag{
				Name:        "dir",
				Aliases:     []string{"d"},
				Usage:       "Folder location of data files",
				Destination: &dir,
				Value:       DirPath,
			},
		},
		Action: func(c *cli.Context) error {
			importing.Load("../configs/cfg.yaml")
			importing.Handle(dir)
			return nil
		},
	}

	err = app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

Flags set options. Action runs the job. It loads the DB config first.

I only used MySQL and MongoDB. Redis for cache was dropped.

### cfg.go

```
type Config struct {
	Env   string `yaml:"env"`
	Mongo struct {
		DNS        string `yaml:"dns"`
		Db         string `yaml:"db"`
		Collection string `yaml:"collection"`
	} `yaml:"mongo"`
	Mysql struct {
		Alias string `yaml:"alias"`
		Dns   string `yaml:"dns"`
	} `yaml:"mysql"`
}
```

Config is read with [github.com/spf13/viper](https://github.com/spf13/viper).

The core is `importing.Handle()`.

A common path: one goroutine per file to read, then write in another — or write each row as you read.

Here every run must check **all** Excel files in the folder first. If one file is bad, stop.

Two choices:

1. Read and write at the same time. On error, roll back and kill all goroutines.
2. Read everything into a map. If it is clean, write to the DB.

I picked (2). The set is small. The files (crawled data) are often messy.

### handle()

```
var (
	rWait   = true
	wWait   = true
	rDone   = make(chan struct{})
	rCrash  = make(chan struct{})
	wDone   = make(chan struct{})
	wCrash  = make(chan struct{})
	once    = &sync.Once{}
	wg      = &sync.WaitGroup{}
        // command-line progress bar
	pb      = mpb.New(mpb.WithWaitGroup(wg), mpb.WithWidth(ProcessBarWidth)) 
)

...

func Handle(dir string) {
	var err error
	var f []os.FileInfo
	var data = &sync.Map{}

	if f, err = files.ReadDir(dir); err != nil {
		abort("-> Failure: " + err.Error())
		return
	}

	read(f, dir, data)
	for rWait {
		select {
		case <-rCrash:
			abort("-> Failure")
			return
		case <-rDone:
			rWait = false
		}
	}

	write2mongo(data)
	for wWait {
		select {
		case <-wCrash:
			abort("-> Failure")
			return
		case <-wDone:
			wWait = false
		}
	}

	pb.Wait()

	tools.Yellow("-> Whether to sync data to mysql? (y/n)")
	if !tools.Scan("aborted") {
		return
	} else {
		tools.Yellow("-> Syncing data to mysql...")
		if err = write2mysql(); err != nil {
			tools.Red("-> Failure：" + err.Error())
		} else {
			tools.Green("-> Success.")
		}
	}
}
```

`rCrash` means a check failed. Close it to stop all readers. `rDone` means all files were read. Write uses the same idea.

### read()

```
func read(fs []os.FileInfo, dir string, data *sync.Map) {
	for _, file := range fs {
		fileName := file.Name()
		_ext := filepath.Ext(fileName)
		if Include(strings.Split(Exts, ","), _ext) {
			wg.Add(1)
			inCh := make(chan File)
			go func() {
				defer wg.Done()
				select {
				case <-rCrash:
					return // exit
				case f := <-inCh:
					e, preData := ReadExcel(f.FilePath, f.FileName, pb)
					if e != nil {
						tools.Red("%v", e)
						once.Do(func() {
							close(rCrash)
						})
						return
					}
					data.Store(f.FileName, preData)
				}
			}()
			go func() {
				inCh <- File{
					FileName: fileName,
					FilePath: dir + string(os.PathSeparator) + fileName,
				}
			}()
		}
	}

	go func() {
		wg.Wait()
		close(rDone)
	}()
}
```

Send file info into `inCh`. `ReadExcel()` checks and parses. On error, close `rCrash`. If it is fine, store it in `sync.Map`.

Excel is read with [github.com/xuri/excelize/v2](https://github.com/xuri/excelize/v2):

### read_excel()

```
func ReadExcel(filePath, fileName string, pb *mpb.Progress) (err error, pre *ExcelPre) {
	f, err := excelize.OpenFile(filePath)
	if err != nil {
		return err, nil
	}

	defer func() {
		if _e := f.Close(); _e != nil {
			fmt.Printf("%s: %v.\n\n", filePath, _e)
		}
	}()

	// Get the first sheet.
	firstSheet := f.WorkBook.Sheets.Sheet[0].Name
	rows, err := f.GetRows(firstSheet)
	lRows := len(rows)

	if lRows < 2 {
		lRows = 2
	}

	rb := ReadBar(lRows, filePath, pb)
	wb := WriteBar(lRows-2, filePath, rb, pb)

	// The first line is the field name.
	var fields []string

	// The data of the file.
	var data [][]string

	InCr := func(start time.Time) {
		rb.Increment()
		rb.DecoratorEwmaUpdate(time.Since(start))
	}

	for i := 0; i < lRows; i++ {
		InCr(time.Now())
		if i == 0 {
			fields = rows[i]
			for index, field := range fields {
				if isChinese := regexp.MustCompile("[\u4e00-\u9fa5]"); isChinese.MatchString(field) || field == "" {
					err = errors.New(fmt.Sprintf("%s: line 【A%d】 field 【%s】 \n", filePath, index, field) + "The first line of the file is not a valid attribute name.")
					return err, nil
				}

				// other rules
			}
			continue
		}

		if i == 1 {
			continue
		}

		data = append(data, rows[i])
	}

	return nil, &ExcelPre{
		FileName:    fileName,
		Data:        data,
		Fields:      fields,
		Prefixes:    Prefix(fileName),
		ProgressBar: wb,
	}
}
```

This also drives the CLI progress bar, plus a few rules. If there are many rules, make an interface and compose them.

The rest is normal DB work. I skip that code.

A run screenshot is lost.

*Update, Jan 2022: the job got more custom. We moved it to a web app.*

[#excel](/blog/?q=excel)
                
                    [#go](/blog/?q=go)
                
                    [#notes](/blog/?q=notes)

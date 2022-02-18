<!-- ---
title: robfig cron
date: 2019-03-08 15:19:28
category: src, cron
--- -->

robfig cron


## 目录结构

```
cron.go //主代码
parser.go //定时任务时间解析
spec.go //任务间隔时间解析
constantdelay.go
```


## cron.go

代码主逻辑

- Cron struct
- Entry struct
- AddJob 添加任务
- Schedule 将job 添加到cron 中
- Run 运行定时任务job

```go
type Cron struct {
	entries  []*Entry
	stop     chan struct{}
	add      chan *Entry
	snapshot chan []*Entry
	running  bool
	ErrorLog *log.Logger
	location *time.Location
}

// Job interface
type Job interface {
	Run()
}

// Entry consists of a schedule and the func to execute on that schedule.
type Entry struct {
	// The schedule on which this job should be run.
	Schedule Schedule

	// The next time the job will run. This is the zero time if Cron has not been
	// started or this entry's schedule is unsatisfiable
	Next time.Time

	// The Job to run.
	Job Job
}


// AddJob 添加任务
func (c *Cron) AddJob(spec string, cmd Job) error {
	//解析任务时间
    schedule, err := Parse(spec)
	if err != nil {
		return err
	}
	c.Schedule(schedule, cmd)
	return nil
}

// Schedule 将job 添加到cron 中
func (c *Cron) Schedule(schedule Schedule, cmd Job) {
	entry := &Entry{
		Schedule: schedule,
		Job:      cmd,
	}
	if !c.running {
		c.entries = append(c.entries, entry)
		return
	}

	c.add <- entry
}


// Run 运行定时任务实体
func (c *Cron) run() {
	// 计算出各job 下次运行时间
	now := c.now()
	for _, entry := range c.entries {
		entry.Next = entry.Schedule.Next(now)
	}

	for {
		// 对定时任务按照下次运行时间进行排序
		sort.Sort(byTime(c.entries))

		var timer *time.Timer
        //如果当前没有定时任务则等待
		if len(c.entries) == 0 || c.entries[0].Next.IsZero() {
			// If there are no entries yet, just sleep - it still handles new entries
			// and stop requests.
			timer = time.NewTimer(100000 * time.Hour)
		} else {
            //以最小时间间隔设置时间触发器
			timer = time.NewTimer(c.entries[0].Next.Sub(now))
		}

		for {
			select {
			case now = <-timer.C: //等下下次运行所有job
				now = now.In(c.location)
				// Run every entry whose next time was less than now
				for _, e := range c.entries {
                    //如果任务执行时间在当前时间之后，说明还没到开始执行时间，由于有排序，所以可以直接跳出了
					if e.Next.After(now) || e.Next.IsZero() {
						break
					}
					go c.runWithRecovery(e.Job) //运行任务
					e.Prev = e.Next
					e.Next = e.Schedule.Next(now) //运行完成后计算下次开始运行时间
				}

			case newEntry := <-c.add: //添加job 处理
				//...
			case <-c.snapshot: //获取当前所有job
				//...
			case <-c.stop: //停止运行
				//...
			}
			break
		}
	}
}
```



## 使用示例

[golib_cron.md](/language/go/library/golib_cron.md)



## 参考资料

- [github.com/robfig/cron](https://github.com/robfig/cron)


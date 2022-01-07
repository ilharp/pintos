# Project 1: Threads

## Task 1: Efficient Alarm Clock

### 数据结构

在 `struct thread` 中新增的字段：

```c
int64_t ticks_blocked;              /* 线程阻塞的时间长度。 */
```

### 算法

Task 1 围绕 `timer_sleep` 的重新实现展开。

我们直接来看函数的重新实现：

```c
// timer.c
void
timer_sleep (int64_t ticks)
{
  if (ticks <= 0) return; // 拒绝不合法的 timer_sleep 请求
  ASSERT (intr_get_level () == INTR_ON);
  enum intr_level old_level = intr_disable (); // 禁止中断
  struct thread *current_thread = thread_current (); // 获取当前线程
  current_thread->ticks_blocked = ticks; // 设置线程阻塞的时间长度
  thread_block (); // 休眠当前线程
  intr_set_level (old_level); // 恢复中断
}
```

`timer_sleep()` 重新实现了线程的休眠请求。在

```c
enum intr_level old_level = intr_disable ();
```

和

```c
intr_set_level (old_level);
```

两句语句之间的内容不会受到中断影响，可保证其内语句执行的同步和原子化。

`thread_current ()` 首先获取了当前运行的线程并将我们刚刚新增的 `ticks_blocked`
字段设置为需要阻塞的时间长度；之后便调用 `thread_block ()` 方法休眠当前线程。

---

而保证正确倒计时，并使线程能够正确恢复的方法则是下面的 `check_blocked()`：

```c
// thread.c

/* 检查线程阻塞。 */
void
check_blocked (struct thread *t, void *aux UNUSED)
{
  if (t->status == THREAD_BLOCKED && t->ticks_blocked > 0) // 检查线程阻塞
  {
      /* 若线程阻塞，则更新计时 */
      t->ticks_blocked--;
      if (t->ticks_blocked == 0)
      {
          /* 若计时器已归零，则执行 thread_unblock(t) 以恢复线程 */
          thread_unblock (t);
      }
  }
}
```

`check_blocked()` 由 `timer_interrupt()` 调用，
使得 `ticks_blocked` 在 `timer_interrupt()` 单位周期内递减，
最终在 `ticks_blocked` 归零时调用 `thread_unblock (t)` 以恢复线程。

### 线程同步

线程同步由上面所提出的中断禁用语句实现。

在中断禁用语句内执行的语句不会受中断影响，因而可以实现同步和原子化。

## Task 2: Priority Scheduler

### 数据结构

在 `struct thread` 中新增的字段：

```c
int base_priority;                  /* 线程基础优先级。 */
struct list locks;                  /* 线程当前持有的锁。 */
struct lock *lock_waiting;          /* 线程正在等待的锁。 */
```

### 算法

要实现优先级调度，首先要做的就是实现对线程优先级的排序。

与一般的考虑不同，我们不将「线程排序」的操作抽象化，
并创建一个单独的逻辑环境供系统专门进行线程排序；
我们将该操作放在每次对线程池进行修改的同时进行。

首先，实现最基本的线程优先级比较算法，
此处便使用了线程的 `priority` 字段：

```c
bool
thread_priority_compare (const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)
{
  return list_entry(a, struct thread, elem)->priority > list_entry(b, struct thread, elem)->priority;
}
```

接着，将现有代码中对线程池进行操作的部分进行替换，
使之在完成自己操作的同时能够对线程池中的线程进行优先级排序。

幸运地，pintos 提供了一个工具方法 `list_insert_ordered()` 让我们完成这项操作。

将代码修改为类似

```c
list_insert_ordered (&ready_list, &t->elem, (list_less_func *) &thread_priority_compare, NULL);
```

的操作，线程排序的任务就会同时执行了。

将多个对线程池进行了操作的方法都检查一遍，并进行相应的修改。

此处必须对逻辑层面所有可能导致线程池中优先级变动的操作都考虑到。举个例子：
`thread_create()` 方法就需要增加如下代码：

```c
thread_yield ();
```

类似的地方还有很多，具体可以查阅 `thread.c` 文件。

---

线程优先级排序完成之后，接下来就可以根据优先级字段进行线程调度了。

首先实现一些工具代码，如优先级更新：

```c
void
thread_update_priority (struct thread *t)
{
  enum intr_level old_level = intr_disable (); // 禁用中断
  int max_priority = t->base_priority;
  int lock_priority;

  if (!list_empty (&t->locks))
  {
    list_sort (&t->locks, lock_priority_compare, NULL);
    lock_priority = list_entry (list_front (&t->locks), struct lock, elem)->max_priority;
    if (lock_priority > max_priority)
      max_priority = lock_priority; // 计算最大优先级
  }

  t->priority = max_priority; // 重新设置优先级为最大优先级
  intr_set_level (old_level); // 恢复中断
}
```

和优先级捐赠：

```c
void
thread_donate_priority (struct thread *t)
{
  enum intr_level old_level = intr_disable (); // 禁用中断
  thread_update_priority (t); // 优先级更新

  if (t->status == THREAD_READY) // 判断线程状态
  {
    list_remove (&t->elem);
    list_insert_ordered (&ready_list, &t->elem, thread_priority_compare, NULL);
  }
  intr_set_level (old_level); // 恢复中断
}
```

此处代码的作用和操作已经用注释标明。

---

接着，在必要的操作处添加优先级调度的操作。

在 `synch.c` 的 `sema_up()` 方法中加入：

```c
list_sort (&sema->waiters, thread_priority_compare, NULL);
```

操作完成后：

```c
thread_yield ();
```

修改 `lock_acquire()` 方法：

```c
void
lock_acquire (struct lock *lock)
{
  struct thread *current_thread = thread_current ();

  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if (lock->holder != NULL && !thread_mlfqs)
  {
    current_thread->lock_waiting = lock;
    struct lock *l = lock;

    // 递归实现优先级捐赠
    while (l && current_thread->priority > l->max_priority)
    {
      l->max_priority = current_thread->priority;
      thread_donate_priority (l->holder); // 进行优先级捐赠
      l = l->holder->lock_waiting;
    }
  }

  sema_down (&lock->semaphore);

  enum intr_level old_level = intr_disable (); // 禁止中断

  current_thread = thread_current ();
  if (!thread_mlfqs)
  {
    current_thread->lock_waiting = NULL;
    lock->max_priority = current_thread->priority;
    thread_hold_lock (lock);
  }
  lock->holder = current_thread;

  intr_set_level (old_level); // 释放中断状态
}
```

此处几个比较关键的地方：

第一，优先级捐赠的实现。

```c
while (l && current_thread->priority > l->max_priority)
{
  l->max_priority = current_thread->priority;
  thread_donate_priority (l->holder); // 进行优先级捐赠
  l = l->holder->lock_waiting;
}
```

这里的 `l` 即为数据结构中新增的 `lock_waiting`
指针字段，在此处用作递归指针。递归通过

```c
l = l->holder->lock_waiting;
```

实现。

第二，注意此处对于 `thread_mlfqs` 变量的判断。
现阶段进行的所有操作都不适用于 `thread_mlfqs`
（即 4.4BSD Scheduler），因此此处使用 if 判断绕过。
下一部分具体阐述 4.4BSD Scheduler。

此外，还需要实现几个优先级的比较函数，此处不再赘述。

---

最后，实现对外接口 `thread_set_priority()`：

```c
void
thread_set_priority (int new_priority)
{
  if (thread_mlfqs)
    return;

  enum intr_level old_level = intr_disable (); // 禁止中断

  struct thread *current_thread = thread_current ();
  int old_priority = current_thread->priority;
  current_thread->base_priority = new_priority; // 设置线程基础优先级为新的优先级

  if (list_empty (&current_thread->locks) || new_priority > old_priority)
  {
    current_thread->priority = new_priority;
    thread_yield ();
  }

  intr_set_level (old_level); // 释放中断状态
}
```

可以看出在设置了线程优先级之后，还执行了 `thread_yield ()`
方法。该方法正是优先级调度的开端。

测试全部通过。程序正常工作。

### 线程同步

线程同步的方式和上一部分一样，仍然是在同步操作处禁用中断。

## Task 3: 4.4BSD Scheduler

### 数据结构

在 `struct thread` 中新增的字段：

```c
int nice;                           /* Niceness. */
fixed_point_t recent_cpu;           /* Recent CPU. */
```

### 算法

首先编写工具方法。涉及的工具方法主要有以下几例：

```c
/* 自增 recent_cpu。 */
void
thread_mlfqs_increase_recent_cpu (void)
{
  ASSERT (thread_mlfqs);
  ASSERT (intr_context ());

  // 获取当前线程。
  struct thread *current_thread = thread_current ();

  // 确保线程非等待。
  if (current_thread == idle_thread)
    return;

  // 自增 recent_cpu。
  current_thread->recent_cpu = fix_add (current_thread->recent_cpu, fix_int (1));
}
```

该方法自增 recent_cpu。

```c
/* 更新线程优先级。 */
void
thread_mlfqs_update_priority (struct thread *t)
{
  // 确保线程非等待。
  if (t == idle_thread)
    return;

  ASSERT (thread_mlfqs);
  ASSERT (t != idle_thread);

  // 计算线程优先级。
  // 公式：
  // priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)
  t->priority = fix_trunc (fix_sub (fix_sub (fix_int (PRI_MAX), fix_div (t->recent_cpu, fix_int (4))), fix_int (2 * t->nice)));

  // 限制线程优先级范围。
  t->priority = t->priority < PRI_MIN ? PRI_MIN : t->priority;
  t->priority = t->priority > PRI_MAX ? PRI_MAX : t->priority;
}
```

该方法是 4.4BSD Scheduler 的主要内容，即线程优先级的调度。
4.4BSD Scheduler 的优先级调度公式已经在 pintos 文档中给出。

值得一提的是，上述定点数的计算使用了 `fixed-point.h` 中的相关工具函数。

`fixed-point.h` 通过宏展开方式进行定点数的运算，使得性能最大化。

此外，同样为了性能最大化，`fixed-point.h` 使用 `int` 存储数据类型。

在文件开头的宏定义中：

```c
#define FIX_BITS 32             /* Total bits per fixed-point number. */
#define FIX_P 16                /* Number of integer bits. */
#define FIX_Q 16                /* Number of fractional bits. */
```

定义了一个 16-16 的定点数数据类型。

```c
/* 更新 load_avg 和 recent_cpu。 */
void
thread_mlfqs_update_load (void)
{
  ASSERT (thread_mlfqs);
  ASSERT (intr_context ());

  // 计算就绪线程的个数。
  size_t ready_threads = list_size (&ready_list);

  // 确保线程非等待。
  if (thread_current () != idle_thread)
    ready_threads++;

  // 计算 load_avg。
  // 公式：
  // load_avg = (59/60)*load_avg + (1/60)*ready_threads
  load_avg = fix_add (fix_div (fix_mul (load_avg, fix_int (59)), fix_int (60)), fix_div (fix_int (ready_threads), fix_int (60)));

  struct thread *t;
  struct list_elem *e = list_begin (&all_list);
  for (; e != list_end (&all_list); e = list_next (e))
  {
    t = list_entry(e, struct thread, allelem);
    if (t != idle_thread)
    {
      // 计算 recent_cpu。
      t->recent_cpu = fix_add (fix_mul (fix_div (fix_mul (load_avg, fix_int (2)), fix_add (fix_mul (load_avg, fix_int (2)), fix_int (1))), t->recent_cpu), fix_int (t->nice));
      thread_mlfqs_update_priority (t);
    }
  }
}
```

最后一个工具方法负责更新 load_avg 和 recent_cpu。

---

接着，重写 `timer_interrupt()` 方法：

```c
/* Timer interrupt handler. */
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  enum intr_level old_level = intr_disable ();

  if (thread_mlfqs)
  {
    // 自增 recent_cpu。
    thread_mlfqs_increase_recent_cpu ();

    if (ticks % TIMER_FREQ == 0)
      // 更新 load_avg 和 recent_cpu。
      thread_mlfqs_update_load ();

    else if (ticks % 4 == 0)
      // 更新当前线程的优先级。
      thread_mlfqs_update_priority (thread_current ());
  }

  thread_foreach (check_blocked, NULL);
  intr_set_level (old_level);
  thread_tick ();
}
```

直接调用上述工具方法，通过公式计算相关数据，
最终完成对线程的优先级调度。

---

最后，实现接口。代码如下：

```c
void
thread_set_nice (int nice UNUSED)
{
  thread_current ()->nice = nice;
  thread_mlfqs_update_priority (thread_current ());
  thread_yield ();
}

int
thread_get_nice (void)
{
  return thread_current ()->nice;
}

int
thread_get_load_avg (void)
{
  return fix_round (fix_mul (load_avg, fix_int (100)));
}

int
thread_get_recent_cpu (void)
{
  return fix_round (fix_mul (thread_current ()->recent_cpu, fix_int (100)));
}
```

代码足够简洁，此处不再赘述。

### 线程同步

同上，线程的同步已经得到保证。

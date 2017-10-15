# Chapter 4

Name: Dennis (沈德祐)

## 前提

Xv6 是運行在多個 processors 的 OS，也就是運行在多個獨立的 CPUs 的 OS。這些 CPUs 會共享實體的 RAM，這些 CPUs 會對共享的 RAM 做 `讀取` 及 `寫入` 的動作。由於同一時間有可能有多個 CPU 同時對 Memory 中相同的資料做 `讀` `寫`，導致資料問題。

因此有了 `Lock` 的解決方式。這一解決方式就是使得某一時間只能有一個 CPU 對某個在 Memory 中的某個資料做 `讀` `寫` 的動作。

## Race Conditions:

以上說明了為何需要 `Locks`。接下來說明何謂 `Race Condition`:

簡單的說，假設有 CPU1 及 CPU2。這兩個 CPU 都會先寫入資料到 a，接著再讀取 a 的資料。這時如果 CPU1 寫入資料到 a 後尚未讀取 a 資料前 CPU2 寫入資料到 a。接著 CPU1 讀取 a 的資料，就會讀取到 CPU2 寫入的資料。這個情況我們就稱作 `Race Condition`。

![image](https://raw.githubusercontent.com/teyushen/106-OS-homework1/gh-pages/%E9%9A%A8%E7%8F%AD%E9%99%84%E8%AE%80a128513/Race%20Condition.png)

#### Solution:

以上的解決方式就是將 a 保護起來。參考課本範例如下：

```c
struct list {
int data;
struct list *next;
};

struct list *list = 0; 
struct lock listlock;

void
insert(int data)
{
	struct list *l;
	
	acquire(&listlock);
	l = malloc(sizeof *l);
	l->data = data;
	l->next = list;
	list = l;
	release(&listlock);
}
```

我們可以看到此段程式碼利用 `acquire` 和 `release` 這兩個 function 來 lock 住想要保護的資料。其中在 `acquire` 和 `release` 中間的程式區間我們稱做 `critical section`。

> 在同一時間，只有一個 CPU 能操作在 critical section 裡面的資料

## Locks

Xv6 有兩種 lock 的類型：

1. spin-locks
2. sleep-locks

### spinlock 資料結構

spinlock 中的 `locked` 是非常重要的 field。用來判斷此 lock 目前是否可以取得。

```c
struct spinlock {
  uint locked;       // 若為 0：此 lock 目前是可取得的；若非 0：此 lock 目前無法取得

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
  uint pcs[10];      // The call stack (an array of program counters)
                     // that locked the lock.
};
```

## Using locks

在我們需要保護 data 時，我們常會遇到何時開始用 lock 及該使用多少 lock 來保護我們的 data。有兩個基本的原則：

1. 任何一個時間點只能有一個 CPU 對同一 variable 做寫入，此時其他 CPU 只能做讀取的的動作
2. 若需保護的 data 在記憶體中存在不同的位置，為了保護資料不變，此時我們應該用同一個 lock 來將所有變數鎖定。

## Deadlock and lock ordering

若有程式碼需要同時取得多個 locks，此時這些程式碼 lock 的順需應該要相同。

**Deadlock**
> 假設有兩段程式碼皆需要 lock 住 `A` 和 `B`。且有兩個執行緒 `Thread1` 和 `Thread2`。
> 
> `Thread1` 執行第一段程式碼 lock 順序是先 `A` 再 `B`；`Thread2` 執行第二段程式碼 lock 順序是先 `B` 再 `A`。
> 若 `Thread1` lock `A` 後，輪到 `Thread2` lock `B`。此時輪到 `Thread1` 需要 lock
> `B` 時，由於 `B` 已經被 lock 住。
> 
> 導致 `Thread1` 因為沒拿到 `B` 將 `A` 抓住不放；同理 `Thread2` 抓著 `B` 不放，此時就會造成 Deadlock

## Interrupt handlers

當一個 process 中斷了另一個 process，且此 process 執行需要某個 lock 時，如果此 lock 已經被被中斷的 process 取得，此時就會造成 processor 甚至整個系統 deadlock。

#### Solution:

在一個 process 要被另一個 process 中斷時，要被中斷的那個 process 不能握有任何 lock，否則不能被中斷。

## 程式碼解釋

### function acquire 解釋

`acquire` 是用來取得 lock 的 function

- 1576 行的 `pushcli` 是為了防止上述 interrupt 導致的 Deadlock。（需搭配 `release` 的 `pushcli` 一起看）
- 1581 行呼叫 `xchg` 判斷目前是否可以取得 lock。直到 locked 回傳 0 代表可取得此 lock。


```c
1574 acquire(struct spinlock *lk)
1575 {
1576 pushcli(); // disable interrupts to avoid deadlock.
1577 if(holding(lk))
1578 panic("acquire");
1579
1580 // The xchg is atomic.
1581 while(xchg(&lk−>locked, 1) != 0)
1582 ;
1583
1584 // Tell the C compiler and the processor to not move loads or stores 1585 // past this point, to ensure that the critical section’s memory 1586 // references happen after the lock is acquired.
1587 __sync_synchronize();
1588
1589 // Record info about lock acquisition for debugging.
1590 lk−>cpu = mycpu();
1591 getcallerpcs(&lk, lk−>pcs);
1592 }
```

### function release 解釋

`release ` 是用來釋放 lock 的 function

- 1622 行的 `popcli ` 在 `acquire` 呼叫 `pushcli` 時就會開始監控 processor 取得多少 lock，當 lock 數量尚未為 0 前，不會被其他 process interrupt。當 lock 數量達到 0 時，其他 process 的 critical section 才可被執行

```c
1600 // Release the lock.
1601 void
1602 release(struct spinlock *lk)
1603 {
1604 if(!holding(lk))
1605 panic("release");
1606
1607 lk−>pcs[0] = 0;
1608 lk−>cpu = 0;
1609
1610 // Tell the C compiler and the processor to not move loads or stores
1611 // past this point, to ensure that all the stores in the critical
1612 // section are visible to other cores before the lock is released.
1613 // Both the C compiler and the hardware may re−order loads and
1614 // stores; __sync_synchronize() tells them both not to.
1615 __sync_synchronize();
1616
1617 // Release the lock, equivalent to lk−>locked = 0.
1618 // This code can’t use a C assignment, since it might
1619 // not be atomic. A real OS would use C atomics here.
1620 asm volatile("movl $0, %0" : "+m" (lk−>locked) : );
1621
1622 popcli();
1623 }
```

## spinlock 流程圖

![spinlock](https://raw.githubusercontent.com/teyushen/106-OS-homework1/gh-pages/隨班附讀a128513/spinlock.png)







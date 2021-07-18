# CH2：I/O運作方式

Interrupt介紹、Hardware Resouce Protection

- I/O運作方式
   - polling I/O
   - Interrupted I/O
   - DMA
- Interrupt介紹
   - Interrupt之處理
   - 種類
- Hardware Resource Protection
   - 基礎建設
      - Dual modes運作
      - privileged instructions
      
   - I/O、Memory、CPU。protection
## I/O運作方式
### polling（詢問式） I/O
1. Def：又叫做Busy-waiting I/O或Programmed I/O
   - Step如下
      1. 執行中的process，發出I/O reguest給OS，希望I/O提供某種I/O服務。**e.g. Disk read a File**
      2. OS收到請求後，（可能）會先暫停（block）該process，即此process會放掉CPU，並置於waiting Queue，等待I/O-compeled。
      3. OS（or kernel）中的I/O-subsystem會處理此請求。
         e.g. 它會check **Disk cache**是否有命中，若有則從Disk cache取出File資料，不用Read I/O，反之則執行Read I/O
      4. I/O-subsystem會Pass此請求給device driver（驅動程式）。
      5. device drive會依此請求，設定相關I/O-commands到device controller（硬體）。
      6. device controller會指揮I/O-Device執行I/O運作。
      7. 此時，CPU可能idle，OS可能會將CPU分派給其它process使用。
      8. CPU會不斷的去Polling I/O-Device controller上之相關registers值，確定I/O運作完成與否或有無error。
      > 缺點：CPU並未將全部的Time用於Process exec.上，而是耗費大量時間去Polling I/O-Device controller，所以CPU utilization不高，且Process Throughput偏低。

![image-20210711115838361](./Imgaes/image-20210711115838361.png)

### Interrupt（中斷式） I/O
1. Def：Step如下
   1. ~   7. 如前述Polling I/O
   8. 當I/O運作完成，I/O-Device controller會發出一個"I/O-Completed" interrupt通知CPU（OS）。
   9. OS收到中斷通知後，（可能）會先暫停目前執行中的Process。
      **e.g. PB exection -> Read status**
   10. OS會依據Interrupt ID（No）查詢Interrupt vector（表），找出中斷對應的**服務處理程式**（ISR；Interrupt Service Routine）之位址。
   11. Jump to ISR位置，ISR執行。
         **e.g. 將File Data從Controller之Buffer registers搬到memory中**
   12. ISR完成，控制權交回kernel I/O subsystem，通知Process其I/O-Completed及告知結果。
   13. OS恢復中斷之前Process的執行（e.g. PB exec.）或交由CPU schedulen決定下一個執行之Process

![image-20210711120841626](./Imgaes/image-20210711120841626.png)

> 優點：
> 
> CPU無須花費時間用於Polling I/O-Device controller，而是可全心用於Process之execution上，所以CPU utilization較高，Throughput相對也較高。故improve the system performance。
> 
> 缺點：
> 1. Interrupt之處理仍須耗費CPU time（e.g. 查表、執行ISR、保存中斷前Process之Status、etc.），此段時間CPU time無法用於user process execution。
>    **Note：若I/O運作時間很快速（大於Interrupt處理Time），此時Polling反而比較好**
> 2. 若Interrupt發生頻率較高，則CPU utilization會很差。因為CPU time幾乎都花在中斷之處理上。
> 3. CPU仍須耗費CPU time用於I/O-Device controller與Memory間的Data transfer上。

   P.40 - 41 ［恐］Lifecycle of An I/O-reguest

### DMA（Direct Memory Access）
1. Def：DMA controller負責I/O-Device與Memory之間的Data transfer工作，此transfer過程不須要CPU之參予。
   因為CPU有更多時間用於Process exec.上。（優點 1）
   - 另外DMA適合用在Block-Transfer oriented I/O-Device，例：Disk
   > Note：可以降低I/O-Completed中斷頻率
   > **Note：考點
   >  Byte-transfer oriented (X)
   >  Character (X)**

   - 缺點：引入DMA controller，會**增加硬體系統設計之複雜度**（Complicates Hardware Design）
      理由：DMA controller必須與CPU競爭Memory與Bus之使用權。
      **故必須有一個硬體協調設計機制，此技術叫做"interleaving"或cycle stealing。**
   - 有時CPU會被迫等待DMA when it make use of memory bus。當與CPU conflict時，通常給DMA高優先權。
      理由：DMA對Memory、Bus之使用頻率低於CPU很多，優先配給DMA會有比較小的平均Waiting Time及較高之產出（Throughput）。
## Interrupt介紹

### Interrupt之處理

1. kernel所在的memory area中，會存有一個"Interrupt vector"（表），內放各式interrupt ID及各式ISRs之位址，此外也會存放這一些ISRs之Binary Code。

![image-20210711125733163](./Imgaes/image-20210711125733163.png)

2. Interrupt發生後，OS之處理Steps：
   1. OS收到中斷後，若要立即處理，則會先暫停目前Process執行，且會保存其Status。（e.g. PB被暫停（running -> ready Queue）PB之Status會保存（ch4））
   2. OS會查詢Interrupt Vector based on "Interrupt ID"，確定何種中斷發生，且找出它的ISR位址。
   3. Jump to ISR位址，執行ISR。
   4. 待ISR完成後，return Contro to kernel。
   5. kernel resumes（恢復）原先中斷前之Process執行。
      **e.g. PB恢復執行**
   
3. Interrupt種類
   ［分類一］分為三種：

   1. External interrupt
      -> CPU以外的週邊設備或元件所發出的。
      > 例："I/O-Completed"中斷，由Controller發出
      >    "I/O-error"中斷，由Controller發出
      >    "machine-check"，開機時各設備回報其是否正常
      
   2. Internal interrupt
      -> CPU執行Process時，遭遇重大error而引起。
      
      > 例：divide-by-zero、illegel特權指令執行
      
   3. Software interrupt
      Def：Process在執行時，若需要OS提供某種服務時，它會發出此類型**中斷**通知OS，OS收到通知後，才會執行相關的**服務項目**（ch3 system calls）。
   
   
   
   ［分類二］分為兩種:star::star::star:：
   
   | Interrupt                                                   | Trap                                                         |
   | ----------------------------------------------------------- | ------------------------------------------------------------ |
   | Def：Hardware-generated change of control flow.             | Software-generated interrupt                                 |
   | 例："I/O-Completed"、"I/O-error"、"machine-check"via device | 用途有二：<br />1. Catch the arithernatic error（即Process執行遭遇重大error）<br />e.g. divide-by-zero、illegal memory access、etc.<br />2. Software interrupt用途Process執行，需要OS提供服務時，會先發trap通知OS。 |
   
   ［分類三］：中斷發生後，是否需要立刻處理？或上個中斷還在處理，又有其他中斷發生，是否要立刻處理？
   > Interrupt之間應有優先權

   分為：
   1. Non-maskable（不可遮罩） interrupt：此中斷須立刻處理。
      **e.g. 重大error引起之中斷（internal中斷）**
   2. Maskable interrupt：此類中斷發生，可以ignore it或delay processing。
      **e.g. Software interrupt**

## Blocking-I/O、Non-Blocking I/O、Asychronous-I/O

1. Blocking-I/O：Process suspended until I/O-completed.

   - Easy to understand and use.
   - Insufficient（不足） for some needs**（e.g. Play video程式）**.

2. Non-Blocking I/O：Process still runs when I/O operation.

   - I/O calls returns **as much as possible（有多少給多少）**.
   - e.g. **user-Interface**、**data copy（buffer I/O）**.
   - Implemented via multi-threading.

3. Asynchronous I/O：Process runs while I/O executes difficult to use.

   - 與Non-Blocking I/O相似

     差別：I/O subsystem singels process when **I/O-completed（整個I/O完成才通知Process）**

圖示：

![image-20210714191051586](./Imgaes/image-20210714191051586-6261054-6261065.png)



## Hardware Resource Protection

### 基礎建設

#### Dual Modes operation

1. **Def**：系統的運作模式至少要可以被區分出兩種Modes

   1. **kernel mode**：又叫**System** or **Privileged** or **Supervisor mode**（更早期叫做monitor mode［恐］）。代表kernel取得對系統的控制權（即kernel取得CPU，在執行一些Systemprocess e.g. ISR、System call、etc.）

      > 在此mode下可以執行特權指令

   2. **user mode**：代表user process取得CPU在執行之mode，在此mode下，**不允許**執行特權指令。此外，Dual mode之實現需要Hardware額外支援。（e.g. CPU提供一個**Mode Bit**，用以區分目前是何種mode。）

   > Note：可以比兩種更多
   >
   > e.g. 
   >
   > - user mode
   >   - virtual user mode
   >   - virtual kernel mode
   > - kernel mode

#### Privileged Instruction（特權指令）

1. **Def**：任何可能造成系統重大危害之指令，均可定義為特權指令。

   > 只允許在kernel mode下執行，不允許在user mode下執行。

2. **一般而言**，下列指令為特權指令：

   - I/O instruction（for I/O protection）
   - Base/Limit register值修改（for Memory protection）
   - Timer值修改指令（for CPU protection）
   - Turn off（or Disable） interrupt
   - Switch（change） mode to kernel mode指令
   - Clear memory

   > 例［恐］［EX］ 要設置特權的指令？
   >
   > 1. change to user mode
   > 2. change to monitor mode
   > 3. read from monitor memory
   > 4. write into monitor memory
   > 5. fetch on instruction from monitor memory
   > 6. turn on timer interrupt
   > 7. turn off timer interrupt
   >
   > Ans：有爭議性（作者主觀）
   >
   > 1. 恐 2、4、7、**3** （政大、中央）
   > 2. 交大 2、4、7、**5**

### I/O-Protection

1. **目的**：防止user process直接執行I/O指令操作I/O Device，降低出錯機率及使用複雜度。

2. **作法**：I/O指令設為特權指令。

   將來使用情境：

   ![image-20210714193859548](./Imgaes/image-20210714193859548.png)

### Memory Protection

1. **目的**：防止user process任意存取其他processes及kernel所在的memory area。

2. ［假設memory管理採用Contiguous Allocation方法（ch 7）］

   **作法**：OS利用一套register：

   - Base register：記錄Process之起始位置
   - Limit register：記錄Process之大小

   Process執行時，搭配下列的Checking flow：

   ![image-20210714194325413](./Imgaes/image-20210714194325413-6263010.png)

   此外，Base及Limit register值之set/change需設為特權指令。

### CPU protection

1. **目的**：防止user process長期/無限期佔用CPU而不釋放

2. **作法**：利用Timer（計時器；硬體），OS會規定一個Process使用CPU之Max-Time-Quantum（最大時間配額），當Process取得CPU後，Timer初值即設為Max-Time-Quantum，隨著Process使用CPU之Time增加，Timer值**會逐步遞減**。當Time值=0時，Timer會發出"Time-Out"（expires） interrupt通知OS，OS會強迫此Process放掉CPU。

   > Note：此機制也適用於RR排班之實施（ch4）

   此外，Timer值之set/change需設為特權指令。

   EX：
   Read the system clock
   Set the system clock 皆不需特權


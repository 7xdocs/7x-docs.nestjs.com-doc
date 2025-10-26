### 任务调度

任务调度允许你安排任意代码（方法/函数）在固定的日期/时间、按重复间隔或指定间隔后执行一次。在 Linux 世界中，这通常由操作系统级别的 [cron](https://en.wikipedia.org/wiki/Cron) 等包处理。对于 Node.js 应用，有几个包可以模拟类似 cron 的功能。Nest 提供了 `@nestjs/schedule` 包，它与流行的 Node.js [cron](https://github.com/kelektiv/node-cron) 包集成。我们将在本章中介绍这个包。

#### 安装

要开始使用它，我们首先安装所需的依赖。

```bash
$ npm install --save @nestjs/schedule
```

要激活任务调度，将 `ScheduleModule` 导入到根 `AppModule` 中，并运行 `forRoot()` 静态方法，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` 调用会初始化调度器，并注册你应用中存在的任何声明式 <a href="techniques/task-scheduling#declarative-cron-jobs">cron 任务</a>、<a href="techniques/task-scheduling#declarative-timeouts">超时任务</a> 和 <a href="techniques/task-scheduling#declarative-intervals">间隔任务</a>。注册发生在 `onApplicationBootstrap` 生命周期钩子时，确保所有模块都已加载并声明了任何调度任务。

#### 声明式 cron 任务

Cron 任务安排一个任意函数（方法调用）自动运行。Cron 任务可以：

- 在指定的日期/时间运行一次。
- 定期运行；循环任务可以在指定间隔内的特定时刻运行（例如，每小时一次、每周一次、每 5 分钟一次）

在包含要执行代码的方法定义前使用 `@Cron()` 装饰器来声明一个 cron 任务，如下所示：

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('Called when the current second is 45');
  }
}
```

在这个例子中，每当当前秒数为 `45` 时，`handleCron()` 方法就会被调用。换句话说，该方法将在每分钟的第 45 秒运行一次。

`@Cron()` 装饰器支持以下标准 [cron 模式](http://crontab.org/)：

- 星号 (例如 `*`)
- 范围 (例如 `1-3,5`)
- 步长 (例如 `*/2`)

在上面的例子中，我们向装饰器传递了 `45 * * * * *`。下面的键说明了 cron 模式字符串中每个位置的解释：

<pre class="language-javascript"><code class="language-javascript">
* * * * * *
| | | | | |
| | | | | 星期几
| | | | 月份
| | | 日期
| | 小时
| 分钟
秒 (可选)
</code></pre>

一些 cron 模式示例：

<table>
  <tbody>
    <tr>
      <td><code>* * * * * *</code></td>
      <td>每秒</td>
    </tr>
    <tr>
      <td><code>45 * * * * *</code></td>
      <td>每分钟，在第 45 秒</td>
    </tr>
    <tr>
      <td><code>0 10 * * * *</code></td>
      <td>每小时，在第 10 分钟开始时</td>
    </tr>
    <tr>
      <td><code>0 */30 9-17 * * *</code></td>
      <td>上午 9 点到下午 5 点之间每 30 分钟</td>
    </tr>
   <tr>
      <td><code>0 30 11 * * 1-5</code></td>
      <td>周一至周五上午 11:30</td>
    </tr>
  </tbody>
</table>

`@nestjs/schedule` 包提供了一个包含常用 cron 模式的便捷枚举。你可以按如下方式使用此枚举：

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('Called every 30 seconds');
  }
}
```

在这个例子中，`handleCron()` 方法将每 `30` 秒被调用一次。如果发生异常，它将被记录到控制台，因为每个用 `@Cron()` 注解的方法都会自动包装在 `try-catch` 块中。

或者，你可以向 `@Cron()` 装饰器提供一个 JavaScript `Date` 对象。这样做会导致任务在指定日期精确执行一次。

> info **提示** 使用 JavaScript 日期算法来安排相对于当前日期的任务。例如，使用 `@Cron(new Date(Date.now() + 10 * 1000))` 来安排一个在应用启动 10 秒后运行的任务。

此外，你可以将附加选项作为第二个参数提供给 `@Cron()` 装饰器。

<table>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        用于在声明后访问和控制 cron 任务。
      </td>
    </tr>
    <tr>
      <td><code>timeZone</code></td>
      <td>
        指定执行任务的时区。这将根据你的时区修改实际时间。如果时区无效，将抛出错误。你可以在 <a href="http://momentjs.com/timezone/">Moment Timezone</a> 网站上查看所有可用的时区。
      </td>
    </tr>
    <tr>
      <td><code>utcOffset</code></td>
      <td>
        这允许你指定时区的偏移量，而不是使用 <code>timeZone</code> 参数。
      </td>
    </tr>
    <tr>
      <td><code>disabled</code></td>
      <td>
        这指示任务是否会被执行。
      </td>
    </tr>
  </tbody>
</table>

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

你可以在声明 cron 任务后通过 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">动态 API</a> 访问和控制它，或者动态创建一个 cron 任务（其 cron 模式在运行时定义）。要通过 API 访问声明式 cron 任务，你必须通过将 `name` 属性作为装饰器第二个参数的可选选项对象传递，将任务与名称关联起来。

#### 声明式间隔任务

要声明一个方法应以（重复的）指定间隔运行，请在方法定义前加上 `@Interval()` 装饰器。将间隔值（以毫秒为单位的数字）传递给装饰器，如下所示：

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('Called every 10 seconds');
}
```

> info **提示** 此机制在底层使用 JavaScript 的 `setInterval()` 函数。你也可以利用 cron 任务来安排循环任务。

如果你想通过 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">动态 API</a> 从声明类的外部控制你的声明式间隔，请使用以下结构将间隔与名称关联：

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

如果发生异常，它将被记录到控制台，因为每个用 `@Interval()` 注解的方法都会自动包装在 `try-catch` 块中。

<a href="techniques/task-scheduling#dynamic-intervals">动态 API</a> 还支持**创建**动态间隔（其属性在运行时定义），以及**列出和删除**它们。

<app-banner-enterprise></app-banner-enterprise>

#### 声明式超时任务

要声明一个方法应在指定的超时时间（一次）运行，请在方法定义前加上 `@Timeout()` 装饰器。将相对于应用启动的相对时间偏移（以毫秒为单位）传递给装饰器，如下所示：

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('Called once after 5 seconds');
}
```

> info **提示** 此机制在底层使用 JavaScript 的 `setTimeout()` 函数。

如果发生异常，它将被记录到控制台，因为每个用 `@Timeout()` 注解的方法都会自动包装在 `try-catch` 块中。

如果你想通过 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">动态 API</a> 从声明类的外部控制你的声明式超时，请使用以下结构将超时与名称关联：

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

<a href="techniques/task-scheduling#dynamic-timeouts">动态 API</a> 还支持**创建**动态超时（其属性在运行时定义），以及**列出和删除**它们。

#### 动态调度模块 API

`@nestjs/schedule` 模块提供了一个动态 API，可以管理声明式 <a href="techniques/task-scheduling#declarative-cron-jobs">cron 任务</a>、<a href="techniques/task-scheduling#declarative-timeouts">超时任务</a> 和 <a href="techniques/task-scheduling#declarative-intervals">间隔任务</a>。该 API 还支持创建和管理**动态** cron 任务、超时任务和间隔任务，这些任务的属性在运行时定义。

#### 动态 cron 任务

使用 `SchedulerRegistry` API 从代码中的任何地方按名称获取 `CronJob` 实例的引用。首先，使用标准的构造函数注入方式注入 `SchedulerRegistry`：

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

> info **提示** 从 `@nestjs/schedule` 包中导入 `SchedulerRegistry`。

然后在类中如下使用它。假设一个 cron 任务是通过以下声明创建的：

```typescript
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

使用以下方式访问此任务：

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

`getCronJob()` 方法返回指定名称的 cron 任务。返回的 `CronJob` 对象具有以下方法：

- `stop()` - 停止计划运行的任务。
- `start()` - 重新启动已停止的任务。
- `setTime(time: CronTime)` - 停止任务，为其设置新时间，然后启动它。
- `lastDate()` - 返回一个 `DateTime` 对象，表示任务最后一次执行的日期。
- `nextDate()` - 返回一个 `DateTime` 对象，表示任务下一次计划执行的日期。
- `nextDates(count: number)` - 提供一个数组（大小为 `count`），包含将触发任务执行的下一个日期集的 `DateTime` 表示。`count` 默认为 0，返回一个空数组。

> info **提示** 在 `DateTime` 对象上使用 `toJSDate()` 方法，将其渲染为与此 DateTime 等效的 JavaScript Date。

使用 `SchedulerRegistry#addCronJob` 方法**动态创建**一个新的 cron 任务，如下所示：

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`time (${seconds}) for job ${name} to run!`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `job ${name} added for each minute at ${seconds} seconds!`,
  );
}
```

在这段代码中，我们使用 `cron` 包中的 `CronJob` 对象来创建 cron 任务。`CronJob` 构造函数将 cron 模式（就像 `@Cron()` <a href="techniques/task-scheduling#declarative-cron-jobs">装饰器</a>一样）作为第一个参数，并将 cron 计时器触发时要执行的回调作为第二个参数。`SchedulerRegistry#addCronJob` 方法接受两个参数：`CronJob` 的名称和 `CronJob` 对象本身。

> warning **警告** 在访问 `SchedulerRegistry` 之前记得先注入它。从 `cron` 包导入 `CronJob`。

使用 `SchedulerRegistry#deleteCronJob` 方法**删除**指定名称的 cron 任务，如下所示：

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`job ${name} deleted!`);
}
```

使用 `SchedulerRegistry#getCronJobs` 方法**列出**所有 cron 任务，如下所示：

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDate().toJSDate();
    } catch (e) {
      next = 'error: next fire date is in the past!';
    }
    this.logger.log(`job: ${key} -> next: ${next}`);
  });
}
```

`getCronJobs()` 方法返回一个 `map`。在这段代码中，我们遍历该 map 并尝试访问每个 `CronJob` 的 `nextDate()` 方法。在 `CronJob` API 中，如果一个任务已经触发并且没有未来的触发日期，它会抛出一个异常。

#### 动态间隔任务

使用 `SchedulerRegistry#getInterval` 方法获取间隔任务的引用。如上所述，使用标准的构造函数注入方式注入 `SchedulerRegistry`：

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

并按如下方式使用：

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

使用 `SchedulerRegistry#addInterval` 方法**动态创建**一个新的间隔任务，如下所示：

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Interval ${name} executing at time (${milliseconds})!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

在这段代码中，我们创建了一个标准的 JavaScript 间隔，然后将其传递给 `SchedulerRegistry#addInterval` 方法。
该方法接受两个参数：间隔的名称和间隔本身。

使用 `SchedulerRegistry#deleteInterval` 方法**删除**指定名称的间隔任务，如下所示：

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`Interval ${name} deleted!`);
}
```

使用 `SchedulerRegistry#getIntervals` 方法**列出**所有间隔任务，如下所示：

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`Interval: ${key}`));
}
```

#### 动态超时任务

使用 `SchedulerRegistry#getTimeout` 方法获取超时任务的引用。如上所述，使用标准的构造函数注入方式注入 `SchedulerRegistry`：

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

并按如下方式使用：

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

使用 `SchedulerRegistry#addTimeout` 方法**动态创建**一个新的超时任务，如下所示：

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Timeout ${name} executing after (${milliseconds})!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

在这段代码中，我们创建了一个标准的 JavaScript 超时，然后将其传递给 `SchedulerRegistry#addTimeout` 方法。
该方法接受两个参数：超时的名称和超时本身。

使用 `SchedulerRegistry#deleteTimeout` 方法**删除**指定名称的超时任务，如下所示：

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`Timeout ${name} deleted!`);
}
```

使用 `SchedulerRegistry#getTimeouts` 方法**列出**所有超时任务，如下所示：

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`Timeout: ${key}`));
}
```

#### 示例

一个可工作的示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/27-scheduling)找到。
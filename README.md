# @requence/producer

This package creates a task that can be processed.

## Basic Usage

```typescript
import createProducer from '@requence/producer'

const producer = await createProducer()
```

Every producer instance needs a `url` parameter to connect to the operator. In the basic example, this parameter gets automatically retrieved from environment variables `REQUENCE_URL`.
To be more explicit about this configuration, you can pass the connection as the first argument:

```typescript
const producer = await createProducer('your operator connection url string')
```

### Create a task template

```typescript
import createProducer, { createTemplate } from '@requence/producer'

// create a template
const template = createTemplate().addService('dummy', '1')

storeTemplateSomehow(template.toJSON())
```

This will create a new template that only consists of one service.

### Create a task

```typescript
const templateJson = loadTemplateSomehow()
const task = producer.createTask(templateJson)
const result = await task.run()
```

### Create and execute a one time task

The `createTask` method also accepts a builder function to streamline
the process for quick one time tasks

```typescript
const task = producer.createTask((builder) => builder.addService('dummy', '1'))
const result = await task.run()
```

### Provide task input and metadata

```typescript
const task = producer
  .createTask(template)
  .withMeta({ traceId: 123 })
  .withInput('some-input-value')

const result = await task.run()
```

## Building complex task logic

In the previous example, only one service got executed. The template builder provides method to make the task infinitely complex.

### Executing a service

```typescript
createTemplate().addService('service-1', '1.0.0')
```

this will execute the service named `service-1` in version `1.0.0`. The version can also be specified as a version range or a version wildcard like `1.0.0`, `^1.0.0`, `1.0.0 - 1.5.0`, `1.0.x` etc. If the version is omitted, `*` is implied.

### sequential services

```typescript
createTemplate().addService('service-1', '1').addService('service-2', '1')
```

this will call `service-1` and `service-2` in sequence, so that `service-2` can access the result of `service-1`

### parallel services

```typescript
createTemplate().addParallel((parallel) =>
  parallel.addService('service-1').addService('service-2'),
)
```

this will call `service-1` and `service-2` in parallel

### parallel execution of sequential services

```typescript
createTemplate()
  .addParallel((parallel) =>
    parallel
      .addSequence((sequence) =>
        sequence.addService('service-1').addService('service-2'),
      )
      .addService('service-3'),
  )
  .addService('service-4')
```

this will execute service 1 to 4 in the following order:

![Parallel Sequence Order](./docs/parallel-sequence-order.svg)

### conditional execution

```typescript
createTemplate()
  .addService('service-1') // assume the response is { "done": boolean }
  .addCondition('service{service-1}.done', '===', true)
  .then((t) => t.addService('service-2'))
  .else((e) => e.addService('service-3'))
```

When `service-1` resolves with `{"done": true}`, `service-2` will get executed, otherwise `service-3`. When no `else` case is defined, the task would stop.

### error handling

There are two ways to deal with errors in tasks:

```typescript
createTemplate().addService('service-1').onFailSkip().addService('service-2')
```

This will allow `service-1` to fail. The task will continue without the result of `service-1` and moves to `service-2`

```typescript
createTemplate()
  .addService('service-1')
  .onFail((f) => f.addService('service-2'))
  .addService('service-3')
```

This will execute `service-2` only when `service-1` fails. Then the task will continue to `service-3`

### adding retries

In some cases a service could fail but recover automatically after a grace period.

```typescript
createTemplate().addService('service-1').withRetry(3, 5_000)
```

This will retry to execute `service-1` three times with a delay of 5 seconds inbetween.

### adding service configuration

```typescript
createTemplate().addService('service-1').withConfiguration('some-config')
```

### adding aliases

When a service is used multiple times in a task, it is hard for other services to retrieve the correct result. For this case, an alias can be defined per service.

```typescript
createTemplate()
  .addService('search')
  .withConfiguration({ searchFor: 'name' })
  .withAlias('name-result')
  .addService('search')
  .withConfiguration({ searchFor: 'job' })
  .withAlias('job-result')
```

## Receiving updates

there are 2 ways to receive task specific updates.

Via callback:

```typescript
producer.createTask(template).run((update) => {
  console.log('received update', update)
})
```

Via Async Iterator:

```typescript
const resultPromise = producer.createTask(template).run()

for await (const update of resultPromise) {
  console.log('received update', update)
}
```

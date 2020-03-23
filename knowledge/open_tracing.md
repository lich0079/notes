# OpenTracing

## Model
  * Trace
    * The description of a transaction as it moves through a distributed system.
  * Span
    * An operation name
    * A start timestamp
    * A finish timestamp
    * A set of zero or more key:value Span Tags. The keys must be strings. The values may be strings, bools, or numeric types.
    * A set of zero or more Span Logs, each of which is itself a key:value map paired with a timestamp.
    * A SpanContext
    * References to zero or more causally-related Spans
  * Span context
    * Trace identifier, span identifier, and any other data that the tracing system needs to propagate to the downstream service
  * References between Spans
    * ChildOf references
      * A Span may be the ChildOf a parent Span
      ```
            [-Parent Span---------]
                [-Child Span----]

            [-Parent Span--------------]
                [-Child Span A----]
                [-Child Span B----]
                [-Child Span C----]
                [-Child Span D---------------]
                [-Child Span E----]
      ```
    * FollowsFrom references
      *  Some parent Spans do not depend in any way on the result of their child Spans. 
        ```
            [-Parent Span-]  [-Child Span-]


            [-Parent Span--]
            [-Child Span-]


            [-Parent Span-]
                        [-Child Span-]
        ```



## Design of the Distributed Tracing systems

 * A tracing instrumentation API: What decorates application code.
   * 对业务代码的包装器API
 * Wire protocol: What gets sent alongside application data in RPC requests.
   * RPC 调用是传了传递业务数据，同时也需要传递 Span context
 * Data protocol: What gets sent asynchronously (out-of-band) to your analysis system.
   * trace data 需要异步的传递给分析系统
 * Analysis system: A database and interactive UI for working with the trace data.
   * 存储数据的数据库， 用来查询数据的交互UI
# how to view EventSource

## 简单流程

````c#
public class CustomEventListener : EventListener
{
        protected override void OnEventSourceCreated(EventSource eventSource)
        {
            if (eventSource.Name == "what_event_i_want")
            {
                EnableEvents(eventSource, which_level_i_want);
            }
        }

        protected override void OnEventWritten(EventWrittenEventArgs eventData)
        {
             var eventId = eventData.EventId;
             // print base information

             if (eventData.PayLoad == null) return;

             foreach (var payload in eventData.PayLoad)
             {
                 // print payload
             }
        }
}
````

在启动程序时加入

````c#
    static async Task Main(string[] args)
    {
        var listener = new CustomEventListener();

        // some some some
    }
````

## 定制化

### 可配置化

进一步的将 `OnEventSourceCreated` 改造成可以代码配置化。

````c#
    public class EventMatchList : List<EventMatch>
    {
        public static readonly EventMatchList Instance = new EventMatchList();
    }

    public class EventMatch
    {
        public String SourceName { get; set; }

        public EventLevel LogLevel { get; set; }

        public List<WriteEventMatch> ArgsMatches { get; } = new ();
    }

    public class class CustomEventListener
    {
        protected override void OnEventSourceCreated(EventSource eventSource)
        {
            var match = EventMatchList.Instance.FirstOrDefault(m => m.SourceName == eventSource.Name);

            if (match != null)
            {
                EnableEvents(eventSource, match.LogLevel);
            }
        }
    }
````

更进一步的将 `OnEventWritten` 改造成可以代码配置化。

````c#
    public class WriteEventMatch
    {
        public Func<EventWrittenEventArgs, bool> Match { get; set; }

        public Action<EventWrittenEventArgs> Apply { get; set; }
    }

    public class EventMatch
    {
        public String SourceName { get; set; }

        public EventLevel LogLevel { get; set; }

        public List<WriteEventMatch> ArgsMatches { get; } = new ();
    }

    public class class CustomEventListener
    {
        protected override void OnEventWritten(EventWrittenEventArgs eventData)
        {
             var match = EventMatchList.Instance.FirstOrDefault(m => m.SourceName == eventData.EventSource.Name);

             if (match == null) return;

             foreach (var writeEventMatch in match.ArgsMatches.Where(writeEventMatch => writeEventMatch.Match(eventData)))
             {
                 writeEventMatch.Apply(eventData);
             }
        }
    }
````

### 实际使用

对常见的 `http client request` 增加日志

首先查看系统源码

````c#
  [EventSource(Name = "Private.InternalDiagnostics.System.Net.Http", LocalizationResources = "FxResources.System.Net.Http.SR")]
  internal sealed class NetEventSource : EventSource
````

虽然是个内部类，但EventSourceName还是知道的，接下来根据名字注入行为即可。

````c#
private static void InitEventMatch()
    {
        var eventMatch = new EventMatch
        {
            SourceName = "Private.InternalDiagnostics.System.Net.Http",
            LogLevel = EventLevel.LogAlways,
        };

        var event8Match = new WriteEventMatch()
        {
            Match = e => e is { EventId: 8, Payload.Count: >= 5 }, 
            Apply = e =>
            {
                if (e.Payload![4] is string s)
                {
                    //Console.WriteLine($"#{e.EventId}# {e.EventName} Message: {s}");
                    Console.WriteLine($"#{Environment.CurrentManagedThreadId}# Message: {s}");

                }
            }
        };
        eventMatch.ArgsMatches.Add(event8Match);

        var event1Match = new WriteEventMatch()
        {
            Match = e => e is { EventId: 1, Payload.Count: >= 3 }, 
            Apply = e =>
            {
                if (e.Payload![2] is string s)
                {
                    Console.WriteLine($"{e.EventName} Message: {s}");
                }
            }
        };

        //eventMatch.ArgsMatches.Add(event1Match);

        var event3Match = new WriteEventMatch()
        {
            Match = e => e is { EventId: 3, Payload.Count: >= 4 }, 
            Apply = e =>
            {
                if (e.Payload![2] is string first && e.Payload![3] is string second)
                {
                    Console.WriteLine($"{e.EventName} Message: {first} <==> {second}");
                }
            }
        };
        //eventMatch.ArgsMatches.Add(event3Match);

        EventMatchList.Instance.Add(eventMatch);
    }
````

````c#
    static async Task Main(string[] args)
    {
        InitEventMatch();

        var listener = new CustomEventListener();

        HttpGetDemo();
    }
````


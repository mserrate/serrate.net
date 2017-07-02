---
title: 'NServiceBus: DRY with unobtrusive conventions'
date: 2012-10-16T10:20:11+00:00
categories:
  - Architecture
  - NServiceBus
tags:
  - NServiceBus
  - SOA
---
Many times when working with NServiceBus in <a href="http://nservicebus.com/docs/Samples/UnobtrusiveModeSample.aspx" target="_blank">unobtrusive mode</a> you may feel that you are repeating the same conventions over and over again on all the endpoints.

The **IWantToRunBeforeConfiguration** interface is a great help in order to embrace the DRY principle.

Just define your implementation in an assembly referenced by all the endpoints:

{% codeblock lang:csharp %}
public class UnobtrusiveConventions : IWantToRunBeforeConfiguration
{
    public void Init()
    {
        Configure.Instance
            .DefiningCommandsAs(t => t.Namespace != null
                && t.Namespace.EndsWith("Commands"))
            .DefiningEventsAs(t =>; t.Namespace != null
                && t.Namespace.EndsWith("Events"))
            .DefiningMessagesAs(t => t.Namespace != null
                && t.Namespace.EndsWith("Messages"));
    }
}
{% endcodeblock %}

and NServiceBus will pick this class automatically for each endpoint.
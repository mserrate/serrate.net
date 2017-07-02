---
title: UI Composition for Services
date: 2012-08-22T09:22:49+00:00
banner: /files/2012/08/uicomposition.jpg
categories:
  - Architecture
tags:
  - DDD
  - SOA
---
One of the most important concepts when applying either **SOA** or **DDD** is the definition of Services (or Bounded Contexts in the DDD lingo).

Each of these services will be responsible for its own **data and behavior** and could also own the UI components of that service.

Let’s see an example with the typical Ecommerce domain:

{% img border /files/2012/08/Services.png 420 258 Services %}

<!--more-->
  
In this case we have two services, Sales & Shipping, and each of them owns its UI components that will be rendered by the UI host. That&#8217;s the result:

{% img border /files/2012/08/uicomposition.png 500 468 Ecommerce composition %}

## Composition with ASP.NET MVC

In this example, I&#8217;m using <a href="http://razorgenerator.codeplex.com/" target="_blank">Razor Generator</a> in order to let services to have their own MVC components in class libraries, which will be referenced by the central UI host:

{% img border /files/2012/08/VSSolution.png 321 596 VS solution %}

The use of Razor Generator is very straightforward:

  1. Install RazorGenerator from VS gallery
  2. On each class library, install RazorGenerator.Mvc from Nuget
  3. Create the folder structure following MVC conventions (Controllers, Views, etc.)
  4. Add a <a href="https://github.com/mserrate/ui-composition/blob/master/UIComposition/Serrate.Sales.UI/Web.config" target="_blank">web.config</a> to get intellisense for your views
  5. Change &#8220;Custom Tool&#8221; to RazorGenerator on your views in order to precompile it

Then, decorate your actions as ChildActionOnly to behave like a widget:

{% codeblock lang:csharp %}
public class FinishOrderController : Controller
{
	private readonly IBus bus;

	public FinishOrderController(IBus bus)
	{
		this.bus = bus;
	}

	[ChildActionOnly]
	public ActionResult SubmitOrCancel()
	{
		ViewBag.OrderId = TheSession.OrderId;

		return PartialView();
	}

	[HttpPost]
	public ActionResult SubmitOrder()
	{
		var cmd = new SubmitOrder()
		{
			OrderId = TheSession.OrderId
		};

		this.bus.Send(cmd);

		return RedirectToAction("Processed", "Order");
	}

	[HttpPost]
	public ActionResult CancelOrder()
	{
		var cmd = new CancelOrder()
		{
			OrderId = TheSession.OrderId
		};

		this.bus.Send(cmd);

		return RedirectToAction("Cancelled", "Order");
	}
}
{% endcodeblock %}

Take a look at the sample on github and let me know!!

<https://github.com/mserrate/ui-composition/tree/master/UIComposition>
---
layout: post
title:  "Using reflection with Apex Plugins"
categories: sfdc apex
---

If you are the developer of a managed package and you want to make it extensible
with Apex Plugins, here is a [really good presentation](http://www.slideshare.net/stephenwillcock/apex-plugins)
by Stephen Willock.

The main point of developing Apex Plugins is to allow developers to introduce
some custom logic that will be executed from within your package. Ever wonder
how you could get the value of a property on a class, or even a specific class
without knowing the class name?

That's where you start using [reflection](http://en.wikipedia.org/wiki/Reflection_(computer_programming))
and make your code examine and modify its value and behavior based on a code (= plugin)
provided by your customers.

Reflection has firstbeen introduced in the
[Summer '12 release](http://blogs.developerforce.com/developer-relations/2012/05/dynamic-apex-class-instantiation-in-summer-12.html)
where the methods `getName` and `newInstance` have been added to the [`Type`](http://www.salesforce.com/us/developer/docs/apexcode/Content/apex_methods_system_type.htm)
system class. You can now run the following snippet:

{% highlight java %}
Type t = Type.forName("YourCustomClass");
YourCustomClass newObj = (YourCustomClass) t.newInstance();
{% endhighlight %}

But how is this being used with the Apex Plugins pattern?

Imagine that you have a class that, let's say, query for some accounts to do
some business logic with those accounts. The initial list of accounts is retrieved
through a [Dynamic SOQL](http://www.salesforce.com/us/developer/docs/apexcode/Content/apex_dynamic_soql.htm)
query.

{% highlight java %}
public class MyAccountController {
  public void businessMethod() {
    String queryString = "SELECT Id, Name FROM Account"
        + " WHERE MyCustomField__c = 'some value'";
    List<Account> accounts = Database.query(queryString);
    // Do something with the list of accounts...
  }
}
{% endhighlight %}

### Why using Apex Plugins?

This class is within your managed package, therefore your customers are not able
to override and change the query string (especially the `WHERE` clauses) to restrict
the number of accounts being used in your custom logic.

You want to add a `IQueryPlugin` interface in your class, allowing your users
to write their own plugins. Here is an example of implementation.

{% highlight java %}
// Note that the class is global since we want our customer to
// implement their own query plugin interface
global class MyAccountController {

  // The business method will look for a registered plugin
  public void businessMethod() {
    // Custom settings, if the user registered their custom plugins
    Registered_Plugin__c plugin = Registered_Plugin__c.getInstance(
        "query-plugin");
    // We determine the class name to use
    String className = plugin != null ?
        plugin.Class_Name__c :
        "DefaultQueryPlugin";
    // Instantiate the plugin
    Type t = Type.forName(className);
    IQueryPlugin queryPlugin = (IQueryPlugin) t.newInstance();
    // Use this plugin to prepend the WHERE clauses
    String queryString = "SELECT Id, Name FROM Account WHERE "
        + queryPlugin.getSoqlFragment();
    List<Account> accounts = Database.query(queryString);
    // Do something with the list of accounts...
  }

  // The plugin interface
  global Interface IQueryPlugin {
    String getSoqlFragment();
  }

  // The default plugin implementation (fallback)
  public class DefaultQueryPlugin implements IQueryPlugin {
    public getSoqlFragment() {
      return "MyCustomField__c = 'some value'";
    }
  }
}
{% endhighlight %}

### Customer registering a plugin

Your customer will then need to write their own implementation of the `IQueryPlugin`
interface, and register it. In our example, we have a
[custom setting](https://help.salesforce.com/apex/HTViewHelpDoc?id=cs_about.htm&language=en)
`Registered_Plugin__c` with a custom field `Class_Name__c`. We are checking for an instance
of such a registered plugin before falling back to our default plugin implementation.

The customer would write the following Apex class.

{% highlight java %}
public class CustomerPlugin implements MyAccountController.IQueryPlugin {
  public getSoqlFragment() {
    return "TheCustomerCustomField__c = 'the customer value'";
  }
}
{% endhighlight %}

By registering this plugin in the custom setting, under the instance name `query-plugin`, your
managed package code will know how to use the customer plugin implementation , since they will
implement your `IQueryPlugin` interface.

### Bonus #1 – Safety check

It's useful to use the [`instanceof` keyword](http://www.salesforce.com/us/developer/docs/apexcode/Content/apex_classes_keywords_instanceof.htm)
after getting back a `t.newInstance` or to catch a `TypeException` to make sure that your customer
did implement the proper interface.

{% highlight java %}
Type t = Type.forName(className);
Object obj = t.newInstance();
if (obj instanceof IQueryPlugin) {
  IQueryPlugin queryPlugin = (IQueryPlugin) obj;
}
{% endhighlight %}

After all, [thou shalt never trust User Input](https://www.ibm.com/developerworks/community/blogs/phpblog/entry/thou_shalt_never_trust_user?lang=en)!

### Bonus #2 - Dynamic Safety check

I faced this issue while attempting to check if `<insert class name here>` was a proper instance
of `<insert plugin class here>` in my code. Unfortunately, as of today, the current Apex code is **not** valid.

{% highlight java %}
Type t1 = Type.forName(className);
Type t2 = Type.forName(interfaceName)
Object obj = t1.newInstance();
if (obj instanceof t2) {
  // This won't compile on the force.com platform
}
{% endhighlight %}

If at runtime, you don't know what you want to cast your user's plugin into, making it
a dynamic plugin instantiation, you are stuck. When asking around me for a solution, I got
proposed the following workaround.

{% highlight java %}
Type classA = Type.forName("DefaultQueryPlugin");
Type classB = Type.forName("CustomQueryPlugin"); 
String s1 = JSON.serialize(classB.newInstance());
Object o2 = JSON.deserialize(s1, classA);
{% endhighlight %}

This is a very elegant solution, allowing you to dynamically check if `classB` is a proper
instance of child instance of `classA` using the `JSON.deserialize()` method.

But `JSON.deserialize()` won't work with abstract classes or interfaces. It will only work
with virtual classes (you need to be able to instantiate `classA`).

Still a good solution if you only ship virtual plugin classes (even with empty implementation)
but this give you a bit more of work and defeat the purpose of interfaces. And your customers
would need to always extends from those virutal plugin classes.

### Conlusion (for now)

Apex reflection is a really good tool, and a nice start to play with Apex Plugins. It does
need a bit of extra work from Salesforce to make it an excellent tool though.

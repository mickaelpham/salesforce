---
layout: post
title:  "Escaping special characters with dynamic SOQL"
categories: sfdc apex
---

For one of my project, I had to build an UI so my users can choose how to filter
a list of account (or any `SObject` in their organization). Among other things
I had to build a dynamic SOQL query string, similar to the one above.

{% highlight sql %}
SELECT Id, Name FROM Account WHERE Name LIKE '%userInput%'
{% endhighlight %}

The UI should allow users to:

* choose the `SObject` the query will be applied against;
* select the field against which the `WHERE` clause will be applied;
* enter any kind of string (`userInput`) for the value.

My Apex controller is using the following code to build the dynamic SOQL
query string (simplified on purpose for this post, the `SObject` and field
against which running the query are hardcoded here):

{% highlight java %}
// Bound to the VF page
public String userInput { get; set; }

// Bound to VF command button
public void doQuery() {
  // SOQL query string
  String q = "SELECT Id, Name FROM Account "
      + "WHERE Name LIKE \"%" + userInput + "%\"";
}
{% endhighlight %}

Here comes the fun part of properly escaping every single special character that
your users could enter in your input text field. And associated is the proper
escape sequence to use.

<!-- Needed to use raw HTML to add Bootstrap CSS class, if you know any better
please contact me! -->
<table class="table table-striped">
  <thead>
    <tr>
      <th>User Input</th>
      <th>Escape Sequence</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Single Quote</td>
      <td><code>\\\'</code></td>
    </tr>
    <tr>
      <td>Double Quotes</td>
      <td><code>"</code> (doesn't need one)</td>
    </tr>
    <tr>
      <td>Percentage</td>
      <td>N/A<sup>*</sup></td>
    </tr>
    <tr>
      <td>Underscore</td>
      <td>N/A<sup>*</sup></td>
    </tr>
    <tr>
      <td>Ampersand</td>
      <td><code>&</code> (doesn't need one)</td>
    </tr>
    <tr>
      <td>Backslash</td>
      <td><code>\\</code></td>
    </tr>
  </tbody>
</table>

The only thing to be aware of is the backslash escape character. Since you want
to escape a single quote character, you end up writing `\\\'` to indicate that this
single quote is to be escaped.

<sup>*</sup> Note from the
[Salesforce documentation](http://www.salesforce.com/us/developer/docs/officetoolkit/Content/sforce_api_calls_soql_select.htm):

> The `LIKE` operator in SOQL and SOSL does not currently support escaping of special
> character `%` or `_`.

Therefore, when your users will enter a `%` or a `_` in the text field, the generated
dynamic SOQL query will use those as wildcards.

#### Note

_Just like every other articles, I'm using the Java highlighter for my sample code
and therefore I need to repace the single quote `'` used in Apex code by double
quotes `"` in my blog, so the highlighter colors are correct._
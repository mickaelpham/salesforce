---
layout: post
title:  "Handlebars and JavaScript Remoting"
categories: sfdc apex
---

Ever since I started working with [JavaScript Remoting](http://www.salesforce.com/us/developer/docs/pages/Content/pages_js_remoting.htm)
in Salesforce, I looked for a template engine using JavaScript to simplify my HTML rendering in my VisualForce pages. There
are quite a few options here, but [this article](https://engineering.linkedin.com/frontend/client-side-templating-throwdown-mustache-handlebars-dustjs-and-more) LinkedIn explained how they choosed dust.js as their client-side templating solution.

I choose to learn more about [Handlebars.js](http://handlebarsjs.com/) and I figured out that this work nicely with JavaScript
Remoting. In this article, I will demo:

* How to construct your JavaScript Remote function
* How to build a simple template using [Handlebars.js](http://handlebarsjs.com/)

### Apex controller

```java
public class SubscribeController {

  /* Public Methods
  --------------------------------------------- */

  /**
   * Constructor
   */
  public SubscribeController() {
  }


  /* JS Remoting methods
  --------------------------------------------- */

  /**
   * Retrieve a Product Rate Plan from Zuora
   */
  @RemoteAction
  public static String findRatePlans(String searchString) {
    // Get a new Zuora ZAPi
    Zuora.ZApi zApi = new Zuora.ZApi();
    zApi.zLogin();
    // Prepare to query Zuora
    String zoql = "SELECT Id, Name, Description "
        + "FROM ProductRatePlan "
        + "WHERE Name LIKE '%" + searchString + "%\'";
    List < Zuora.ZObject > results = zApi.zQuery(zoql);
    // Prepare the list of returned product rate plan(s)
    List<ProductRatePlan> productRatePlans = new List<ProductRatePlan>();
    for (Zuora.ZObject zObject : results) {
      ProductRatePlan prp = new ProductRatePlan();
      prp.zuoraId = (String) zObject.getValue("Id");
      prp.name = (String) zObject.getValue("Name");
      prp.description = (String) zObject.getValue("Description");
      // Add this product rate plan to the list returned
      productRatePlans.add(prp);
    }
    return JSON.serialize(productRatePlans);
  }


  /* Inner Classes (wrappers)
  --------------------------------------------- */

  /**
   * Product Rate Plan wrapper
   */
  public class ProductRatePlan {
    public String zuoraId { get; set; }
    public String name { get; set; }
    public String description { get; set; }
    public List < ProductRatePlanCharge > charges { get; set; }
  }

  /**
   * Product Rate Plan Charge wrapper
   */
  public class ProductRatePlanCharge {
    public String zuoraId { get; set; }
    public List < ProductRatePlanChargeTier > tiers { get; set; }
  }

  /**
   * Product Rate Plan Charge Tier wrapper
   */
  public class ProductRatePlanChargeTier {
    public String zuoraId { get; set; }
    public Double listPrice { get; set; }
  }
}
```

### VisualForce Page

```html
<apex:page showHeader="true" sidebar="false" title="Subscribe" controller="SubscribeController">

  <input type="text" id="q" />
  <button onclick="queryZuora();">Find</button>

  <div id="ratePlanContent"></div>
  <div id="responseErrors"></div>

  <script type="text/javascript"
    src="{!URLFOR($Resource.DevResources, 'js/handlebars-v1.3.0.js')}">
  </script>
  <script type="text/javascript"
    src="{!URLFOR($Resource.DevResources, 'js/jquery-1.11.0.min.js')}">
  </script>

  <!-- Product Rate Plan template -->
  <script id="prp-template" type="text/x-handlebars-template">
    <table>
      <tbody>
        {{#rateplan}}
          <tr>
            <td>{{name}}</td>
            <td>{{description}}</td>
          </tr>
        {{/rateplan}}
      </tbody>
    </table>
  </script>
  
  <script type="text/javascript">

    var j$ = jQuery.noConflict();
    var ratePlanTemplate = null;

    j$(document).ready(function() {
      // Compile handlebars template
      var ratePlanSource = j$('#prp-template').html();
      ratePlanTemplate = Handlebars.compile(ratePlanSource);
    });

    function queryZuora() {
      // Retrieve the input value from the text field
      var searchString = document.getElementById('q').value;
      // Call the RemoteAction method
      Visualforce.remoting.Manager.invokeAction(
        '{!$RemoteAction.SubscribeController.findRatePlans}',
        searchString,
        function (result, event) {
          if (event.status) {
            // Render the result
            var data = { rateplan: j$.parseJSON(result) };
            console.log(data);
            j$('#ratePlanContent').html(ratePlanTemplate(data));
          } else if (event.type === 'exception') {
            j$('#responseErrors').html(event.message + '<br/>\n<pre>' + event.where + '</pre>');
          } else {
            j$('#responseErrors').html(event.message);
          }
        },
        { buffer: true, escape: false, timeout: 30000 }
      );
    }

  </script>

</apex:page>
```
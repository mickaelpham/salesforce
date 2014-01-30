---
layout: post
title:  "JSONParser class to serialize/deserialize SObjects"
categories: sfdc apex
---

The force.com platform is awesome. Really.

### My problem

I wanted to write a quick import/export feature for my application. Since I'm using some
[protected custom settings](https://help.salesforce.com/apex/HTViewHelpDoc?id=cs_accessing.htm&language=en)
to store the records I want to export, it's useful to think ahead and anticipate the fact
that your customers will often have multiple Salesforce organization (sandboxes, etc.) and
you do not want them to manually set up their environment.

### Goals

* Clean the custom field records from all unecessary data
  * This include system fields like `SetupUserId`, `LastModifiedDate`, etc.
* Export the records as a JSON file
* Upload the file in another organization
* Import (= insert) the records from the JSON file in the new organization

### Export the records

Since I want to export the records as a JSON file, I first need to create a VisualForce page
with the following header:

{% highlight html %}
<apex:page cache="true" contentType="application/json#{!filename}.json"
           showHeader="false" sidebar="false" standardStylesheets="false"
           controller="MyExportController">
  <apex:outputText value="{!jsonString}" escape="false" />
</apex:page>
{% endhighlight %}

And the associated controller:

{% highlight java %}
public class MyExportController {

  public String filename { get; set; }

  public String jsonString { get; set; }

  public MyExportController() {
    // Append the date time to the filename
    this.filename = "records-" + DateTime.now().format("yyyy-MM-dd-hh-mm-ss");
    // Retrieve the records (it's a custom setting)
    List<MyRecord__c> records = MyRecord__c.getAll().values();
    // Serialize the list
    jsonString = JSON.serializePretty(records);
  }
}
{% endhighlight %}

And the produced JSON file:

{% highlight json %}
[ {
  "attributes" : {
    "type" : "MyRecord__c",
    "url" : "/services/data/v29.0/sobjects/MyRecord__c/a0hi000000287sNAAQ"
  },
  "LastModifiedById" : "005i0000001TV5aAAG",
  "LastModifiedDate" : "2014-01-30T03:29:32.000+0000"
  "Name" : "MyCustomComponent",
  "SetupOwnerId" : "00Di0000000dqmLEAQ",
  "SystemModstamp" : "2014-01-30T03:29:32.000+0000",
  "CreatedById" : "005i0000001TV5aAAG",
  "CreatedDate" : "2014-01-30T03:29:32.000+0000",
  "IsDeleted" : false,
  "Id" : "a0hi000000287sNAAQ",
  "Description__c" : "Used in my custom flow",
  "Is_Protected__c" : false
}, {
  "attributes" : {
    "type" : "MyRecord__c",
    "url" : "/services/data/v29.0/sobjects/MyRecord__c/a0hi000000287sIAAQ"
  },
  "LastModifiedById" : "005i0000001TV5aAAG",
  "LastModifiedDate" : "2014-01-30T03:28:13.000+0000"
  "Name" : "MyList",
  "SetupOwnerId" : "00Di0000000dqmLEAQ",
  "SystemModstamp" : "2014-01-30T03:28:13.000+0000",
  "CreatedById" : "005i0000001TV5aAAG",
  "CreatedDate" : "2014-01-30T03:28:13.000+0000",
  "IsDeleted" : false,
  "Id" : "a0hi000000287sIAAQ",
  "Description__c" : "Selector used in the quote flow",
  "Is_Protected__c" : true
} ]
{% endhighlight %}

Unfortunately, with this approach, we retrieve all fields, including the system fields. Plus,
the `Id` field is populated with the record current ID, but we want to avoid this if we want
to insert those records in another Salesforce organization.

To solve this, we use the `clone()` SObject instance method (see the
[documentation](http://www.salesforce.com/us/developer/docs/apexcode/Content/apex_System_SObject_clone.htm))
, and we also manually query the interesting fields (typically, all the custom fields,
not the system-related ones) manually. So our controller becomes:

{% highlight java %}
public class MyExportController {

  public String filename { get; set; }

  public String jsonString { get; set; }

  public MyExportController() {
    // Append the date time to the filename
    this.filename = "records-" + DateTime.now().format("yyyy-MM-dd-hh-mm-ss");
    // Retrieve the original records
    List<MyRecord__c> originalRecords = [
      SELECT Name, Description__c, Is_Protected__c
      FROM MyRecord__c LIMIT 10000
    ];
    // Blank the ID fields using the clone() method
    List<MyRecord__c> records = new List<MyRecord__c>();
    for (MyRecord__c r : originalRecords) {
      records.add(r.clone(false, false, false, false));
    }
    // Serialize the list
    jsonString = JSON.serializePretty(records);
  }
}
{% endhighlight %}

So we end up with a much cleaner JSON file:

{% highlight json %}
[ {
  "attributes" : {
    "type" : "MyRecord__c"
  },
  "Name" : "MyCustomComponent",
  "Description__c" : "Used in my custom flow",
  "Is_Protected__c" : false
}, {
  "attributes" : {
    "type" : "MyRecord__c"
  },
  "Name" : "MyList",
  "Description__c" : "Selector used in the quote flow",
  "Is_Protected__c" : true
} ]
{% endhighlight %}

### Upload the file

In a VisualForce page, simply use the `apex:importFile` tag like this:

{% highlight html %}
<apex:inputFile value="{!document.body}" id="importFile" />
{% endhighlight %}

And in the associated controller, you will be able to read the body
of the uploaded file. Now, it's time to import those record in the
new Salesforce organization.

### Import the records

This is dead simple with Salesforce, since we can deserialize a JSON string
into an SObject instance. Here is the way I did it (but I'm sure there are
better ways, more optimized).

{% highlight java %}
/**
 * Import the file and save it to the database
 */
public void doImportFile() {
  JSONParser parser = JSON.createParser(document.body.toString());
  List<MyRecord__c> records = null;
  while (parser.nextToken() != null) {
    // Read the value as a list of MyRecord__c object
    records = (List<MyRecord__c>) parser.readValueAs(List<MyRecord__c>.class);
  }
  insert records;
}
{% endhighlight %}

And we are all set!
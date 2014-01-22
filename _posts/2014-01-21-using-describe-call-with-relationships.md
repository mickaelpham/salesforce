---
layout: post
title:  "Using the Describe call with relationships fields"
categories: sfdc apex
---

With the force.com platform and Apex, it's really easy to describe an object and
all of its fields. But you can do better and also explore all the related objects
through the relationship fields (either Master-Detail or Lookup fields).

The following snippet demonstrates how to describe a `MyObject__c` custom object
and all of its field, using only **one describe** call.

{% highlight java %}
// First retrieve the object type
Schema.SObjectType objectType = Schema.getGlobalDescribe().get("MyObject__c");
// Get the object associated describe result (cost: 1 describe call)
Schema.DescribeSObjectResult objectDescribe = objectType.getDescribe();
// Store the object fields outside of the loop and avoid consuming any more
// describe calls
Map<String, Schema.SObjectField> objectFields = objectDescribe.fields.getMap();
// Loop through the fields
for (String fieldName : objectFields.keySet()) {
    // Retrieve the field describe
    Schema.DescribeFieldResult dfr = objectFields.get(fieldName).getDescribe();
    // Do whatever you want with it
    System.debug("Retrieved " + dfr.getLabel() + " of type " + dfr.getType());
}
{% endhighlight %}

In the example above, we are simply using the `DescribeFieldResult` to get the
field label and field type. But we can also use it to explore relationships. Nothing
that a good recursive method can't do.

{% highlight java %}
// the following function returns a list of all fields that can be reached
List<String> exploreFields(Schema.SObjectType objectType, Integer level) {
  // prepare the list of fields to return
  List<String> fields = new List<String>();
  if (cachedObjects.get(objectType) == null) {
    cachedObjects.put(objectType, objectType.getDescribe());
  }
  // get the describe result
  Schema.DescribeSObjectResult objectDesc = cachedObjects.get(objectType);
  // get the fields
  Map<String, Schema.SObjectField> objectFields = objectDesc.fields.getMap();
  // loop throug the fields, and if one is a relationship, recursively call
  // the function
  for (String fieldName : objectFields.keySet()) {
    // Retrieve the field describe
    Schema.DescribeFieldResult dfr = objectFields.get(fieldName).getDescribe();
    // if the field is a relationship
    if (dfr.getReferenceTo().size() == 1
        && String.isNotBlank(dfr.getRelationshipName())
        && level < MAX_LEVEL) {
        fields.addAll(exploreFields(dfr.getReferenceTo()[0]), level + 1);
    }
    else {
      fields.add(dfr.getLabel());
    }
  }
  return fields;
}

// it's wise to cache the retrieved DescribeSObjectResult to avoid hitting the
// Salesforce governor limits
Map<Schema.SObjectType, Schema.DescribeSObjectResult> cachedObjects;

// you also need to prevent a too deep exploration of relationship
static final Integer MAX_LEVEL = 3;
{% endhighlight %}

The key points here are:

* cache your `DescribeSObjectResult` so you don't have to make more `getDescribe()` calls than necessary
* use recursivity to explore relationship fields, but limit your self to a certain deep-level
* this also prevents you from infinitely looping between two objects with a relationship to each other

You are all set!
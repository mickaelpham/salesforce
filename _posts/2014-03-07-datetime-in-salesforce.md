---
layout: post
title:  "DateTime in Salesforce"
categories: sfdc apex
---

_Disclaimer: this article is more about my experiments with the Force.com platform
and might not be 100% accurate when reproduced on your own tenant. But it can help
you with the right questions to ask._

### Inserting a record

When inserting a record with a `DateTime` field, Salesforce will use the connected
user's Time Zone information&mdash;from the profile, available under "Personal Setup"
&rarr; "My Personal Information" &rarr; "Personal Information" &rarr; "Time Zone."

(note: I'm in `(GMT+01:00) Central European Time (Europe/Paris)` for this example)

```java
Contact c = new Contact(
  FirstName="First",
  LastName="Last",
  DateTimeField__c=DateTime.now()
);
insert c;
```

This will insert a contact in the datbase with the following datetime:
`2014-03-07 10:56:00` (in GMT). The conversion happens during the insertion.

Observations:

* Salesforce stores datetime field in GMT
* The timezone used for conversion comes from the user's profile

### Retrieving a record

However, when retrieving a record&mdash;in Apex code&mdash;the GMT value is returned
by default:

```java
List<Contact> contacts = [SELECT DateTimeField__c FROM Contact];
for (Contact c : contacts) {
  System.debug(c.DateTimeField__c);
}
```

The code above will display `DEBUG|2014-03-07 10:56:00` in **GMT**. However, we can use the
`format()` method from the `DateTime` instance to change this:

```java
// [...]
System.debug(c.DateTimeField__c.format("yyyy-MM-dd\'T\'hh:mm:ssZ"));
```

Again, this method relies on the connected user's timezone, as defined in his/her
profile. Therefore, with my user in `(GMT+01:00) Central European Time (Europe/Paris)`
this example will displays:

```
DEBUG|2014-03-07T11:56:00+0100
```

However using a different user (or updating your profile) with a different
timezone like `(GMT-08:00) Pacific Standard Time (America/Los_Angeles)` will
display the date time converted in the proper time zone:

```
DEBUG|2014-03-07T02:56:00-0800
```

### Criterion using DateTime field

We know that Salesforce stores `DateTime` fields in GMT timezone. Now how does
this behave when using dynamic SOQL to query such records?

As an extra difficulty (in my case) I only have a `String` representation (without
the timezone information) stored in the database. Why without the timezone information?
Well, the `valueOf(String)` method from the `DateTime` object does not accept
the timezone information:

> The specified string should use the standard date format "yyyy-MM-dd HH:mm:ss"
> in the local time zone.

[Documentation](http://www.salesforce.com/us/developer/docs/apexcode/Content/apex_methods_system_datetime.htm#apex_System_Datetime_valueOf)

We will have the following steps

1. Capture the datetime user input (`String`) in his/her timezone
2. Convert this input in GMT before inserting it in the db
3. At runtime (while building the criterion) use the proper GMT format
4. If the user whishes to edit this criterion, convert it back in his/her timezone

This way we are storing the user input in GMT format in the db, so we can use it as a
criterion directly in GMT format. However, if the user wishes to edit this
criterion, we present it in his/her timezone.

```java
// This would come from our VF Page
String userInput = "2014-03-01 09:30:00";
System.debug("### User Input = " + userInput);

// We convert it to GMT before inserting the record
String storedInDb =
  DateTime.valueOf(userInput).formatGmt("yyyy-MM-dd HH:mm:ss");
System.debug("### Stored in db = " + storedInDb);

// At runtime, we generate the proper format to be used as a criterion
String atRuntime =
  DateTime.valueOfGmt(storedInDb).formatGmt("yyyy-MM-dd\'T\'HH:mm:ss\'Z\'");
System.debug("###  At runtime = " + atRuntime);

// If the user wants to edit, we convert back to his/her timezone
String userEdit =
  DateTime.valueOfGmt(storedInDb).format("yyyy-MM-dd HH:mm:ss");
System.debug("### User Edit = " + userEdit);
```

As a best practice, while working with Salesforce, I would always recommend using
GMT to store a date in it's string representation, because it's really easy to get
the timezone of the connected user, and all `DateTime` methods like `valueOf` or
`format` do this conversion (to the user's timezone) for you.



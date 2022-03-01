# Forms in InterSystems Reports

Recently I needed to to create a report in InterSystems Reports, which was not the usual _table_, but rather a form, to be specific a bill (but it does not matter that it was a bill specifically, just that it was not a _table_ or a _chart_). This article deals with the case where you need to create a report which is an invoice a bill or something of a similar nature. 

I would also describe several tricks, which might come useful. This article assumes familiarity with InterSystems Reports Designer. Check out [Getting Started with InterSystems Reports](https://learning.intersystems.com/course/view.php?id=1538) if you want to learn the basics of creating reports in InterSystems Reports Designer. 

# Report types

First, let's talk about different report types. While this is my own taxonomy, having done several PoCs using InterSystems Reports I arrived at the following three report types:

1. Table reports. The simplest report there is, table report is, essentially, a result of a query execution piped into a pdf or excel file. They are created for two main purposes: to denote a state of a database at some specific point of time OR to be a base for some adhoc analysis (for the better or worse of it). Analysis can be done entirely by the consumer or parts of the analysis (grouping, filtering, etc.) can be offloaded to the reports server

2. Chart reports. Unlike table reports, which are one query per report, chart reports use one query per chart, so there are a lot of independent queries. Chart reports are used for visualisation in a case where we don't want user to be able to modify it (unlike BI tools where base dashboard/widget usually IS modifyalbe by the end user - at least to some extent). 

3. Form reports. They are a more challenging report type combining one basic query and several related subqueries. If you want to create a bill, invoice or something similar - you'll need a form report.  

# Data model

The main difference with tha data model (comperad to the normal table or chart reports) is that you need to use queries of a different cardinality together.

In our example we will use two tables - customers and items. We will create a billing report, where each customer has a separate bill and each bill has a varying number of items.

Here's our data model. First we have a client:

```objectscript
Class report.Client Extends %Persistent
{

Property Name As %String;

Property Address As %VarString;

/// Client account number in some billing system
Property BillingOptionA As %Integer;

/// Client account number in another billing system
Property BillingOptionB As %Integer;
}
```

Note that client has two billing options - accounts in some external billing systems (for example banks or something similar). Where we generate a report we want to show all availiable billing options (client can have only one billing option for example).

Next there are orders - each client has a random number of orders to include in a bill: 

```objectscript
Class report.Orders Extends %Persistent
{

Property Client As Client;

Property Item As %String;

Property Amount As %Integer;
}
```

The form report we want to produce is a bill - one page per client listing all his orders and billing information.

So to start with we need to create 1 base query (clients) and 1 subquery (items). Since there's no connections so far just add a whole table as a query.

# Running an example

If you want to get a running demo, you can either start this project in [Docker](https://github.com/eduard93/reports).

1. Load `iris.xml` into InterSystems IRIS
2. Execute:

```
do ##class(report.Client).Populate()
do ##class(report.Orders).Populate()
```
3. Create a new folder and place `report.cat.xml` and `reportset.cls.xml` there.
4. In `report.cat.xml` modify `JDBCConnection` to connect to an InterSystems IRIS instance in (1).
5. Open `report.cat.xml` connection in InterSystems Reports Designer (version 18.2 or later).
6. Open `reportset.cls.xml` report in InterSystems Reports Designer.

# Initial work

1. Create `Clients` and `Orders` queries:

![image](https://user-images.githubusercontent.com/5127457/156050226-a3a743fd-4913-4971-b006-388797babb2f.png)
![image](https://user-images.githubusercontent.com/5127457/156050298-849f7a59-60c6-4e27-82a1-f8bd5e445c6d.png)

2. Create a new report of a `Banded `type:

![image](https://user-images.githubusercontent.com/5127457/156051275-d99acb48-8149-44b6-96db-c26e7c0d32ec.png)

3. Choose `Clients` query as a data source and add all properties:

![image](https://user-images.githubusercontent.com/5127457/156051517-7bddbe37-3ac3-44b0-a0c7-0029d7777085.png)

![image](https://user-images.githubusercontent.com/5127457/156051602-c7c2c222-cb7f-4f8a-a1eb-b3a6d5cb9792.png)

4. You should get something like this:

![image](https://user-images.githubusercontent.com/5127457/156051668-341def38-15c6-4232-b6dd-c8ab2a27f8ae.png)

Or displayed:

![image](https://user-images.githubusercontent.com/5127457/156051735-efb623b4-2c51-4ab7-8a48-69652252515d.png)

5. Let's reformat it a little to make it look more like a bill and also hide ID column:

![image](https://user-images.githubusercontent.com/5127457/156052803-f43a2ffd-3af6-4b7c-9918-959f6e54a462.png)

Our report starts looking like a bill:

![image](https://user-images.githubusercontent.com/5127457/156052915-15f29bf1-7ca2-4710-b39a-0a75b74f4341.png)

# Connecting item queries

Now we need to add orders to our bill. Add a new Table (Group Left) to the banded object. Choose `Orders` dataset:

![image](https://user-images.githubusercontent.com/5127457/156054658-8b68dbfe-cac3-4290-906e-72645db06754.png)

New table inside the banded object:

![image](https://user-images.githubusercontent.com/5127457/156054760-17032186-f1ab-4332-a5b0-8e44b4ca6ab0.png)

But if you display it as is it would show all orders for every client. To prevent that press 🖱️ RMB on the orders table and choose `Data Container Link...` option:

![image](https://user-images.githubusercontent.com/5127457/156055374-b5d3d4f6-1ff0-41ba-8420-9a07bdd47a08.png)

In there link Client ID and Order Client:

![image](https://user-images.githubusercontent.com/5127457/156055465-09f3a22b-ccd3-4a68-9bf3-79f853c4d4aa.png)

Now our bill displays only Orders for a current client:

![image](https://user-images.githubusercontent.com/5127457/156055557-a3e5d0ed-93e0-4901-b434-da4e6ff30426.png)

That's pretty much it. Next let's discuss some tricks.

# Formulas

Formaulas is a powerful instrument allowing arbitrary computations over base query properties. 

## Builtins

Most properties can be set to formulas as opposed to constants. For example, our client may have only one billing option and we need to hide a corresponding QR code if the billing option value is empty. To do that set `Invisible` property to `IsNull(@BillingOptionB)` (press `Fx` to enter a formula, [list of supported functions](https://reportkbase.logianalytics.com/designer16/userguide/index.htm#t=HTML%2Fappendix%2Fapdx_fctn.htm), also available in Formula wizard):

![image](https://user-images.githubusercontent.com/5127457/156154786-801b250f-cea1-4c6d-a494-28938b7fceb5.png)

This way if the value of `BillingOptionB` is null (or empty string as the case may be), the QR code is not displayed at all.

## Custom

Next let's align Address string to take more space if `BillingOptionB` QR code is not displayed. There's no ready made function for that, so we need to write our own. The language of choice here is Java. Press `<New Formula...>` button and create a new formula named `WidthFormula` with this code `if (IsNull(@BillingOptionB)) then return 3.72 else return 2.72`:

![image](https://user-images.githubusercontent.com/5127457/156157740-891261af-8c76-47e9-8da5-657a37873d48.png)

And set it as Width value in address:

![image](https://user-images.githubusercontent.com/5127457/156166662-c6c36d97-68d9-4382-ba92-9acbe4adfcd1.png)

Now render for ClientA and ClientB would look like this:

![image](https://user-images.githubusercontent.com/5127457/156166730-b76d9ecf-7b7f-4661-8d3e-b8ff2aeba68d.png)

![image](https://user-images.githubusercontent.com/5127457/156166754-148a78d1-ca3d-406a-a4bd-03845e62e4af.png)

Note that address now takes all availiable space.


# Several data panes

What if we need two tables, one after the other? In that case we need to add another Detail Panel. Press 🖱️ RMB on existing Detail panel and click `Insert Panel After`. 
After that add a table to a second Detail Panel.

![image](https://user-images.githubusercontent.com/5127457/156169355-81e5f9dd-1925-4ed0-8dec-7a1f7f58777c.png)

Tables do not overlap:

![image](https://user-images.githubusercontent.com/5127457/156169668-e60e0a20-2ed1-4d51-8667-3cda824716d9.png)

# Vertical text

If you want to add vertical text (actually any non-horizontal text) do by adding a UDO component of a JRptator type:

![image](https://user-images.githubusercontent.com/5127457/156169824-318aeb95-3f8c-4816-80d4-f92ea0a2ef0b.png)

And set it's `Rotate` property to `-90`:

![image](https://user-images.githubusercontent.com/5127457/156169948-ffe7a394-061a-414e-a4b0-c4f017920f4c.png)


# Summary

# Links

- [Getting Started with InterSystems Reports](https://learning.intersystems.com/course/view.php?id=1538)
- [Running InterSystems Reports in containers](https://community.intersystems.com/post/running-intersystems-reports-containers)
- [Repository](https://github.com/eduard93/reports)
- [List of supported functions](https://reportkbase.logianalytics.com/designer16/userguide/index.htm#t=HTML%2Fappendix%2Fapdx_fctn.htm)

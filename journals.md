# Restoring production class using journals

Recently I needed to restore a version of a production class, which was overwritten by compilation and running UpdateProduction. As the correct version was unavailable in the source control, I used journals to restore the data. Journals store a plethora of information about what's happening in the system and are quite a powerful tool. This article explains how to work with journals to extract the data you require.

# Journaling

Global journaling records all global update operations performed on a database. Let's see what it looks like. Go to SMP > System Operation > Journals. It lists all available journals. Journals follow the naming pattern: `YYYYMMDD.N` where `N` goes from `001` to `999`:

![image](https://github.com/eduard93/Articles/assets/5127457/d4c626e6-4b66-4b02-9d2d-dc69e4abdfa3)

I recommend exploring journals on an idle system, as high-load systems can have a large number of journal records. Click on the latest journal (first in the list). Now open a separate terminal window and get your jobid (`write $job`). Set process filter to your jobid. You should have one record:

![image](https://github.com/eduard93/Articles/assets/5127457/b45efc9e-72c7-4e5b-88d3-d1f48c9bd9cc)

Let's try to change something. In my terminal, I'll run `set ^a=123`, and here's what appears in journals (click on the Search button to rerun the search):

![image](https://github.com/eduard93/Articles/assets/5127457/d5fa942f-41f5-47f1-ab18-e3ef95ff1c06)

We can see some basic parameters such as type, global, and db, but to see all properties, click on the Offset value:

![image](https://github.com/eduard93/Articles/assets/5127457/f17ced75-da01-406c-b7b5-1a4e423fa89b)

You should see something like this:

![image](https://github.com/eduard93/Articles/assets/5127457/2a78f6d4-8075-4b5e-ab73-55605741d36a)

For programmatic access, we'll use [%SYS.Journal.Record](https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25SYS.Journal.Record), and it's subclasses, mostly [%SYS.Journal.SetKillRecord](https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25SYS.Journal.SetKillRecord).

Note that Old Value is empty. That is not because `^a` did not exist beforehand (validate it by running `set ^a=456` and checking created journal record, which would NOT have Old Value set), but because we are not in transaction. Since there's no need to rollback, Old Values are not stored. To compare, let's run the following code:

```objectscript
TSTART
set ^a=789
TCOMMIT
```

![image](https://github.com/eduard93/Articles/assets/5127457/8a739bf9-1722-4087-aea3-99bb3236a8bf)

and the SET record now has an Old Value:

![image](https://github.com/eduard93/Articles/assets/5127457/2687d333-4a83-4814-a5cb-53c7fc2f214b)

Now that we got the basic idea about journaling let's discuss programmatic approaches to accessing journal records. There are two ways to do that: SQL and Objects.

## SQL

Use [%SYS_Journal.Record_List](https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25SYS.Journal.Record#anchor_queries) query. Here's an example of how to get all changes in a transaction:

```sql
SELECT * 
FROM %SYS_Journal.Record_List('c:\intersystems\iris\mgr\journal\20230607.002')
WHERE 1=1 AND
      ProcessID=10336 AND 
      InTransaction=1
```

This query returns the same records as an SMP Journaling page. 

![image](https://github.com/eduard93/Articles/assets/5127457/102e53f2-83fc-4a6a-8806-12a27083eeb8)

It does not matter for demo purposes, but the `Record_List` query accepts `Offsets` and `Match` arguments, which work much faster than SQL:

```sql
SELECT * 
FROM %SYS_Journal.Record_List('c:\intersystems\iris\mgr\journal\20230607.002',,,,$listbuild('ProcessID','=','10336'))
WHERE 1=1 AND
      InTransaction=1
```

![image](https://github.com/eduard93/Articles/assets/5127457/73ce2e57-7c92-4e41-ac3d-88f2b32a3ed5)

Since `Match` accepts only one condition, make sure to move the condition with the highest selectivity there. If you need to rerun the query on the same data, use `Offset`.

## Objects

Here's how we can iterate journal records from objectscript: 

```objectscript
ClassMethod Scan()
{
    Set FilePath = "c:\intersystems\iris\mgr\journal\20230607.002"
    Set jrnforef = ##class(%SYS.Journal.File).%OpenId(FilePath)
    set record = jrnforef.FirstRecord
    while record '="" {
        if record.%IsA("%SYS.Journal.SetKillRecord"), 
           record.DatabaseName="c:\intersystems\iris\mgr\user\", 
           record.ProcessID=10336,
           record.InTransaction,
           1 {
            w "set ",record.GlobalNode,"="
            zw record.OldValue
        }
        set record = record.Next
    }
}
```

The code would output `set ^a=123`. Tthe API is largely identical to SQL.

# Production

Back to our production issue. Production definition is stored in XData named `ProductionDefinition`. Assuming our production class is called `User.Production` and consulting `%Dictionary.XDataDefinition`'s `%LoadData` method we can construct the start of glvn: `^oddDEF("User.Production","x","ProductionDefinition"`. Let's run this query:

```sql
SELECT GlobalNode, NewValue, OldValue
FROM %SYS_Journal.Record_List('c:\intersystems\iris\mgr\journal\20230607.002',,,,$listbuild('GlobalNode','[','^oddDEF("User.Production","x","ProductionDefinition"'))
```

![image](https://github.com/eduard93/Articles/assets/5127457/4e88f233-ea3b-45b7-bae5-ee3016df1378)

From that we can:

1. Reconstruct XData.
2. Stop production.
3. Replace XData.
4. Start production.

In the same way, `^oddDEF` contains definitions for all parts of the code - the most interesting would be the method code as it's also stored in strings (properties, indices, and other similar elements are stored in a structured they so reconstructing them is possible but requires either writing generation code or re-executing `set`s against `^oddDEF`). 

However, in my case, the previous good compilation was too long ago, and the only recent compilation had only new values (which I didn't need). However, the production class is a special case, as it is also stored in the `Ens.Config` package as objects (base `Production` object and `Item` objects), and they are always saved in a transaction. 

Let's try to see if there are any `Ens.Config.Item` changes. There are:

```sql
SELECT GlobalNode, NewValue, OldValue
FROM %SYS_Journal.Record_List('c:\intersystems\iris\mgr\journal\20230607.002',,,,$listbuild('GlobalNode','[','^Ens.Config.ItemD'))
WHERE 1=1 AND
      InTransaction=1
```
![image](https://github.com/eduard93/Articles/assets/5127457/457aef5b-3555-47b2-95a4-8ff395834456)


However, SQL does not display `$lb` structures, so we need to rewrite our query as code:

```objectscript
ClassMethod ScanEns()
{
    Set FilePath = "c:\intersystems\iris\mgr\journal\20230607.002"
    Set jrnforef = ##class(%SYS.Journal.File).%OpenId(FilePath)
    set record = jrnforef.FirstRecord
    while record '="" {
        if record.%IsA("%SYS.Journal.SetKillRecord"), 
           record.DatabaseName="c:\intersystems\iris\mgr\user\", 
           (record.GlobalNode["^Ens.Config.ItemD") || (record.GlobalNode["^Ens.Config.ProductionD"),
           record.InTransaction,
           1 {
            w "set ",record.GlobalNode,"="
            zw record.OldValue
        }
        set record = record.Next
    }
}
```

Here's what it returns:

```objectscript
set ^Ens.Config.ItemD(1)=$lb("","User.BS","",1,0,,"User.BS",1,"","","",0,"","","")
set ^Ens.Config.ItemD(2)=$lb("","User.BO","",0,0,,"User.BO",1,$lb($lb($lb("IPAddress","Adapter","127.0.0.1"))),"","",0,"","User.Production","")
set ^Ens.Config.ProductionD("User.Production")=$lb("",1,"Production",$lb($lb("1"),$lb("2")),0,1,"","")
```

The production contains a `$list` of items, and items contain all their settings. Now we are ready to reconstruct the production class (it's safer to do it in a separate empty namespace):

1. Clear old data:

```objectscript
do ##class(Ens.Config.Item).%KillExtent()
do ##class(Ens.Config.Production).%KillExtent()
```

2. Run `set` commands we got from journals (possibly adjusting them depending on your requirements).
3. Reconstruct the class:
```objectscript
do ##class(Ens.Config.Item).%BuildIndices()
set prodname = "User.Production"
set prod = ##class(Ens.Config.Production).%OpenId(prodname)
zwrite prod.SaveToClass()
```
It should create (or update) `User.Production` class with XData constructed from the `Ens.Config.Item` objects.

# Conclusion

Journals can help you to inspect or rollback unwanted changes. After you have identified a journal with the changes you need, you can copy it locally for further investigation (rollover to a new journal if it's still an active journal). If you don't know which global was affected, perform a similar action on an idle system and check which journal records were created as a result.

# References
- [%SYS.Journal.Record](https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25SYS.Journal.Record) 
- [%SYS.Journal.SetKillRecord](https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls?LIBRARY=%25SYS&CLASSNAME=%25SYS.Journal.SetKillRecord).

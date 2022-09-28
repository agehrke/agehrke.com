---
title: "Fixing Sitecore Fast Query - querying by field ID"
date: "2014-12-15"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

Sitecore Fast Query is a XPath like query syntax for retrieving and filtering items from the Sitecore database, which uses the database engine to execute queries. This means a Fast Query is directly translated to a SQL query against one of the Sitecore SQL databases. The key tables/views used in the SQL queries are _Items_ table and _Fields_ view, which queries _VersionedFields_, _UnversionedFields_, and _SharedFields_ tables. TL;DR - Are you only interested in the code to fix Sitecore Fast Query and be able to query by field ID, [see this gist](https://gist.github.com/agehrke/17b1e11807e8b37692bd).

### Motivation

So you might be thinking: why use a Sitecore Fast Query at all? Sitecore has _Sitecore Search_ based on Lucene or Solr indexes! Well, sometimes you require up-to-date data, and can't rely on the sometimes stale data in a Lucene index.

### Fast Query example

Let's look at an example of a Sitecore Fast Query, and the SQL generated.

Fast Query:

fast://\*\[@@templateid='{C29492D2-6F79-463C-A7FB-B318FAB90639}' and @#Postal Code#='43870'\]

Generated SQL: \[sql\] exec sp\_executesql N'SELECT DISTINCT \[i\].\[ID\] \[ID\], \[i\].\[ParentID\] \[ParentID\] FROM \[Items\] \[i\] LEFT OUTER JOIN ( SELECT \[Fields\].\* from \[Fields\] INNER JOIN \[Items\] ON \[Fields\].\[FieldID\] = \[Items\].\[ID\] AND lower(\[Items\].\[Name\]) = ''postal code'' ) \[Fields1\] ON \[i\].\[ID\] = \[Fields1\].\[ItemId\] WHERE ((\[i\].\[TemplateID\] = @value1) and (coalesce(\[Fields1\].\[Value\], '''') LIKE @value2))',N'@value1 nvarchar(38),@value2 nvarchar(5)',@value1=N'{C29492D2-6F79-463C-A7FB-B318FAB90639}',@value2=N'43870' \[/sql\]

The query searches for all items of a specific templateid which has a _Postal Code_ field with value '43870'. As you can see the SQL query uses a subselect on _Items_ and _Fields_ to get the field details for the _Postal Code_ field. We should be able to eliminate the join on _Items_ table, if we know the ID of the field we are querying. But how do you specify a field ID instead of name in a Sitecore Fast Query? Turns out no documentation exists - like so many other Sitecore features... So we turn to the best documentation tool for Sitecore, [a decompiler, such as dotPeek](http://www.agehrke.com/2014/11/sitecore-debugging-dotpeek/ "Sitecore debugging with dotPeek"). My colleague Carsten had already asked him self the same question, and had the answer.

### How to query by field ID

Use `@f012901edc1e449b79077eedc19cab81d` as field name. Notice the leading _f_ followed by a GUID, your field ID. The implementation in `Sitecore.Data.DataProviders.Sql.FastQuery.FieldTranslator` expects a field name of 33 characters, including the _f_, which means that you can not include dashes in your guid. Tip: Use the `Sitecore.Data.ID.ToShortID()` method to convert your field id into the proper format.

Now, one would expect that the query above could be re-written to:

fast://\*\[@@templateid='{C29492D2-6F79-463C-A7FB-B318FAB90639}' and @f012901edc1e449b79077eedc19cab81d='43870'\]

Unfortunately this does not work, and throws an error.

### Fixing querying by field ID - take 1

When using the XPath Builder tool in Sitecore and the query above, you get the error: _Index (zero based) must be greater than or equal to zero and less than the size of the argument list_. You might recognize the text, it's the exception message thrown from `string.Format`. Running the Fast Query manually through the `Sitecore.Data.Database.SelectSingleItem()` method, you will see that exception is thrown deep inside `Sitecore.Data.DataProviders.Sql.SqlDataApi` - if I remember correctly. Having a closer look at the implementation of `Sitecore.Data.DataProviders.Sql.FastQuery.FieldTranslator` one sees that a `IDFieldInfo` class is used to build the field querying part - the part in the SQL subselect above. The implementation of `IDFieldInfo` is quite simple, the important part is this line:

\[csharp\] context.AddSubquery(context.SqlApi.Format("(SELECT \* FROM {0}Fields{1} WHERE {0}FieldID{1}='") + context.SqlApi.FormatRawID(fieldId) + "')", "Fields") \[/csharp\]

The `fieldId` variable is a `Sitecore.Data.ID` instance, and we all know how the string representation of an ID looks: _{3db0ff43-3912-4fe0-9f75-9e26e2051a22}_. But hey! Curly brackets have a special meaning in `string.Format` and should be escaped using `{{` and `}}`. So how does the `FormatRawID()` method format an ID? Yup, you guessed it: `id.ToString()`.

So if we can fix the `FormatRawID()` method in `Sitecore.Data.DataProviders.Sql.SqlDataApi` class, we should have a working Sitecore Fast Query. Fortunately the SqlDataApi class is _public abstract_ and the method _protected virtual_ - two minutes later, we have a new implementation of `FormatRawID()`. Actually Sitecore has two implementations of the SqlDataApi class, one for SQL Server and one for Oracle. Since we are working with a SQL Server, so we inherit from `Sitecore.Data.SqlServer.SqlServerDataApi`. But how do we instruct Sitecore to use our new `SqlServerDataApi` implementation? A quick scan through Sitecore's web.config reveals:

\[xml\] <dataApis> <!-- Data api for accessing SQL Server databases. --> <dataApi name="SqlServer" type="Sitecore.Data.SqlServer.SqlServerDataApi, Sitecore.Kernel"> <param connectionStringName="$(1)" /> </dataApi> </dataApis> \[/xml\]

Wow! This is exactly what we need! So we change the type attribute to point to our new class, and expect to have a working Sitecore Fast Query. But our query still fails, and the debug breakpoint in `FormatRawID()` is never reached. Turns out Sitecore doesn't use the config setting for anything - the setting has the exact same value in the web.config for Oracle.

### Fixing querying by field ID - take 2

Instead you have to replace the entire `SqlDataProvider`, in our case the `Sitecore.Data.SqlServer.SqlServerDataProvider`. But this class has the following constructor:

\[csharp\] public SqlServerDataProvider(string connectionString) : base((SqlDataApi) new SqlServerDataApi(connectionString)) { } \[/csharp\]

Wow, hardcoded to `SqlServerDataApi`! Great job, Sitecore. So we are stuck with the default `SqlServerDataApi`. Instead we must override the `Sitecore.Data.DataProviders.Sql.FastQuery.QueryToSqlTranslator` which parses a Fast Query to _steps/opcodes_ using `Sitecore.Data.Query.QueryParser` and translates these to SQL using `Sitecore.Data.DataProviders.Sql.FastQuery.IOpcodeTranslator` implementations. The previously mentioned class `FieldTranslator` is such an implementation. Below is a reimplementation of `FieldTranslator.RenderFieldByGuid` with the only change to use `KvIDFieldInfo` instance instead of `IDFieldInfo`. `KvIDFieldInfo` includes the fix to proper escape the field ID in our Fast Query.

\[csharp\] public class KvFieldTranslator : FieldTranslator { protected override string RenderFieldByGuid(Sitecore.Data.ID fieldID, ITranslationContext context) { context.Data\["complex-fields"\] = true; var info = context.Fields\[fieldID\] as IFieldInfo; if (info == null) { info = new KvIDFieldInfo(fieldID, context); context.Fields\[fieldID\] = info; }

return this.RenderField(info, context); } }

public class KvIDFieldInfo : IFieldInfo { public string Alias { get; private set; }

public KvIDFieldInfo(Sitecore.Data.ID fieldId, ITranslationContext context) { this.Alias = context.AddSubquery(context.SqlApi.Format("(SELECT {0}ItemId{1}, {0}Value{1} FROM {0}Fields{1} WHERE {0}FieldID{1}='") + context.SqlApi.Safe(fieldId.ToGuid().ToString()) + "')", "Fields"); } } \[/csharp\]

Now we need to use our custom `KvFieldTranslator` for query opcodes of type `Sitecore.Data.Query.FieldElement`.

\[csharp\] public class KvQueryToSqlTranslator : Sitecore.Data.DataProviders.Sql.FastQuery.QueryToSqlTranslator { public KvQueryToSqlTranslator(Sitecore.Data.DataProviders.Sql.SqlDataApi api) : base(api) { ((Sitecore.Data.DataProviders.Sql.FastQuery.BasicTranslatorFactory)this.\_factory).Register(typeof(FieldElement), new KvFieldTranslator()); } } \[/csharp\]

And lastly a custom `SqlServerDataProvider` which instantiates our `KvQueryToSqlTranslator`.

\[csharp\] public class KvSqlServerDataProvider : Sitecore.Data.SqlServer.SqlServerDataProvider { public KvSqlServerDataProvider(string connectionString) : base(connectionString) { }

protected override Sitecore.Data.DataProviders.Sql.FastQuery.QueryToSqlTranslator CreateSqlTranslator() { return new KvQueryToSqlTranslator(this.Api); } } \[/csharp\]

Instruct Sitecore to use our data provider:

\[xml\] <dataProviders> <main type="Kraftvaerk.KvSqlServerDataProvider, Kraftvaerk"> <param connectionStringName="$(1)" /> <Name>$(1)</Name> </main> </dataProviders> \[/xml\]

Now our Sitecore Fast Query with field ID works as expected, and generates the following SQL query:

\[sql\] exec sp\_executesql N'SELECT DISTINCT \[i\].\[ID\] \[ID\], \[i\].\[ParentID\] \[ParentID\] FROM \[Items\] \[i\] LEFT OUTER JOIN ( SELECT \[ItemId\], \[Value\] FROM \[Fields\] WHERE \[FieldID\]=''67e0f992-ae49-4113-8408-e355d5be6b97'' ) \[Fields1\] ON \[i\].\[ID\] = \[Fields1\].\[ItemId\] WHERE ((\[i\].\[TemplateID\] = @value1) and (coalesce(\[Fields1\].\[Value\], '''') LIKE @value2))',N'@value1 nvarchar(38),@value2 nvarchar(5)',@value1=N'{C29492D2-6F79-463C-A7FB-B318FAB90639}',@value2=N'43870' \[/sql\]

So our query now only contains a subquery on _Fields_ view for a fixed field id, which should be faster than the previous query. But there is plenty of room for further improvement. In future post I'll examine the execution plans for the two queries, and show how to optimize them with a few indexes in the Sitecore SQL database.

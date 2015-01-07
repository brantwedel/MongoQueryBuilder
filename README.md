# MongoQueryBuilder
First working draft.

You must define conventions. The test currently defines two conventions:
1) A simple property equality convention `ByPropertyName`
2) A more complex Array property 'contains' check `PropertyNameContains`

It does type checking at runtime bootstrap time, to make sure you aren't defining a `ByName(int)` when the value of `Name` is actually `string`

To define a convention, too many things must happen...

Given the interface
```
    public interface IQueryBuilderMethodConvention
    {
        bool Matches(Type type, MethodInfo method);
        IMongoQuery GenerateQueryComponent(IInvocation invocation);
        UpdateBuilder GenerateUpdateComponent(IInvocation invocation);
    }
```

You must implement it with a Convention, say... you want to match equality with a `ByName` convention.

```
public class ByConvention : IQueryBuilderMethodConvention
{
    public Func<Type, MethodInfo, bool>[] Criteria =
        {
            (t,m) => m.Name.StartsWith("By"),
            (t,m) => m.GetParameters().Length == 1,
            (t,m) => t.GetProperties()
                .Any(p => p.Name == ExtractPropertyName(m.Name)),
            (t,m) => t.GetProperties()
                .First(p => p.Name == ExtractPropertyName(m.Name))
                .PropertyType == m.GetParameters().First().ParameterType
            };

    public bool Matches(Type type, MethodInfo method)
    {
        return this.Criteria.All(i => i(type, method));
    }

    public IMongoQuery GenerateQueryComponent(IInvocation invocation)
    {
        return Query.EQ(
            ByConvention.ExtractPropertyName(invocation.Method.Name),
            BsonValue.Create(invocation.Arguments[0]));
    }
    public UpdateBuilder GenerateUpdateComponent(IInvocation invocation)
    {
        return null;
    }
    public static string ExtractPropertyName(string methodName)
    {
        return Regex.Replace(methodName, "^By", "");
    }
}
```
    
But after that ugliness, you can simply define your QueryBuilder convention-based interface

```
public class Company
{
    public int Id {get;set;}
    public string Name {get;set;}
}
public interface ICompanyQueryBuilder : IQueryBuilder<Company>
{
    ICompanyQueryBuilder ByName(string name);
}
```

And then execute this test

```
var provider = new StandardRepositoryProvider();
var repo = provider.CreateRepository<Company, ICompanyQueryBuilder>(
    new RepositoryConfiguration
    {
        CollectionName = "companies",
        DatabaseName = "testdata",
        ConnectionString = "mongodb://localhost",
        SafeModeSetting = SafeMode.True
    // another option is Assembly.GetExecutingAssembly() to load all the relevant types
    }, typeof(ByConvention), typeof(ICompanyQueryBuidler));
    
    
repo.Collection.Drop();
repo.Save(new Company
{
    Id = 1,
    Name = "Test One"
});
repo.Save(new Company
{
    Id = 2,
    Name = "Test Two",
});

Assert.AreEqual(1, repo.Query()
    .ByName("Test One")
    .GetAll()
    .Count);
Assert.AreEqual(2, repo.Query()
    .GetAll(true) // the true allows you to execute empty queries for GetAll -- if it's false(default), it won't let you.
    .Count);
```

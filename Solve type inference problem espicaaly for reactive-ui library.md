### The Problem

```C#

class BaseViewModel<TModel>
{
    public TModel Model { get; set; }
}

class User
{
    public string Name { get; set; }
    public byte Age { get; set; }
}

class MainViewModel : BaseViewModel<User>
{
    public string Id { get; set; }
}

static class BaseViewModelExtensions
{
    public static bool Check<TB, TM>(this TB bVm, Func<TM, bool> check)
    where TB: BaseViewModel<TM>
    {
        return check(bVm.Model);
    }

    public static void Check2<TB, TM>(this TB bVm)
        where TB: BaseViewModel<TM>
    {
    }
}

static class BaseViewModelEnhancedExtensions<TB, TM>
    where TB: BaseViewModel<TM>
{
    public static void Check(TB bVm)
    {
    }
}

static class Test
{
    public static void Run()
    {
        var mvm = new MainViewModel {Model = {Name = "O", Age = 30}, Id = "RIH"};
        mvm.Check<MainViewModel, User>(m => m.Age > 10);
        BaseViewModelEnhancedExtensions<MainViewModel, User>.Check(mvm);

        // Due to limitation of C# generic inferred as described in its Specifications: `7.5.2.1 - 7.5.2.6`

        // we can not do next
        // mvm.Check2();
        // BaseViewModelEnhancedExtensions.Check(mvm);
    }
}

```

#### First Solution: we will introduce Self Property to enhance types inferring

```C#

class BaseViewModel<TModel, TThis>
where TThis: BaseViewModel<TModel, TThis>
{
    public TModel Model { get; set; }

    public (TThis vm, TModel m) Self => ((TThis) this, Model);
}

class User
{
    public string Name { get; set; }
    public byte Age { get; set; }
}

class MainViewModel : BaseViewModel<User, MainViewModel>
{
    public string Id { get; set; }
}

static class BaseViewModelExtensions
{
    public static bool DoWork<TB, TM>(this TB vm, Func<TM, bool> exe)
    where TB: BaseViewModel<TM, TB>
    {
        return exe(vm.Model);
    }

    public static bool Work<TB, TM>(this (TB vm, TM m) self, Func<TM, bool> exe)
        where TB: BaseViewModel<TM, TB>
    {
        return exe(self.vm.Model);
    }
}


static class Test
{
    public static void Run()
    {
        var mvm = new MainViewModel {Model = {Name = "O", Age = 30}, Id = "RIH"};

        mvm.Self.Work(s => string.IsNullOrEmpty(s.Name));

    }
}


```

#### Second Solution: We will introduce ISelf interface

```C#

// TSub should increases as many as sub types may needed [TSub1, TSub2]
interface ISelf<TSub, TThis> where TThis: ISelf<TSub, TThis>
{
    (TThis th, TSub sub) Self { get; }
}

interface IBaseViewModel<TModel, TSelf> : ISelf<TModel, TSelf> where TSelf : ISelf<TModel, TSelf>
{
    TModel Model { get; set; }
}

// helper
class User
{
    public string Name { get; set; }
    public byte Age { get; set; }
}

#region Second Solution - method 1

class BaseViewModel<TModel> : IBaseViewModel<TModel, BaseViewModel<TModel>>
{
    public TModel Model { get; set; }

    public (BaseViewModel<TModel> th, TModel sub) Self => (this, Model);
}

class MainViewModel : BaseViewModel<User>
{
    public string Id { get; set; }
}

#endregion


#region Second Solution - method 2

class BaseViewModel2<TModel, TThis> : IBaseViewModel<TModel, TThis> where TThis: BaseViewModel2<TModel, TThis>
{
    public TModel Model { get; set; }

    public (TThis th, TModel sub) Self => ((TThis) this, Model);
}


class MainViewModel2 : BaseViewModel2<User, MainViewModel2>
{
    public string Id { get; set; }
}

#endregion


static class BaseViewModelExtensions
{
    #region Use Hard Constraints

    public static bool Work<TB, TM>(this (TB vm, TM m) self, Func<TM, bool> exe)
        where TB: BaseViewModel2<TM, TB>
    {
        return exe(self.vm.Model);
    }

    public static bool Work<TM>(this (BaseViewModel<TM> vm, TM m) self, Func<TM, bool> exe)
    {
        return exe(self.vm.Model);
    }

    #endregion

    #region Use Mixed Constraints: ISelf + Hard Types

    public static bool Fun<TM>(this ISelf<TM, BaseViewModel<TM>>  b, Func<TM, bool> exe)
    {
        return exe(b.Self.sub);
    }

    public static bool Fun<TB, TM>(this ISelf<TM, TB> b, Func<TM, bool> exe)
        where TB : BaseViewModel2<TM, TB>
    {
        return exe(b.Self.sub);
    }

    #endregion


    #region Hit the edge of genericity

    // the most generic one : will work for booth.
    public static bool MostFun<TB, TM>(this ISelf<TM, TB> b, Func<TM, bool> exe)
        where TB : ISelf<TM, TB>
    {
        return exe(b.Self.sub);
    }

    #endregion

}

static class Test
{
    public static void Run()
    {
        var mvm = new MainViewModel {Model = {Name = "O", Age = 30}, Id = "RIH"};
        var mvm2 = new MainViewModel2 {Model = {Name = "O", Age = 30}, Id = "RIH"};

        #region HC

        mvm.Self.Work(s => string.IsNullOrEmpty(s.Name));
        mvm2.Self.Work(s => string.IsNullOrEmpty(s.Name));

        #endregion

        #region MC

        mvm.Fun(s => string.IsNullOrEmpty(s.Name));
        mvm2.Fun(s => string.IsNullOrEmpty(s.Name));

        #endregion

        #region MG

        mvm.MostFun(s => string.IsNullOrEmpty(s.Name));
        mvm2.MostFun(s => string.IsNullOrEmpty(s.Name));

        #endregion

    }
}

```

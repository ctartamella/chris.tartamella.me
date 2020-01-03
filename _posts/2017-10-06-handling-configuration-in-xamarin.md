---
layout:     post
title:      Handling Configuration in Xamarin
date:       2017-08-06
description:    "Leverage the power of .NET Core's configuration library"
tags:       dotnet dotnetcore xamarin mobile
---

One of the great things about .NET Core is it's high level of customization.  You can opt-in to and configure almost anything an app would need without inheriting any of the cruft you don't.  To achieve this, the team implemented a very high level of modularity.  Things that would have shipped with the framework before were broken out into separate NuGet packages.  One of my favorites is the [Microsoft.Extensions.Configuration](https://github.com/aspnet/Configuration) package.  Consumers can leverage this into a highly customized configuration for their application which can draw on multiple sources in such a way that makes app.config files and transformations weak in the knees.  In this article, I'm going to show you how to leverage this in your Xamarin mobile apps to create application configs that can meet any need.

# The New Standard

The first thing you need to do is get rid of those PCLs.  This has been covered in many other places and is beyond the scope of this article, but I will include a sample project here for readers to extrapolate from.  There are a few important parts to point out.  First, we target .NET Standard 2.0 here.  This is supported as of Xamarin Forms 2.4 which was recently released.  In this scenario, we also want to define the AssetTargetFallback.  In previous versions of the standard, this was called the TargetPlatformFallback but the purpose is more or less the same.  By setting this, we are saying that we accept PCL packages which support the defined platforms as dependencies when pulling from NuGet.  Not all packages have migrated but most are still compatible with the standard.

The next thing you will notice is are NuGet references defined inline.  We no longer need a packages.config file to list out our dependent packages.  Sometimes you will see `NoWarn="NU1701"` to suppress a NuGet error warning when a package is pulled due to the AssetTargetFallback setting above.

The final part of this project extends the default MSBuild rules for XAML files.  We update the rules for .xaml.cs files to add the `DependentUpon` attribute that you would see in old PCL project files.  This creates the hierarchy you are used to seeing in the IDE for code-behind files.  Second, we add a rule that globs all XAML files and defines them as EmbeddedResources.  This isn't really different in practice than old style .csproj files, however it is much more succinct and easy to maintain.  It also allows us to leverage the configuration packages you see references to.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <AssetTargetFallback>$(AssetTargetFallback);portable-net45+win8+wp8+wpa81</AssetTargetFallback>
    <RootNamespace>ConfigurationSample.Core</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Xamarin.Forms" Version="2.4.0.283" />
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="2.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="2.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="2.0.0" />
    ...
  </ItemGroup>

  <ItemGroup>
    <Compile Update="**\*.xaml.cs" DependentUpon="%(Filename)" />
    <EmbeddedResource Include="**\*.xaml" SubType="Designer" Generator="MSBuild:UpdateDesignTimeXaml" />
  </ItemGroup>
</Project>
```

# What's the Configuration, Kenneth

So, now that we are using .NET Standard, how do we implement configuration in our Xamarin app?  First, let's look and see what happens in a typical .NET Core web application.  From Microsoft's own [documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration?tabs=basicconfiguration), we see an app who loads `appsettings.json` from the root directory of the app and builds a configuration out of it.  This Configuration object could then be saved in your IoC container, set as a static property on some root object, or thrown away after using it.  Further, this example shows how you can leverage nesting in your JSON file.

```csharp
using System;
using System.IO;
using Microsoft.Extensions.Configuration;

public class Program
{
    public static IConfigurationRoot Configuration { get; set; }

    public static void Main(string[] args = null)
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json");

        Configuration = builder.Build();

        Console.WriteLine($"option1 = {Configuration["option1"]}");
        Console.WriteLine($"option2 = {Configuration["option2"]}");
        Console.WriteLine(
            $"suboption1 = {Configuration["subsection:suboption1"]}");
        Console.WriteLine();

        Console.WriteLine("Wizards:");
        Console.Write($"{Configuration["wizards:0:Name"]}, ");
        Console.WriteLine($"age {Configuration["wizards:0:Age"]}");
        Console.Write($"{Configuration["wizards:1:Name"]}, ");
        Console.WriteLine($"age {Configuration["wizards:1:Age"]}");
        Console.WriteLine();

        Console.WriteLine("Press a key...");
        Console.ReadKey();
    }
}
```

```json
{
  "option1": "value1_from_json",
  "option2": 2,

  "subsection": {
    "suboption1": "subvalue1_from_json"
  },
  "wizards": [
    {
      "Name": "Gandalf",
      "Age": "1000"
    },
    {
      "Name": "Harry",
      "Age": "17"
    }
  ]
}
```

While simple, this example exposes a major problem when porting to Xamarin.  We don't have a root directory from which to load a JSON file.  If we assume this is the easiest way to store the information we need (and it probably is), we need to be able to load files from somewhere else.  If we explore the API, we find that there is an override `AddJsonFile(IFileProvider, string, bool, bool)`.  The key here is the `IFileProvider` interface.  For reference, the interface is included here.  This interface gets implemented by any system that can provide configuration information.  For JSON files coming from disks, this interface will read the file system and return data appropriately.

```csharp
/// <summary>A read-only file provider abstraction.</summary>
public interface IFileProvider
{
    /// <summary>Locate a file at the given path.</summary>
    /// <param name="subpath">Relative path that identifies the file.</param>
    /// <returns>The file information. Caller must check Exists property.</returns>
    IFileInfo GetFileInfo(string subpath);

    /// <summary>Enumerate a directory at the given path, if any.</summary>
    /// <param name="subpath">Relative path that identifies the directory.</param>
    /// <returns>Returns the contents of the directory.</returns>
    IDirectoryContents GetDirectoryContents(string subpath);

    /// <summary>
    /// Creates a <see cref="T:Microsoft.Extensions.Primitives.IChangeToken" /> for the specified <paramref name="filter" />.
    /// </summary>
    /// <param name="filter">Filter string used to determine what files or folders to monitor. Example: **/*.cs, *.*, subFolder/**/*.cshtml.</param>
    /// <returns>An <see cref="T:Microsoft.Extensions.Primitives.IChangeToken" /> that is notified when a file matching <paramref name="filter" /> is added, modified or deleted.</returns>
    IChangeToken Watch(string filter);
}
```

The trick getting configuration on Xamarin is implementing this in such a way that mobile apps can access configuration files.  The first thing that comes to mind is bundling config files with the embedded resources (just like compiled XAML files are) and reading them from there using reflection at runtime.  The implmentation will be listed here.

This is the top level interface implmentation.  It exists only to create instances of other interfaces which we also define.

```csharp
public class ResourceFileProvider : IFileProvider
{
    public IFileInfo GetFileInfo(string subpath)
    {
        return new ResourceFileInfo(subpath);
    }

    public IDirectoryContents GetDirectoryContents(string subpath)
    {
        return new ResourceDirectoryContents();
    }

    public IChangeToken Watch(string filter)
    {
        return new ResourceChangeToken();
    }

}
```

This is our implementation of `IDirectoryContents`.  We don't implement an enumerator simply because we don't need one.  This is really just a no-op implmentation.

```csharp
public class ResourceDirectoryContents : IDirectoryContents
{
    public IEnumerator<IFileInfo> GetEnumerator()
    {
        throw new NotImplementedException();
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }

    public bool Exists { get; }
}
```

The crux of the implementation lies here.  We make the assumption that configuration files are in the root folder for simplicity.  When creating the stream, we use reflection to return the resource stream.  All other properties are defined relative to this.

```csharp
public class ResourceFileInfo : IFileInfo
{
    public ResourceFileInfo(string path)
    {
        PhysicalPath = path;
        Name = Path.GetFileName(path);
    }

    public Stream CreateReadStream()
    {
        var assembly = typeof(ResourceFileInfo).GetTypeInfo().Assembly;
        return assembly.GetManifestResourceStream(PhysicalPath);
    }

    public bool Exists
    {
        get
        {
            var assembly = typeof(ResourceFileInfo).GetTypeInfo().Assembly;
            return assembly.GetManifestResourceNames().Contains(Name);
        }
    }

    public long Length => CreateReadStream().Length;
    public string PhysicalPath { get; }
    public string Name { get; }
    public DateTimeOffset LastModified => DateTimeOffset.Now;
    public bool IsDirectory => false;
}
```

As these files are resources, changes are impossible so we essentially disable any change tokens.

```csharp
public class ResourceChangeToken : IChangeToken
{
    public IDisposable RegisterChangeCallback(Action<object> callback, object state) => Disposable.Empty;
    public bool HasChanged => false;
    public bool ActiveChangeCallbacks => false;
}
```

Using this implmentation, configuring your Configuration system is now easy.

# Put It All Together

First, we need to embed the configuration.  Go back to your shared project configuration and make the appropriate changes.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <AssetTargetFallback>$(AssetTargetFallback);portable-net45+win8+wp8+wpa81</AssetTargetFallback>
    <RootNamespace>ConfigurationSample.Core</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Xamarin.Forms" Version="2.4.0.283" />
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="2.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="2.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="2.0.0" />
    ...
  </ItemGroup>

  <ItemGroup>
    <Compile Update="**\*.xaml.cs" DependentUpon="%(Filename)" />
    <EmbeddedResource Include="**\*.xaml" SubType="Designer" Generator="MSBuild:UpdateDesignTimeXaml" />
  </ItemGroup>

  <ItemGroup>
    <None Remove="Config\*.json" />
    <EmbeddedResource Include="Config\config.debug.json" Condition="'$(Configuration)'=='Debug'" LogicalName="config.json" />
    <EmbeddedResource Include="Config\config.release.json" Condition="'$(Configuration)'!='Debug'" LogicalName="config.json" />
    <EmbeddedResource Include="Config\platform.android.json" Condition="'$(Platform)'=='Android'" LogicalName="platform.json" />
    <EmbeddedResource Include="Config\platform.ios.json" Condition="'$(Platform)'!='Android'" LogicalName="platform.json" />
  </ItemGroup>
</Project>
```

Now, what the heck have I done here?  For my particular example, I wanted to have separate Debug and Release configuations so that I could define different backends.  However, I also have things like HockeyApp IDs that are defined per platform.  So, I have devised a system with four separate configuration files.  In the project file, I use build variables to map either `config.debug.json` or `config.release.json` to a single resource file named `config.json`.  This will be included as an embedded resource file.  Similarly, I have `platform.android.json` and `platform.ios.json` which get mapped to `platform.json` in the resource manifest.

Your situation here can always vary.  That is really what is great here.  We can create a configuration layout that meets our needs exactly.  In an upcoming state we will even define the load order so if two files share a setting, the one that is loaded last will take priority.

Speaking of loading these files, remember that .NET Core example I showed earlier where we loaded appsettings.json?  Well now we can do something very similar.  You can do this in many places, but I do it in my platform dependent startup code.  For iOS, I have the following.

```csharp
[Register("AppDelegate")]
internal class AppDelegate : FormsApplicationDelegate
{
    public override bool FinishedLaunching(UIApplication uiApplication, NSDictionary launchOptions)
    {
        var config = new ConfigurationBuilder()
            .AddJsonFile(new ResourceFileProvider(), "config.json", false, false)
            .AddJsonFile(new ResourceFileProvider(), "platform.json", false, false)
            .Build();

        // Register this with your IoC container

        Forms.Init();

        LoadApplication(new App());
        return base.FinishedLaunching(uiApplication, launchOptions);
    }
}
```

Notice how we call `AddJsonFile` with our newly created `ResourceFileProvider` implementation and pass the logical name of the config file as we added it to the project.  That's it!  The `config` object is now a key-value store with all of your parameters loaded.  You can query child objects and return other configuration values with their parameters just like you would in your .NET Core app.

# Safety First

Let's go a step further.  No one wants to remember key names when querying these values.  Let's add some type safety.  We do this by creating a POCO (plain old C object) that describes our configuration model and adding ONE line to the startup configuration.  Let's do it.  First, the model object.

```csharp
public class Configuration
{
    public string HockeyAppKey { get; set; }
    public ApiConfiguration Api { get; set; }
}

public class ApiConfiguration
{
    public string Root { get; set; }
    public string Client { get; set; }
    public string Secret { get; set; }
    public string GrantType { get; set; }
}
```

config.debug.json

```json
{
    "Api": {
        "Root": "https://yourapiroot.com",
        "Client": "*******",
        "Secret": "******",
        "GrantType": "password"
    }
}
```

platform.ios.json

```json
{
    "HockeyAppKey": "*******"
}
```

Note that I mirror the structure of the configuration files here as well.  For reference, I also listed debug and iOS configuration files.  Note how these are melded together to form a single configuration model.  Now, we perform the mapping.  In the startup (AppDelegate for instance on iOS):

```csharp
[Register("AppDelegate")]
internal class AppDelegate : FormsApplicationDelegate
{
    public override bool FinishedLaunching(UIApplication uiApplication, NSDictionary launchOptions)
    {
        var config = new ConfigurationBuilder()
            .AddJsonFile(new ResourceFileProvider(), "config.json", false, false)
            .AddJsonFile(new ResourceFileProvider(), "platform.json", false, false)
            .Build()
            .Get<Configuration>();

        // Register this with your IoC container

        Forms.Init();

        LoadApplication(new App());
        return base.FinishedLaunching(uiApplication, launchOptions);
    }
}
```

Note the single addition of the call to `Get<TConfigModel>()`.  Now, rather than a key-value dictionary, we get an instance of the model object we defined.

# Wrap Up

Hopefully you see the usefulness of this beyond a simple app.config.  This seems (to me at least) much more maintainable when compared to app.config transforms and much easier to use once its setup.  I'd love to hear what you think in the comments.  Next up, I may wrap this implemenation up into a more usable NuGet package for people to consume if there is enough demand.
---
layout:     post
title:      Implementing Toggle Buttons in Xamarin Forms
date:       2018-01-03
description:    "Discover a simple way to implement radio button-like toggles in Xamarin forms in a cross-platform manner."
tags:       forms xamarinforms xamarin mobile controls
---

# Introduction

One control that Xamarin has seemingly left out of its library of controls is a toggle button, sometimes known as a radio button.  I recently ran into a use case for one so I worked out an implementation that I think is both clean and reusable.  I'm going to demonstrate it here and show you how you too can implement this really useful control.

# Design Requirements

First lets talk about my needs for this particular implementation as it may help you to understand some of the decisions that were made in coming up with this design.  In my particular application, I have a main view that has two available views (grid and list) and I wanted to toggle between those views.  Because each view mode corresponded to a different layout object type, it made sense to do this via `enum`.

> If you are interested in the list control being used, check
> out [Syncfusion's](https://www.syncfusion.com/) awesome control
> library for Xamarin forms.  It provides may of the features that
> Telerik provides but also has a free option for independent
> developers and > small businesses.

```csharp
public enum ViewType
{
    Grid,
    List
}
```

# Implementation

## Toggle Button

Because this toggle button set was going to appear in something akin to a toolbar, I chose to implement this as an image only button.  Further, I wanted fine tune control of margins around my images and toggle a background when the button was selected.  To facilitate this, I created a custom `ToggleButton` control.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ContentView xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="Core.Controls.ToggleButton">
    <ContentView.GestureRecognizers>
        <TapGestureRecognizer x:Name="TapGesture" />
    </ContentView.GestureRecognizers>

    <Grid>
        <Image Margin="2,2,2,2" x:Name="ImageView" />
    </Grid>
</ContentView>
```

The interface for this is very simple.  Note the tap gesture recognizer used to capture "clicks".  Most of the implementation is contained in code-behind.

```csharp
public partial class ToggleButton
{
    public static readonly BindableProperty ImageProperty = BindableProperty.Create(
        nameof(Image), typeof(ImageSource), typeof(ToggleButton), propertyChanged: OnImageChanged);

    public static readonly BindableProperty CommandProperty = BindableProperty.Create(
        nameof(Command), typeof(ICommand), typeof(ToggleButton), propertyChanged: OnImageChanged);

    public static readonly BindableProperty CommandParameterProperty = BindableProperty.Create(
        nameof(CommandParameter), typeof(object), typeof(ToggleButton), propertyChanged: OnImageChanged);

    private static void OnImageChanged(BindableObject bindable, object oldValue, object newValue)
    {
        if (!(bindable is ToggleButton control))
            return;

        control.ImageView.Source = control.Image;
    }

    public ToggleButton ()
    {
        InitializeComponent ();
        TapGesture.Command = TapCommand;
    }

    public event EventHandler<object> Clicked;

    public ImageSource Image
    {
        get => (ImageSource) GetValue(ImageProperty);
        set => SetValue(ImageProperty, value);
    }

    public ICommand Command
    {
        get => (ICommand )GetValue(CommandProperty);
        set => SetValue(CommandProperty, value);
    }

    public object CommandParameter
    {
        get => GetValue(CommandParameterProperty);
        set => SetValue(CommandParameterProperty, value);
    }

    private ICommand TapCommand => new Command(() =>
    {
        Clicked?.Invoke(this, CommandParameter);
        Command?.Execute(CommandParameter);
    });
}
```

Here, I have implemented 4 very familiar properties.  Note the private command `TapCommand`.  This is assigned to the tap gesture recognizer in code-behind so that we can both fire the `Clicked` event _and_ call the bound command.  This will come in handy later.  Further, we have bindable properties for the image to display and the parameter to pass to the command.  When the user taps the button, it both fires the event and executes the command, passing `CommandParameter` to both.

In hindsight, after a few revisions, this control is less of a toggle button and more of just a regular button that exposes propeties that are useful for this case.  You could easily use a similar approach with some other control being toggled.

## Toolbar Implementation

This is where it starts to get a little interesting.  The tool bar is the control which contains the set of `ToggleButton`.  The XAML code starts off in a very uninteresting fashion.  Notice we have bound shared commands to each of the buttons and instead opted to pass our enum type as a parameter.  Each button also gets its own image set.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ContentView xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:controls="clr-namespace:Core.Controls"
             xmlns:pageModels="clr-namespace:Core.PageModels"
             x:Class="Core.Controls.ToolbarControl">
    <StackLayout Orientation="Horizontal">
        ...
        <controls:ToggleButton x:Name="ListButton" Image="ListView"
                               Command="{Binding ChangeViewCommand}"
                               CommandParameter="{x:Static pageModels:ViewType.List}" />
        <controls:ToggleButton x:Name="GridButton" Image="GridView"
                            Command="{Binding ChangeViewCommand}"
                            CommandParameter="{x:Static pageModels:ViewType.Grid}" />
        ...
    </StackLayout>
</ContentView>
```

> As an aside, note the syntax used to reference the `enum` value.  We are able to
> treat it as if it were another static reference and easily pass it as the parameter.
> This is a little XAML nugget that I think some people miss.

The code-behind looks like this.  Because this control is split off from its container page, we expose a few bindable properties.  First, we define the current view type using our `enum`.  Note that the binding on this is two way.  This is critical as this needs to be changeable in both our consuming view model and within the control itself.  We also provide a command to change the view (which is passed on to the buttons themselves) and our implementation exposes a property to set the selected background color.

> Aside #2.  Please note the `OnParentSet` override.  I see people forget this all the time
> and it can be very important.  Note that when the parent is not `null` we know the control has just been
> attached.  This is when we take the time to subscribe to necessary events.  However, many developers
> overlook **unsubscribing** from those events as well.  Failure to do so will leave dangling references and
> cause the object to not be garbage collected, thus leaking memory.  In a `Page` you can do something
> similar with the `OnAppearing` and `OnDisappearing` overrides.  **Always** ensure you have a matching
> unsubscribe for each event subscription.  This has been a public service announcement from your friendly
> neighborhood unmanaged developer.

```csharp
public partial class ToolbarControl
{
    public static BindableProperty CurrentViewTypeProperty = BindableProperty.Create(
        nameof(CurrentViewType), typeof(ViewType), typeof(ToolbarControl), ViewType.List, BindingMode.TwoWay, propertyChanged: OnUpdateNeeded);

    public static BindableProperty SelectedColorProperty = BindableProperty.Create(
        nameof(SelectedColor), typeof(Color), typeof(ToolbarControl), Color.White, propertyChanged: OnUpdateNeeded);

    private static void OnUpdateNeeded(BindableObject bindable, object oldValue, object newValue)
    {
        if (!(bindable is ToolbarControl control))
            return;

        if (control.CurrentViewType == ViewType.Grid)
        {
            control.GridButton.BackgroundColor = control.SelectedColor;
            control.ListButton.BackgroundColor = Color.Transparent;
        }
        else
        {
            control.ListButton.BackgroundColor = control.SelectedColor;
            control.GridButton.BackgroundColor = Color.Transparent;
        }
    }

    public ToolbarControl ()
    {
        InitializeComponent();
    }

    protected override void OnParentSet()
    {
        base.OnParentSet();
        if (null == Parent)
        {
            GridButton.Clicked -= OnClicked;
            ListButton.Clicked -= OnClicked;
        }
        else
        {
            GridButton.Clicked += OnClicked;
            ListButton.Clicked += OnClicked;
        }
    }

    public ViewType CurrentViewType
    {
        get => (ViewType) GetValue(CurrentViewTypeProperty);
        set => SetValue(CurrentViewTypeProperty, value);
    }

    public Color SelectedColor
    {
        get => (Color)GetValue(SelectedColorProperty);
        set => SetValue(SelectedColorProperty, value);
    }

    private void OnClicked(object sender, object args)
    {
        CurrentViewType = args is ViewType viewType ? viewType : ViewType.Grid;
    }
}
```

Other features of note here is the click handler itself which merely sets the current view type.  Because this is a bindable property, the property changed handler is also called.  We see that that simply checks the `CurrentViewType` property and sets the control backgrounds accordingly.

## Scalability

Granted, this is a very simple implementation.  I originally wrote the more complex version but scaled it back when I realized it was overkill for me.  In that case, the `ToggleButton` control had a static field of type `IDictionary<string, IDictionary<string, ToggleButton>>` which contained state data.  It also had a `string` property for the toggle set name and a control identifier.  This allowed for multiple toggle sets with varying numbers of controls.  As the buttons were initialized, they populated the state dictionary (and cleaned up afterwards).  When a button was hit, it was able to look up all other buttons in its set and modify their properties appropriately.

If there is interest in this implementation, I will gladly write about it in the future.

# Putting It All Together

Now all that remains is using it.  The XAML is trivial.

```xml
<controls:ToolbarControl SelectedColor="{StaticResource LightGreyColor}"
                         CurrentViewType="{Binding CurrentViewType}"/>
```

Furthermore, the code-behind is fairly common, and would likely be considered implementation specific.

```csharp
public abstract class ExplorerPageModel : PageModelBase
{
    private ViewType _currentViewType;
    private LayoutBase _currentLayout;
    ...
    public ViewType CurrentViewType
    {
        get => _currentViewType;
        set
        {
            SetProperty(ref _currentViewType, value);
            SetLayoutForView(value);
        }
    }

    public LayoutBase CurrentLayout
    {
        get => _currentLayout;
        private set => SetProperty(ref _currentLayout, value);
    }

    private void SetLayoutForView(ViewType viewType)
    {
        CurrentLayout = new ExplorerLayoutConverter()
            .Convert(viewType, typeof(object), null, CultureInfo.InvariantCulture) as LayoutBase;
    }
    ...
}
```

As you can see, this is fairly implementation specific.  We handle the view type changing in its setter, updating the `LayoutBase` instance that is bound to our list view.  In theory, many things could be done here.  All that is needed is to react to the toggle value changing.  All visual changes are handled at the control level here.  Finally, for completeness, I include the converter used here.  This should not be that surprising to anyone.

```csharp
public class ExplorerLayoutConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (!(value is ViewType type))
            throw new ArgumentOutOfRangeException();

        switch (type)
        {
            case ViewType.Grid:
                return new GridLayout { SpanCount = 3 };
            case ViewType.List:
                return new LinearLayout();
            default:
                throw new ArgumentOutOfRangeException();
        }
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```

# Bonus 1: Checkbox Implementation

I mentioned that many of the choices here were made based on my actual design specs.  As a small bonus, let's talk about how you could use this to implement a check box.  At it's core, a checkbox is simple a radio button with a single option that can be turned on or off.  So, rather than a `ViewType` you would only track a single `bool`.  Also, on click, rather than changing a background you would simply change the image displayed on the button itself.  Your resources could contain a checked and unchecked state.

This might not be the best implementation, but it might be useful in the case you are already implementing a toggle button set as well.

# Bonus 2: Push Button Animation

As a final bonus, I mentioned that I wanted to simulate the push of a tool bar button.  I have implemented this using Behaviors and Xamarin Forms' built in animation methods.

Behaviors are a newer feature, and as a former C++ developer who still misses multiple inheritance sometimes, behaviors are awesome because they allow you to implement "composition" of features in your controls without going to the trouble of a renderer.  Take push button animation as an example.

```csharp
public class AnimatedButtonBehavior : BindableBehavior<ToggleButton>
{
    protected override void OnAttachedTo(ToggleButton bindable)
    {
        base.OnAttachedTo(bindable);
        bindable.Clicked += OnClicked;
    }

    protected override void OnDetachingFrom(ToggleButton bindable)
    {
        base.OnDetachingFrom(bindable);
        bindable.Clicked -= OnClicked;
    }

    private async void OnClicked(object sender, object eventArgs)
    {
        if (AssociatedObject == null)
            return;

        AssociatedObject.AnchorX = 0.48;
        AssociatedObject.AnchorY = 0.48;
        await AssociatedObject.ScaleTo(0.8, 50, Easing.Linear);
        await Task.Delay(100);
        await AssociatedObject.ScaleTo(1, 50, Easing.Linear);
    }
}
```

This makes use of a class that, for me, is pretty much an auto-include in any Xamarin Forms project, the `BindableBehavior`.  I don't remember where I got my implementation from, so for academic reasons I will point you to the version [here](https://gist.github.com/asimmon/2c7c70ba94505c8231e9).  This class is like any MVVM class implementation.  There are many out there and they all do more or less the same thing.

```csharp
public class BindableBehavior<T> : Behavior<T> where T : BindableObject
{
    public T AssociatedObject { get; private set; }

    protected override void OnAttachedTo(T visualElement)
    {
        base.OnAttachedTo(visualElement);

        AssociatedObject = visualElement;

        if (visualElement.BindingContext != null)
            BindingContext = visualElement.BindingContext;

        visualElement.BindingContextChanged += OnBindingContextChanged;
    }

    private void OnBindingContextChanged(object sender, EventArgs e)
    {
        OnBindingContextChanged();
    }

    protected override void OnDetachingFrom(T view)
    {
        view.BindingContextChanged -= OnBindingContextChanged;
    }

    protected override void OnBindingContextChanged()
    {
        base.OnBindingContextChanged();
        BindingContext = AssociatedObject.BindingContext;
    }
}
```

The idea here is to track changes in binding context and set it appropriately within the behavior.  For convenience, we also track the attached object in the `AssociatedObject` property.  In our behavior, we simply hook into the `Clicked` event of the button (which is why we added both events and commands to the `ToggleButton` to begin with) and fire an animation asynchronously when it fires.  This allows us to separate the animation behavior from the business logic in our view models.  Implementing this changes our `ToolbarControl` so that we set the `Behaviors` collection on our `ToggleButton`.

```xml
 <controls:ToggleButton x:Name="ListButton" Image="ListView"
                               Command="{Binding ChangeViewCommand}"
                               CommandParameter="{x:Static pageModels:ViewType.List}">
    <controls:ToggleButton.Behaviors>
        <behaviors:AnimatedButtonBehavior>
    </controls:ToggleButton.Behaviors>
</controls:ToggleButton>
```

Becuase the `Behaviors` property is a collection, you could list multiple behaviors in a list and achieve the "composition" of various features while still separating them in a logical fashion.  This is classic "favor composition over inheritance" mindset I was referring to.

# Wrap Up

That's all for now, folks.  Hopefully you found this implementation useful and maybe even picked up a trick or two.  Feel free to hit up the comment section below with questions.
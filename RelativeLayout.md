#Grokking Relative Layout#

For a Xamarin Forms app there are several layout containers to choose from. Most UIs should be achievable with a StackLayout or a GridLayout. There are a few exceptions when those two, I had one of those cases, so this post looks at using RelativeLayout. 

**The Problem: Why a RelativeLayout**
I had to show an triangle in the corner of the screen. See the image below.

A StackLayout is very difficult to get items to overlap, negative margin can be used, but only for a known distance (won't always work for screens with different sizes). Items can be overlapped with a GridLayout if they are placed in the same cell. This renders the items over the entire cell, and my requirement was only part of the cell, the triangle. Using xaml only, this is not possible, and could only be done in code with a translation. 

**What is a View**
A view, the 'things' that we're trying to layout on the screen requires a few key pieces of information, some attributes. 
- An origin to start drawing the item from. This is an X and Y value, with (0,0) at the top left hand corner (of the device or container it's being drawn in)
- A width 
- A height

**Setting the attributes of a child**
A RelativeLayout allows you to specify the the X,Y, Width and Height of a child. The values that are set can be pulled from any other child or parent. They do not, need to be the same attribute (X,Y,Width,Height). To set the attributes of the child is as follows:

<pre lang="xml">
    <BoxView  RelativeLayout.YConstraint= TODO
              RelativeLayout.XConstraint= TODO
              RelativeLayout.WidthConstraint= TODO
              RelativeLayout.WidthConstraint= TODO />
</pre>

The simplest value to apply to each of these is the same as parent. Setting all four values to be the same as parent would make the view fill the space the parent is given. This is as follows

<pre lang="xml">
    <BoxView RelativeLayout.YConstraint= "{ConstraintExpression Type=RelativeToParent, Property=Y}"
             RelativeLayout.XConstraint= "{ConstraintExpression Type=RelativeToParent, Property=X"
             RelativeLayout.HeightConstraint= "{ConstraintExpression Type=RelativeToParent, Property=Height}"
             RelativeLayout.WidthConstraint= "{ConstraintExpression Type=RelativeToParent, Property=Width,Factor=1,Constant=0}" />
</pre>

First the value must be declared as ConstraintExpression. The type of the relationship is then declared, in this case RelativeToParent, we'll look at another one is just a moment. The last line also shows two extra parameters that can be used (are generally needed to do anything useful with RelativeLayout). The constant attribute simply adds the value to the property (like an offset).  ie for HeightConstraint if constant=10 then the height would be the property + 10. The value can be negative. Factor is simply a multiplier to the Property attribute. eg. 

<pre lang="xml">
    <BoxView RelativeLayout.WidthConstraint= "{ConstraintExpression Type=RelativeToParent, Property=Width,Factor=0.5,Constant=0}"
             ... />
</pre>

If the width of the parent is 480 and the factor is 0.5 then the with of the item will be 240 meaning the view will be half the width of the parent. 

**The Relative part of a RelativeLayout**
Each of the four attributes can be set from any attribute of another view with the relative layout. 

<pre lang="xml">
    <BoxView RelativeLayout.YConstraint="{ConstraintExpression Type=RelativeToParent, Property=Width,Factor=0.5}"
             RelativeLayout.XConstraint="{ConstraintExpression Type=RelativeToParent, Property=X"
             RelativeLayout.HeightConstraint="{ConstraintExpression Type=RelativeToParent, Property=Height Factor=0.5}"
             RelativeLayout.WidthConstraint="{ConstraintExpression Type=RelativeToParent, Property=Width,Factor=1,Constant=0}/>
</pre>

So this would set the Y value to be half of the hight of the parent, with the X and width filling the whole screen this will fill the bottom half of its parent. If RelativeLayout is the root, then it will fill half of screen.

**What I used it for**
I need to show a triangle in the top left corner of a cell (the cell is part of a ListView), and I also wanted it to scale for screen sizes (the test was to rotate to landscape and still have everything layout correctly). This was achieved by nesting everything in a relative layout. The content was placed in a StackLayout. A BoxView was then used set relative to the origin, a fixed width and a negative constant so that it is always placed correctly. Because the width is always fixed size then the constant value can be known ensuring it will always be placed correctly. Here's the xaml to make it clear. 

<pre lang="xml">
    <ViewCell>
            <RelativeLayout HeightRequest="100" MinimumHeightRequest="100">
            <StackLayout HorizontalOptions="FillAndExpand" x:Name="anchorView">
                    <Label Text="Your code here"/>
            </StackLayout>
            <BoxView Color="#FFD180" Rotation="-45" Opacity=".90"
                        RelativeLayout.YConstraint="{ConstraintExpression Type=RelativeToParent, Property=Y,Factor=0.5}"
                        RelativeLayout.XConstraint="{ConstraintExpression Type=RelativeToView, ElementName=anchorView, 
                                                    Property=Width, Factor=0, Constant=-210}"
                        RelativeLayout.WidthConstraint="{ConstraintExpression Type=RelativeToParent,Property=Width,Factor=0,Constant=400}"
                        RelativeLayout.HeightConstraint="{ConstraintExpression Type=RelativeToParent,Property=Height,Factor=0.5}" />
            </RelativeLayout>
    </ViewCell>
</pre>

**What it can't do**
Halfway through this post I tried to create a great example using a RelativeLayout and xaml only to find that it can't be done. For those who have using RelativeLayout in android, Xamarin Forms' RelativeLayout is not as expressive. For a page to be effective, it's best to avoid fixed sizing, or it won't scale well on all those android devices (and and the growing number of devices for apple too). 

RelativeLayout can't be a StackLayout. In order to place one view below the other, you would would need to set the Y value of the second view equal to Y value of the top view + the height. This calculation can't be done in xaml. Because of this, it's difficult to layout a view that will fit all screen sizes. Using the Constant attribute doesn't work well across screen sizes.
It can't be a GridLayout as well for the same reason as the StackLayout, it's not possible in xaml to add values together. 

**Summary**
RelativeLayout is great for adding a few styles, but for most of your views stick to GridLayout and StackLayouts. 


# SwiftUI: Liquid Glass. From Basic to (A Little) Advance.

![](https://miro.medium.com/v2/resize:fill:64:64/1*QgtFRscRjbI92j5PPpsfEg.jpeg) 

[Itsuki](https://medium.com/@itsuki.enjoy?source=post%5Fpage---byline--cdef4e4c5b90---------------------------------------)

Follow

11 min read

·

Jun 15, 2025

32

[ ](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F%5F%2Fbookmark%2Fp%2Fcdef4e4c5b90&operation=register&redirect=https%3A%2F%2Flevelup.gitconnected.com%2Fswiftui-liquid-glass-from-basic-to-a-little-advance-cdef4e4c5b90&source=---header%5Factions--cdef4e4c5b90---------------------bookmark%5Ffooter------------------) 

Listen

Share

![](https://miro.medium.com/v2/resize:fit:590/1*VtSQhrHcGDJGlxKbgB0cFA.gif) 

**Fancy**[**Liquid Glass**](https://developer.apple.com/documentation/technologyoverviews/liquid-glass)**is coming!**

A new dynamic material that combines the optical properties of glass with a sense of fluidity!

Standard system components like`[NavigationStack](https://developer.apple.com/documentation/SwiftUI/NavigationStack)`,`[titleBar](https://developer.apple.com/documentation/SwiftUI/WindowStyle/titleBar)`and`[toolbar(content:)](https://developer.apple.com/documentation/SwiftUI/View/toolbar%28content:%29)`adopt to this new design automatically!

If you want to check out how those view are changed, this[**_Apple’s Adopting Liquid Glass_**](https://developer.apple.com/documentation/technologyoverviews/adopting-liquid-glass)is for you!

In this article, we will be focusing on apply the effect to our own custom views!

(I know Apple also has this[**_Applying Liquid Glass to custom views_**](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views), but trust me, there is more than that!)

So! Here is what we have!

* Of course, all the basics of`[glassEffect](https://developer.apple.com/documentation/swiftui/view/glasseffect%28%5F:in:isenabled:%29)`, shapes, colors, and whatever!
* Usage of`[GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer)`view
* Customize transition behavior with`[glassEffectTransition](https://developer.apple.com/documentation/swiftui/view/glasseffecttransition%28%5F:isenabled:%29)`
* Control animation grouping behavior`[glassEffectID](https://developer.apple.com/documentation/swiftui/view/glasseffectid%28%5F:in:%29)`
* Combine multiple liquid glass effects to a single one with`[glassEffectUnion](https://developer.apple.com/documentation/swiftui/view/glasseffectunion%28id:namespace:%29)`

**_And!_**

Some buggy behaviors I encountered, Some points to be careful about!

(Yes, beta is never perfect! That’s why it is fun!)

A little code snippet is provided at the end of this article, grab it and let’s start!

# Basic

Applying Liquid Glass to any custom view is as simple as attaching the`[glassEffect(_:in:isEnabled:)](https://developer.apple.com/documentation/swiftui/view/glasseffect%28%5F:in:isenabled:%29)`modifier.

let text = Text("˚ʚ♡ɞ˚")  
    .padding(.all, 8)  
  
text.glassEffect()

![](https://miro.medium.com/v2/resize:fit:336/1*xg4wqpXMS8fqmdx-WClmkA.png) 

This modifier will render a shape anchored behind a view with the Liquid Glass material and applies the foreground effects of Liquid Glass over a view.

By default, the modifier uses the`[regular](https://developer.apple.com/documentation/swiftui/glass/regular)`variant of`[Glass](https://developer.apple.com/documentation/swiftui/glass)`and applies the given effect within a`[Capsule](https://developer.apple.com/documentation/swiftui/capsule)`shape.

If we have buttons, we can also apply the`[GlassButtonStyle](https://developer.apple.com/documentation/swiftui/glassbuttonstyle)`directly instead of using the`[glassEffect(_:in:isEnabled:)](https://developer.apple.com/documentation/swiftui/view/glasseffect%28%5F:in:isenabled:%29)`modifier

Button(action: {}, label: {  
    text  
})  
.buttonStyle(.glass)

![](https://miro.medium.com/v2/resize:fit:328/1*qQGgQJn0G2Hn50TIzTE3JA.png) 

## Customization

There are couple customization we can make here.

* Change the shape of it by passing in the`shape`parameter.
* Apply a`[tint](https://developer.apple.com/documentation/swiftui/glass/tint%28%5F:%29)`color to the`[Glass](https://developer.apple.com/documentation/swiftui/glass)`.
* Add`[interactive(_:)](https://developer.apple.com/documentation/swiftui/glass/interactive%28%5F:%29)`to make them react to touch and pointer interactions, ie: the same responsive and fluid reactions that`[glass](https://developer.apple.com/documentation/swiftui/primitivebuttonstyle/glass)`provides to standard buttons we have above

VStack {  
    Text("Default")  
    text.glassEffect()  
}  
  
VStack {  
    Text("Shape")  
    text.glassEffect(.regular, in: RoundedRectangle(cornerRadius: 8))  
}  
  
VStack {  
    Text("Tint")  
    text.glassEffect(.regular.tint(.green.opacity(0.4)))  
}  
  
VStack {  
    Text("Interactive")  
    text.glassEffect(.regular.interactive())  
}

![](https://miro.medium.com/v2/resize:fit:526/1*yd6yZ2w_M2J79ZoYTWRiUA.gif) 

## Buggy Behavior

Here is a little problem with`[interactive](https://developer.apple.com/documentation/swiftui/glass/interactive%28%5F:%29)`!

I am not sure whether this is how it SHOULD look like or it is an Apple miss, but let me share it with you here really quick!

Let’s add some shape different from the default.

text.glassEffect(  
  .regular.interactive().tint(.green.opacity(0.4)),   
  in: RoundedRectangle(cornerRadius: 8)  
)

![](https://miro.medium.com/v2/resize:fit:396/1*hz8UGqUhBsf5_VVWLcxUcw.png) 

As you can see, if we press on it, it will react with the default`[Capsule](https://developer.apple.com/documentation/swiftui/capsule)`shape instead of our customized`RoundedRectangle`!

# Glass Effect Container

When we use`[GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer)`together with our`[glassEffect(_:in:isEnabled:)](https://developer.apple.com/documentation/swiftui/view/glasseffect%28%5F:in:isenabled:%29)`modifier above, each view with a glass effect contributes a shape rendered with the effect to a set of shapes.

SwiftUI renders the effects together, improving rendering performance and allowing the effects to interact with and morph into one another.

To create a container, we have the`[init(spacing:content:)](https://developer.apple.com/documentation/swiftui/applying-liquid-glass-to-custom-views)`initializer.

It is really interesting how does the`[GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer)`blend the child view shapes together, animate the transition, and how does that depend on the`spacing`parameter above.

So!

Let’s add couple buttons and sliders to get an overall image!

@State private var offsetX: CGFloat = 0.0  
@State private var offsetY: CGFloat = 0.0  
@State private var containerSpacing: CGFloat = 0.0  
@State private var show: Bool = true  
  
//...  
  
HStack {  
    Text("**Offset X**")  
    Slider(value: $offsetX, in: -60.0 ... 0.0, step: 10.0, label: {}, minimumValueLabel: {  
        Text("-60.0")  
    }, maximumValueLabel: {  
        Text("0.0")  
    })  
}  
  
HStack {  
    Text("**Offset Y**")  
    Slider(value: $offsetY, in: -20.0 ... 20.0, step: 10.0, label: {}, minimumValueLabel: {  
        Text("-20.0")  
    }, maximumValueLabel: {  
        Text("20.0")  
    })  
}  
  
HStack {  
    Text("**Glass Container \nSpacing**")  
    Slider(value: $containerSpacing, in: 0.0 ... 60.0, step: 10.0, label: {}, minimumValueLabel: {  
        Text("0.0")  
    }, maximumValueLabel: {  
        Text("60.0")  
    })  
}  
  
HStack(spacing: 24){  
    Button(action: {  
        withAnimation(.linear(duration: 1), {  
            show.toggle()  
        })  
    }, label: {  
        Text("Show/Hide")  
    })  
    .buttonStyle(.borderedProminent)  
  
    GlassEffectContainer(spacing: containerSpacing) {  
          
        HStack {  
            Image(systemName: "heart.fill")  
                .frame(width: 80.0, height: 80.0)  
                .font(.system(size: 36))  
                .glassEffect(.regular.tint(.green.opacity(0.2)))  
  
            if show {  
                Image(systemName: "heart.fill")  
                    .frame(width: 80.0, height: 80.0)  
                    .font(.system(size: 36))  
                    .glassEffect(.regular.tint(.green.opacity(0.2)))  
                    .offset(x: offsetX, y: offsetY)  
            }  
        }  
    }  
}

![](https://miro.medium.com/v2/resize:fit:590/1*USmYYLYVF-B40ho35nIXig.gif) 

Time for some configuration and customization!

## Customize Transition Behavior

As we can see above, by default, the`[GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer)`will animate the transition using`[matchedGeometry](https://developer.apple.com/documentation/swiftui/glasseffecttransition/matchedgeometry)`, with the`[MatchedGeometryProperties](https://developer.apple.com/documentation/swiftui/matchedgeometryproperties)`set to`frame`and`anchor`set to`center`.

We can customize this by using the`[glassEffectTransition(_:isEnabled:)](https://developer.apple.com/documentation/swiftui/view/glasseffecttransition%28%5F:isenabled:%29)`modifier.

There are two`[GlassEffectTransition](https://developer.apple.com/documentation/swiftui/glasseffecttransition)s`we can choose from.

* `[identity](https://developer.apple.com/documentation/swiftui/glasseffecttransition/identity)`: The identity transition specifying no changes.
* `[matchedGeometry](https://developer.apple.com/documentation/swiftui/glasseffecttransition/matchedgeometry)`for the matched geometry glass effect transition.

// identify  
GlassEffectContainer(spacing: 30) {  
  
    HStack(spacing: 8) {  
        _image(systemName: "heart.fill")  
  
        if show {  
            _image(systemName: "leaf.fill")  
                .glassEffectTransition(.identity)  
        }  
    }  
}  
  
// matchedGeometry  
GlassEffectContainer(spacing: 30) {  
  
    HStack(spacing: 8) {  
        _image(systemName: "heart.fill")  
  
        if show {  
            _image(systemName: "leaf.fill")  
                .glassEffectTransition(.matchedGeometry(properties: [.position]))  
  
        }  
    }  
}  
  
private func _image(systemName: String) -> some View {  
    Image(systemName: systemName)  
        .frame(width: 48, height: 48)  
        .font(.system(size: 20))  
        .glassEffect(.regular.tint(.green.opacity(0.2)))  
  
}

![](https://miro.medium.com/v2/resize:fit:582/1*-ViXkfpeiW7F4w3MvNIJZw.gif) 

## Control Animation Grouping Behavior

First of all, what do I mean by**_Animation Grouping Behavior?_**

Suppose we have three views and we are hiding the second one.

GlassEffectContainer(spacing: 20.0) {  
    HStack(spacing: 16) {  
        let image = _image(systemName: "heart.fill")  
  
        image  
          
        if show {  
            image  
        }  
          
        image  
    }  
}

(By the way, same helper`_image(systemName: String)`function as above!)

![](https://miro.medium.com/v2/resize:fit:488/1*mvPNEcVkgBym7nXdbbXjQg.gif) 

As you can see, the third one is morphed into the second one!

What if we want the second view to go into the first one instead?

Here is where this`[glassEffectID(_:in:)](https://developer.apple.com/documentation/swiftui/view/glasseffectid%28%5F:in:%29)`modifier comes in handy!

If you are used to the`[matchedGeometryEffect](https://developer.apple.com/documentation/swiftui/view/matchedgeometryeffect%28id:in:properties:anchor:issource:%29)`(if not, you can get a quick catch from my previous article[**_SwiftUI: Custom View Transition(Navigation) With Matched Geometry_**](https://blog.stackademic.com/swiftui-custom-view-transition-nav-with-matched-geometry-032552356fc5)), this modifier is almost the same!

Accepting an`id`and a`namespace`to associate each Liquid Glass effect with a unique identifier within a`[Namespace](https://developer.apple.com/documentation/swiftui/namespace)`.

Here is how we can use it!

@Namespace private var namespace  
@Namespace private var namespace2  
  
// ...  
GlassEffectContainer(spacing: 20.0) {  
    HStack(spacing: 16) {  
        let image = _image(systemName: "heart.fill")  
  
        image  
            .glassEffectID(1, in: namespace)  
  
        if show {  
            image  
                .glassEffectID(2, in: namespace)  
        }  
          
        image  
            .glassEffectID(3, in: namespace2)  
  
    }  
}

![](https://miro.medium.com/v2/resize:fit:590/1*yzVePtl7AaykXJSXbPZMgw.gif) 

There are couple little points I would like to point out about this`[glassEffectID(_:in:)](https://developer.apple.com/documentation/swiftui/view/glasseffectid%28%5F:in:%29)`modifier!

First of all, if you have checked out[_Apples’s documentation on Liquid Glass_](https://developer.apple.com/documentation/swiftui/applying-liquid-glass-to-custom-views#Morph-Liquid-Glass-effects-during-transitions), it mentions that we have to

> **Associate** **each Liquid Glass effect with a unique identifier within a namespace the**`[**Namespace**](https://developer.apple.com/documentation/swiftui/namespace)`**property wrapper provides**. These IDs ensure SwiftUI animates the same shapes correctly when a shape appears or disappears due to view hierarchy changes. SwiftUI uses the spacing provided to the effect container along with the geometry of the shapes themselves to determine appropriate shapes to morph into and out of.

No, we don’t have to!

Setting`glassEffectID`to**ALL**views**_we want to animate_**, or to**NONE**of the views will result in the same effect.

let toggle: () -> Void = {  
    withAnimation(.linear(duration: 0.5), {  
        show.toggle()  
    })  
}  
  
GlassEffectContainer(spacing: 20.0) {  
    HStack(spacing: 16) {  
        Button(action: {  
            toggle()  
        }, label: {  
            Text("Toggle (all with ID)")  
        })  
        .buttonStyle(.glass)  
        .glassEffectID("button", in: namespace)  
  
        let image = _image(systemName: "heart.fill")  
        if show {  
            ForEach(0..<3, id: \.self) { index in  
                image  
                    .glassEffectID(index, in: namespace)  
            }  
        }  
    }  
}  
  
GlassEffectContainer(spacing: 20.0) {  
      
      
    HStack(spacing: 16) {  
          
        Button(action: {  
            toggle()  
        }, label: {  
            Text("Toggle (all w/o ID)")  
        })  
        .buttonStyle(.glass)  
  
          
        let image = _image(systemName: "heart.fill")  
        if show {  
            ForEach(0..<3, id: \.self) { index in  
                image  
            }  
        }  
    }  
}

![](https://miro.medium.com/v2/resize:fit:590/1*EgUwl7svGfniLvVl_r5Hag.gif) 

Now, what will happen if we assign`[glassEffectID](https://developer.apple.com/documentation/swiftui/view/glasseffectid%28%5F:in:%29)`to some of the views but not other?

The ones without will**NOT**participate in the animation.

GlassEffectContainer(spacing: 20.0) {  
    HStack(spacing: 16) {  
        Button(action: {  
            toggle()  
        }, label: {  
            Text("Button w/o ID")  
        })  
        .buttonStyle(.glass)  
  
          
        let image = _image(systemName: "heart.fill")  
        if show {  
            ForEach(0..<3, id: \.self) { index in  
                image  
                    .glassEffectID(index, in: namespace)  
            }  
        }  
    }  
}

![](https://miro.medium.com/v2/resize:fit:590/1*RGKkDjANuRLWhuqvIilIWA.gif) 

## Group Shapes Within One Container

To combine the geometries of multiple views to a single liquid glass effect, we can use the`[glassEffectUnion(id:namespace:)](https://developer.apple.com/documentation/swiftui/view/glasseffectunion%28id:namespace:%29)`modifier.

Yes!

Again, that`id`, that`namespace`.

This time, views with the same`id`will be combined into a single shape with the applied Liquid Glass material.

GlassEffectContainer(spacing: 20.0) {  
    HStack(spacing: 16) {  
        let image = _image(systemName: "heart.fill")  
  
        if show {  
            ForEach(0..<4, id:\.self) { index in  
                image  
                    .glassEffectUnion(id: index < 2 ? 0 : 1, namespace: namespace)  
            }  
        }  
    }  
    .frame(height: 64)  
    .frame(maxWidth: .infinity, alignment: .leading)  
}

![](https://miro.medium.com/v2/resize:fit:516/1*WghvnflW2VneAVnRmotoTQ.png) 

Of course, in our case, we**_COULD_**just have two`GlassEffectContainer`and an extra`HStack`wrapping around those.

But!

This modifier is especially useful if we want to create/group views dynamically or for views that live outside of an`[HStack](https://developer.apple.com/documentation/swiftui/hstack)`or`[VStack](https://developer.apple.com/documentation/swiftui/vstack)`.

Here are couple little points to keep in mind!

First of all, the shape of the combined one is determined by the**_FIRST_**shape in the union.

GlassEffectContainer(spacing: 20.0) {  
    HStack(spacing: 16) {  
        let capsule = Image(systemName: "heart.fill")  
            .frame(width: 48, height: 48)  
            .font(.system(size: 20))  
            .glassEffect(.regular.tint(.green.opacity(0.2)))  
          
        let rect = Image(systemName: "heart.fill")  
            .frame(width: 48, height: 48)  
            .font(.system(size: 20))  
            .glassEffect(.regular.tint(.green.opacity(0.2)), in: Rectangle())  
  
  
        ForEach(0..<4, id:\.self) { index in  
            Group {  
                if index % 2 == 0 {  
                    rect  
                } else {  
                    capsule  
                }  
  
            }.glassEffectUnion(id: index < 2 ? 0 : 1, namespace: namespace)  
        }  
    }  
}

![](https://miro.medium.com/v2/resize:fit:536/1*K9U6GX-3wj6qnbYmqX9JGw.png) 

Two! Setting the`spacing`of the`GlassEffectContainer`will not change how view are grouped together, but only the relationship between groups when threshold reached.

![](https://miro.medium.com/v2/resize:fit:572/1*i31Icihguiul_qKwy7NCLA.png) 

 GlassEffectContainer spacing: 0.0

![](https://miro.medium.com/v2/resize:fit:552/1*ySak3OpxDglIg1Z03AHoTA.png) 

 GlassEffectContainer spacing: 20.0

![](https://miro.medium.com/v2/resize:fit:544/1*cRt24MJ3NfeviAGJCo6rqw.png) 

 GlassEffectContainer spacing: 200.0

# Little Code Snippet!

Liquid Glass Effect is pretty interesting (in terms of developing, I don’t really like how it looks though…when I use it)! Please use this little code snippet to give it a try yourself!

  
import SwiftUI  
  
struct ContentView: View {  
      
    @State private var offsetX: CGFloat = 0.0  
    @State private var containerSpacing: CGFloat = 30.0  
    @State private var show: Bool = true  
    @Namespace private var namespace  
    @Namespace private var namespace2  
      
    var body: some View {  
        let toggle: () -> Void = {  
            withAnimation(.linear(duration: 0.5), {  
                show.toggle()  
            })  
        }  
          
        ScrollView {  
            VStack(spacing: 24) {  
  
                Text("Basic Usage")  
                    .font(.headline)  
                  
                let text = Text("˚ʚ♡ɞ˚")  
                    .padding(.all, 8)  
  
                  
                HStack(spacing: 16) {  
                    VStack {  
                        Text("Default")  
                            .font(.subheadline)  
                        text.glassEffect()  
                    }  
                      
                    VStack {  
                        Text("Shape")  
                            .font(.subheadline)  
                        text.glassEffect(.regular, in: RoundedRectangle(cornerRadius: 8))  
                    }  
                      
                    VStack {  
                        Text("Tint")  
                            .font(.subheadline)  
                        text.glassEffect(.regular.tint(.green.opacity(0.4)))  
                    }  
                      
                    VStack {  
                        Text("Interactive")  
                            .font(.subheadline)  
                        text.glassEffect(.regular.interactive())  
  
                    }  
  
                }  
                .frame(maxWidth: .infinity, alignment: .leading)  
  
                VStack {  
                    Text("ButtonStyle")  
                        .font(.subheadline)  
                    Button(action: {}, label: {  
                        text  
                    })  
                    .buttonStyle(.glass)  
  
                }  
                .frame(maxWidth: .infinity, alignment: .leading)  
                  
                HStack(spacing: 24) {  
                    Text("**NOTE**: \nWith `interactive`, when user interacting with the view, the shape used is the default capsule regardless of the parameters set. \nTherefore, if we have customized those parameters, there will be a discrepancy.")  
                        .font(.caption)  
                      
                    text.glassEffect(.regular.interactive().tint(.green.opacity(0.4)), in: RoundedRectangle(cornerRadius: 8))  
  
                }  
                  
                RoundedRectangle(cornerRadius: 4)  
                    .frame(height: 2)  
                 
                  
                Text("Glass Effect Container")  
                    .font(.headline)  
  
                Text("Basic")  
                    .font(.subheadline)  
                    .fontWeight(.bold)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                  
                  
                HStack {  
                    Text("**Offset X**")  
                        .font(.subheadline)  
                    Slider(value: $offsetX, in: -60.0 ... 0.0, step: 5.0, label: {}, minimumValueLabel: {  
                        Text("-60.0")  
                    }, maximumValueLabel: {  
                        Text("0.0")  
                    })  
                }  
  
                HStack {  
                    Text("**Glass Container \nSpacing**")  
                        .font(.subheadline)  
                    Slider(value: $containerSpacing, in: 0.0 ... 60.0, step: 5.0, label: {}, minimumValueLabel: {  
                        Text("0.0")  
                    }, maximumValueLabel: {  
                        Text("60.0")  
                    })  
  
                }  
                  
                HStack(spacing: 24){  
                    Button(action: {  
                        toggle()  
                    }, label: {  
                        Text("Show/Hide")  
                    })  
                    .buttonStyle(.borderedProminent)  
  
                    GlassEffectContainer(spacing: containerSpacing) {  
  
                        HStack(spacing: 8) {  
                            _image(systemName: "heart.fill")  
  
                            if show {  
                                _image(systemName: "leaf.fill")  
                                    .offset(x: offsetX)  
  
                            }  
                        }  
                    }  
                }  
                .frame(maxWidth: .infinity, alignment: .leading)  
                  
                Divider()  
  
                Text("`glassEffectTransition`")  
                    .font(.subheadline)  
                    .fontWeight(.bold)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                  
                Text("When used with `GlassEffectContainer`, SwiftUI will use the provided transition to apply changes to the glass effect when you add or remove views with these effects from the view hierarchy. ")  
                    .font(.caption)  
                    .fixedSize(horizontal: false, vertical: true)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
  
  
                HStack(spacing: 24){  
                    Button(action: {  
                        toggle()  
                    }, label: {  
                        Text("Identity")  
                    })  
                    .buttonStyle(.borderedProminent)  
  
                    GlassEffectContainer(spacing: 30) {  
  
                        HStack(spacing: 8) {  
                            _image(systemName: "heart.fill")  
  
                            if show {  
                                _image(systemName: "leaf.fill")  
                                    .glassEffectTransition(.identity)  
  
                            }  
                        }  
                    }  
                }  
                .frame(maxWidth: .infinity, alignment: .leading)  
  
                HStack(spacing: 24){  
                    Button(action: {  
                        toggle()  
                    }, label: {  
                        Text("Matched Geometry")  
                    })  
                    .buttonStyle(.borderedProminent)  
  
                    GlassEffectContainer(spacing: 30) {  
  
                        HStack(spacing: 8) {  
                            _image(systemName: "heart.fill")  
  
                            if show {  
                                _image(systemName: "leaf.fill")  
                                    .glassEffectTransition(.matchedGeometry(properties: [.position]))  
  
                            }  
                        }  
                    }  
                }  
                .frame(maxWidth: .infinity, alignment: .leading)  
                  
  
                Divider()  
  
                Text("`glassEffectID`")  
                    .font(.subheadline)  
                    .fontWeight(.bold)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                  
                Text("**NOTE**: Setting `glassEffectID` to *all* views we want to animate, or to *none* of the views will result in the same effect. ")  
                    .font(.caption)  
                    .fixedSize(horizontal: false, vertical: true)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
  
  
                GlassEffectContainer(spacing: 20.0) {  
                    HStack(spacing: 16) {  
                          
                        Button(action: {  
                            toggle()  
                        }, label: {  
                            Text("Toggle (all with ID)")  
                        })  
                        .buttonStyle(.glass)  
                        .glassEffectID("button", in: namespace)  
  
                          
                        let image = _image(systemName: "heart.fill")  
                        if show {  
                            ForEach(0..<3, id: \.self) { index in  
                                image  
                                    .glassEffectID(index, in: namespace)  
                            }  
                        }  
                    }  
                    .frame(height: 64)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                }  
                  
                GlassEffectContainer(spacing: 20.0) {  
                      
                      
                    HStack(spacing: 16) {  
                          
                        Button(action: {  
                            toggle()  
                        }, label: {  
                            Text("Toggle (all w/o ID)")  
                        })  
                        .buttonStyle(.glass)  
  
                          
                        let image = _image(systemName: "heart.fill")  
                        if show {  
                            ForEach(0..<3, id: \.self) { index in  
                                image  
                            }  
                        }  
                    }  
                    .frame(height: 64)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                }  
                  
                Text("**NOTE**: \nIf *some* of the views within the `GlassEffectContainer` have `glassEffectID` set while other do not, the ones without will **NOT** participate in the animation.")  
                    .font(.caption)  
                    .fixedSize(horizontal: false, vertical: true)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
  
                GlassEffectContainer(spacing: 20.0) {  
                    HStack(spacing: 16) {  
                          
                        Button(action: {  
                            toggle()  
                        }, label: {  
                            Text("Button w/o ID")  
                        })  
                        .buttonStyle(.glass)  
  
                          
                        let image = _image(systemName: "heart.fill")  
                        if show {  
                            ForEach(0..<3, id: \.self) { index in  
                                image  
                                    .glassEffectID(index, in: namespace)  
                            }  
                        }  
                    }  
                    .frame(height: 64)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                }  
                  
                  
                Text("Use `glassEffectID` to control animation grouping behavior. \nSame `namespace` for views to be grouped together.")  
                    .font(.caption)  
                    .fixedSize(horizontal: false, vertical: true)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
  
                GlassEffectContainer(spacing: 20.0) {  
                    HStack(spacing: 16) {  
                          
                        Button(action: {  
                            toggle()  
                        }, label: {  
                            Text("Default")  
                        })  
                        .buttonStyle(.glass)  
  
                          
                        let image = _image(systemName: "heart.fill")  
  
                        image  
                          
                        if show {  
                            image  
                        }  
                          
                        image  
  
                    }  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                }  
                  
                GlassEffectContainer(spacing: 20.0) {  
                    HStack(spacing: 16) {  
                          
                        Button(action: {  
                            toggle()  
                        }, label: {  
                            Text("Namespace 1=2!=3")  
                        })  
                        .buttonStyle(.glass)  
  
                          
                        let image = _image(systemName: "heart.fill")  
  
                        image  
                            .glassEffectID(1, in: namespace)  
  
                        if show {  
                            image  
                                .glassEffectID(2, in: namespace)  
                        }  
                          
                        image  
                            .glassEffectID(3, in: namespace2)  
  
                    }  
                    .frame(maxWidth: .infinity, alignment: .leading)  
                }  
  
                  
                Divider()  
  
                  
                Text("`glassEffectUnion`")  
                    .font(.subheadline)  
                    .fontWeight(.bold)  
                    .frame(maxWidth: .infinity, alignment: .leading)  
  
  
                Text("**NOTE**: \n1. The shape of the combined view is determined by the FIRST shape in the union. \n2. Setting `GlassEffectContainer` `spacing` will not effect how views are grouped together, but only the relationship between groups when threshold reached.")  
                    .font(.caption)  
                    .fixedSize(horizontal: false, vertical: true)  
  
                let containerSpacings = [0.0, 20.0, 120.0]  
  
                ForEach(containerSpacings, id: \.self) { spacing in  
                    VStack {  
                        Text("Container spacing: \(String(format: "%.1f", spacing))")  
                            .font(.subheadline)  
                            .minimumScaleFactor(0.5)  
                            .frame(maxWidth: .infinity, alignment: .leading)  
                          
                        GlassEffectContainer(spacing: spacing) {  
                              
                            HStack(spacing: 16) {  
                                  
                                let capsule = Image(systemName: "heart.fill")  
                                    .frame(width: 48, height: 48)  
                                    .font(.system(size: 20))  
                                    .glassEffect(.regular.tint(.green.opacity(0.2)))  
                                  
                                let rect = Image(systemName: "heart.fill")  
                                    .frame(width: 48, height: 48)  
                                    .font(.system(size: 20))  
                                    .glassEffect(.regular.tint(.green.opacity(0.2)), in: Rectangle())  
                                  
                                  
                                ForEach(0..<4, id:\.self) { index in  
                                    Group {  
                                        if index % 2 == 0 {  
                                            rect  
                                        } else {  
                                            capsule  
                                        }  
                                          
                                    }.glassEffectUnion(id: index < 2 ? 0 : 1, namespace: namespace)  
                                }  
                            }  
                            .frame(height: 64)  
                            .frame(maxWidth: .infinity, alignment: .leading)  
                              
                        }  
  
                    }  
                      
                }  
  
            }  
            .padding(.horizontal, 16)  
            .padding(.vertical, 32)  
  
        }  
        .background(.yellow.opacity(0.1))  
  
    }  
      
      
    private func _image(systemName: String) -> some View {  
        Image(systemName: systemName)  
            .frame(width: 48, height: 48)  
            .font(.system(size: 20))  
            .glassEffect(.regular.tint(.green.opacity(0.2)))  
  
    }  
}

Thank you for reading!

That’s it for this article!

Happy glassing!
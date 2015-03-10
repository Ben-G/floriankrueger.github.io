---
layout:     post
title:      "AutoLayoutKit"
date:       2013-12-24 13:00:00
categories: iOS OSX Cocoa CocoaPods AutoLayout Framework
cover:      '/assets/images/post-autolayoutkit/cover.png'
published:  true
sitemap:
  lastmod: 2014-12-24
  priority: 0.7
  changefreq: 'monthly'
  exclude: 'no'
---

A few days ago I proudly released my first OpenSource iOS component via CocoaPods: [AutoLayoutKit](https://github.com/floriankrueger/AutoLayoutKit). It's a tiny DSL that lets you create and modify `NSLayoutConstraints` in code without all the hassle when using the native API that Apple gave us for this purpose.

First of all: Why do I even care about the native `NSLayoutConstraint` API? Apple clearly states (citation needed) that the most convenient way of creating `NSLayoutConstraint`s is the Interface Builder (IB). If we need to do Layout in code we are are given the *"Visual Format Language"* (kind of an ASCII art approach, I'm going to call it "VFL" from now on) and only for cases where the VFL is not expressive enough (this is a known scenario), we might consider using the native API.

Well, in most of my projects we started out creating layout constraints in IB before we moved on to the VFL and finally dropped deeper to the native API for the cases that still weren’t possible or easily achievable using one of the two abstractions. At this point of course, all of the layout definition was spread between the different tools and approaches. This is why I (personally) don’t do layout constraints in IB or with the VFL. If I have the choice, I don’t use IB at all. There are plenty of sources that you can ask for reasons not to use IB. My main reason is: I’m a Software Developer after all and I do code, not a fancy clickety-click user interface that removes 50% of the framework's features for the sake of abstraction.

"OK, fine" you might say now and check the native constraint API to see what I'm talking about. "How do I, for example, align a childview to its superview's center?" you might ask. Here's how you do it (assuming `self` is the superview):


```objective-c
NSLayoutConstraint *c;

c = [NSLayoutConstraint constraintWithItem:childView
                                 attribute:NSLayoutAttributeCenterX
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:self
                                 attribute:NSLayoutAttributeCenterX
                                multiplier:1.f
                                  constant:0.f];
[self addConstraint:c];

c = [NSLayoutConstraint constraintWithItem:childView
                                 attribute:NSLayoutAttributeCenterY
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:self
                                 attribute:NSLayoutAttributeCenterY
                                multiplier:1.f
                                  constant:0.f];
[self addConstraint:c];

```


As you can clearly see at first sight, this code creates two constraints and adds them to self. The first centers the `childView` horizontally in the superview, the second one vertically. You're not seeing it? Ah wait, of course: I forgot to determine the size of the `childView` (this is not necessary if the childView has an intrinsic content size).


```objective-c
c = [NSLayoutConstraint constraintWithItem:childView
                                 attribute:NSLayoutAttributeWidth
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:nil
                                 attribute:NSLayoutAttributeNotAnAttribute
                                multiplier:1.f
                                  constant:30.f];
[childView addConstraint:c];

c = [NSLayoutConstraint constraintWithItem:childView
                                 attribute:NSLayoutAttributeHeight
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:nil
                                 attribute:NSLayoutAttributeNotAnAttribute
                                multiplier:1.f
                                  constant:30.f];
[childView addConstraint:c];
```


Well, that was *"easy"*. I think by now you got the point: This API is ugly as hell and as inconvenient as possible.

## A DSL to the Rescue

This is where **AutoLayoutKit** comes into play. Let me rewrite the whole statement from above.

```objective-c
[ALKConstraints layout:childView do:^(ALKConstraints *c) {
  [c make:ALKCenterX equalTo:self s:ALKCenterX];
  [c make:ALKCenterY equalTo:self s:ALKCenterY];
  [c set:ALKWidth to:30.f];
  [c set:ALKHeight to:30.f];
}];
```

Oh wait, that's it? **Yes**. And you can even add constants and multipliers here. The trick is, that I took the extra effort to create some methods that assume default values (like 1.f for the multiplier and 0.f for the constant) when the developer doesn't require them to be some other value.

I didn't even stop there: AutoLayoutKit keeps track of all of your constraints that you might want to modify later on. Just add an optional name parameter and call the constraint whatever you want it to be called. Whenever you need to access the constraint, ask the targetView for the constraint with the name and *boom* there it is.

## More?

So that's basically all that AutoLayoutKit does FOR you. But there's more: AutoLayoutKit works best when used with a pattern that I developed throughout my time as an iOS engineer. It's heavily inspired by various open source projects and examples I've seen and used so most of the credit goes to the open source community.

## From ViewController to View-Controller

So we all know that Cocoa and CocoaTouch are MVC frameworks by heart. There are classes and frameworks for the Model (e.g. `CoreData` and `ManagedObjectModels`), there are Views (e.g. Buttons or Sliders) and there are Controllers: ViewControllers - most of the time.

The original idea is (or was), that all the "V" stuff of MVC is done in IB, all the "M" stuff is done in CoreData and the coordination between them is done using ViewControllers. That used to be the "C" part, but (especially when you stop using IB) there is a great chance that ViewControllers become the "VC" part. It’s just a matter of integrity that there isn't a `ModelViewController` class already.

To avoid this, I started to draw a thick, red line between Views and Controllers and returned to the original way of View Classes:

**Every ViewController has a view but it has nothing to do or even knowledge about things like subviews or layout.**

This might seem very obvious but just take a walk around GitHub and explore some example projects or even go through some of the programming examples provided by Apple. Often View Controllers do things like layouting subviews, creating and maintaining buttons and imageviews, moving them around with animations, while the ViewControllers' view remains a plain UIView with no further implementation. A shallow container that knows nothing about its content. It sounds ridiculous to set background colors on UIButtons to create a highlighted state; we use `setHighlighted:` instead, leaving the rest to the UIButton. But when it comes to the ViewControllers' view, we tend to write lines and lines of code inside `loadView`, or even worse `viewDidLoad`.

## The View-Controller

There are four things I do when creating a new ViewController now:

1. Create a `UIViewController` subclass named `<Prefix><Purpose>ViewController` (e.g. `ALKExampleViewController`)
2. Create a `UIView` subclass named `<Prefix><Purpose>View` (e.g. ALKExampleView)
3. Create the `UIView` inside `loadView` and assign it to the VC's `view` property
4. Create `setup`, `setupSubviews` and `layoutSubviews` methods in the view (there's a template [here](https://gist.github.com/floriankrueger/7667447))

Afterwards, there are a few rules that apply:

1. All subview creation code goes into `setupSubviews`
2. All subview layout code goes into `setupLayout`
3. All setup of the view itself goes into `setup` before or after the calls to the other setup methods (whichever is best for your purpose)
4. All subviews are private (there's an exception)
5. State and content of the view is maintained through designated getter and setter methods, either created by properties or manually. How that state or content is presented is completely up to the view

Of course, there are exceptions:

1. The view is layouted within the `loadView` method of the ViewController. This is most likely the size of the UIScreen bounds and flexible height and width through resizing masks (yes).
2. Sometimes I make Buttons readonly properties. That is, when I need to attach targets to these buttons and I don't want to route these through custom methods (you can really go to far at this point). The remaining rule is: No layouting or other visual modification of the view.
3. In some special cases, the ViewControllers' view needs to have very specific layouting parameters you might not be aware of to work properly. In those cases it's best practice to add the view as a "fullscreen" subview to the automatically created view of the ViewController in `viewDidLoad`.

## Conclusion

*AutoLayoutKit* is a nice way of managing your `NSLayoutConstraints` in code without the need to write a book full of layout definitions. It's not complete yet and there are some unsolved conceptual issues like the way you need to deal with layout guides in iOS7 but I am working on that (and you're welcome if you like to contribute). More important than using *AutoLayoutKit* is: keep M, V and C separated and don't use IB. It'll kill you sooner or later.

---
title: "RFC: A better design for handling Mobjects in Manim"
date: "2022-05-06"
draft: false
comments: true
discussionId: rfc-mobjects
---


*This is a draft proposal for a redesign on Manim's mobjects. I have no idea whether this is a good idea or not.*

In Manim currently, Mobjects are initialized as instances of certain classes. These classes themself follow an inheritance structure, with `VMobject` being the parent of most geometrical objects. While this approach seems sensible at first glance, there are a number of issues with it that lead me to believe that it is not a clean approach:

- An object's type does not necessarily match its geometry: A user can, for example, create a `Square` and transform it to a `Circle`. However, after the transformation, the transformed Mobject is still a `Square`; the only thing that has changed is the rendering information (geometry and style data). If they were to try to access this Mobject's radius for example, they would not be able to, since the Mobject is still secretly a square. The mismatch that can occur between a Mobject's actual geometry with its type, attributes and methods can cause confusion and reflects Manim's poor design. Furthermore, if a renderer would like to optimize the rendering of certain Mobjects, it cannot determine whether a Mobject of type `Square` is really a square, making the task very difficult.

- Other problems with Transformation: To transform a Mobject to another Mobject, Manim currently has to modify all attributes related to geometry and style data. These attributes are ungrouped and it can be difficult for humans to understand and keep track of all these attributes. It would be better if this information was properly encapsulated.

- Geometrical objects are tied too closely to VMobjects: other than a few exceptions, every Mobject in Manim is a VMobject. A VMobject (vectorized mobject) is essentially a collection of bezier curves. This offers a lot of flexibility - this is what allows all VMobjects to share common animations - Create, Write and transformation between arbitrary VMobjects. However, this flexibility comes at the expense of long render times. It's often the case that such flexibility is not required; the user should be able to decide when this is true and switch to a cheaper rendering method in those cases.

In short, using types (which can't be changed after instantiation) to provide information on Mobjects isn't a good idea, when a large part of the philosophy of Manim is in allowing Mobjects to freely transform.

## Immutability?

In the past, I have considered ideas with Mobject immutability, i.e. Mobject data is locked unless specifically unlocked. While this resolves the renderer optimization problem (a renderer can trust that a locked object of a certain type really has geometry corresponding to that type), it does not solve the problems of mismatched type and attributes, and it also does not make sense to render a special type of VMobject in a way that ignores its VMobject-ness (being composed of bezier curves) entirely.

## A proposed solution

Let's take a `Circle` for example. Normally, when `Circle()` is run, Python calls the `__new__` method, instantiating a new object of the `Circle` type. However, I propose that instead of doing this, we instead instantiate an abstract mobject wrapper object, and pass the specific information to be encapsulated within the wrapper. This might look something like this:

```py
class Object:
    pass


class Wrapper:
    def __init__(self, cls, *args, **kwargs):
        self.obj = cls._initialize(*args, **kwargs)
        self.rendering_data = self.obj._to_data()
        # or perhaps just a cached property that should be 
        # accessed at render time


class Mobject:
    @classmethod
    def _initialize(cls, *args, **kwargs):
        obj = Object.__new__(cls)
        obj.__init__(*args, **kwargs)
        return obj

    # etc

class Circle(Arc):
    def __new__(cls, *args, **kwargs):
        return Wrapper(Circle, *args, **kwargs)

    def __init__(self, radius, color, **kwargs):
        self.radius = radius
        self.color = color 
        super().__init__(
        # stuff,
        **kwargs
    )
```

This way, when a `x = Circle()` is run, what is actually assigned to `x` is a wrapper encapsulating a `Circle` object.

This has multiple advantages. When transforming one object to another, Manim can simply swap out the `obj` attribute of the wrapper (first to something like `GeneralVMobject` and finally to the destination object). Since the methods and attributes of the encapsulated object change correctly, this also allows certain methods (like getting the bounding box) to be specialized for certain types of Mobjects, rather than always using the expensive general VMobject method.

We could also have a `material` attribute with the wrapper. This determines whether the Mobject is rendered, for example, as a VMobject or primitive. Another use case is rendering a surface either as a checkerboard or a proper OpenGL mesh. The decoupling of renderer specific information from Mobjects also helps in supporting multiple renderers (although I think that in the long term, it's too complicated to support multiple renderers, unless we hire someone).

## Things I haven't figured out yet

Where exactly should the methods for determining rendering data (in the case where we're rendering as a VMobject, where the bezier curves should be located) live?

We should be very clear on what style attributes we want VMobjects to have. Things like varying stroke width and custom color gradients are going to be very hard to represent without vertex information, and it's perhaps worth scrapping them from Manim entirely.

## Disadvantages

Overriding the `__new__` method is not exactly that clean either.

Users will have to type `thing.obj.attribute` rather than just `thing.attribute` to access what they want.

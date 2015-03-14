# Things Autodesk Can Do to Improve Python API #

Now that I've started to use the API more regularly, i'm having deja vu all over again. Once again it is painfully obvious that I am using python to interact with another language -- this time c++ --because so many of the python idioms are broken.

## No Brainers ##

lets start with a point, a point array, and a transform to demonstrate a few simple usability features. If you're following along at home, the code below will make the objects that we'll use in the rest of the examples:

```
>>> import maya.OpenMaya as api
>>> # a point
>>> pt = api.MPoint(1,2,3)
>>> 
>>> # a point array
>>> ptArray = api.MPointArray()
>>> ptArray.append( pt )
>>> ptArray.append( api.MPoint(4,5,6) )
>>>
>>> # a transform
>>> sel = api.MSelectionList()
>>> dag = api.MDagPath()
>>> sel.add( 'persp' )
>>> sel.getDagPath( 0, dag )
>>> trnsFn = api.MFnTransform( dag )
```

### str and repr ###

for classes that represent numeric types -- MPoint, MVector, MMatrix, MAngle, MTime, etc -- it would be nice to have `__str__` or `__repr__` produce more useful information.

```
>>> print pt
[1,2,3]
>>> print repr(pt)
maya.OpenMaya.MPoint( 1,2,3 )
```

for types like MFn's where a valid repr would be difficult to produce, it would be great to at least have a str method that gave us the object's name:

```
>>> print trnsFn
persp
```

### len ###

so simple, so useful:

```
>>> len(pt)
3
>>> len(ptArray)
2
```

### iter ###

It would certainly be great to be able to iterate through all the array-type classes, such as MPoint, MVector, MMatrix, the M\*Array classes (MPointArray, MDagPathArray, etc), and the MIt**classes.**

```
>>> for comp in pt: print comp
1
2
3
```

combining iteration with the hypothetical `__repr__` above makes for nice results with MPointArray:

```
>>> for pt in ptArray: print repr(pt)
maya.OpenMaya.MPoint( 1,2,3 )
maya.OpenMaya.MPoint( 4,5,6 )
```

I should mention that right now if you try this, you'll enter an infinite loop. Fun.

once `__len__` and `__iter__` are implemented we'll get the ability to cast to python iterables:

```
>>> list(pt)
[1,2,3]
>>> list(ptArray)
[maya.OpenMaya.MPoint( 1,2,3 ), maya.OpenMaya.MPoint(4,5,6)]
```

### More Keyword Arguments ###

There are certain commonly use methods that take a slew of arguments, and while the API documentation describes the default values for these, it does not allow the use of python keywords.

a great example is MPlug.partialName.  Let's say all these defaults look fine except you want to use long names. You currently have to do this:

```
myplug.partialName( False, #includeNodeName
                  False, #includeNonMandatoryIndices
                  False, #includeInstancedIndices
                  False, #useAlias
                  False, #useFullAttributePath
                  True #useLongNames
)
```

I add the comments so that i can keep track of what the heck is going on. Wouldn't it be great if you could do this:
```
myplug.partialName(useLongNames=True) 
```


## Advanced Problems ##

### Array Pointers ###

Currently, if you want to do something as simple as set the scale of a transform, you're facing a lot of MScriptUtil legwork, because it expects a const double scale[3](3.md)

```
>>> # set the scale of the transform ( instantiated above )
>>> su = api.MScriptUtil()
>>> dblPtr = su.asDoublePtr()
>>> su.setDoubleArray( dblPtr, 0, 2.0 )
>>> su.setDoubleArray( dblPtr, 1, 2.0 )
>>> su.setDoubleArray( dblPtr, 2, 2.0 )
>>> trnsFn.setScale( dblPtr )
```

Would it be possible to accept a list of python values?

```
>>> # set the scale of the transform ( instantiated above )
>>> trnsFn.setScale( [2.0, 2.0, 2.0] )
```

In general, anywhere that an array ( double3, float3, etc ) is expected as an argument or returned as a result, a python list of appropriate values should be used instead.
Array Types

All the various array types -- MPointArray, MVectorArray, MFloatVectorArray, MDagPathArray, MColorArray -- are a real annoyance.

There are two viable solutions:

  1. Enable instantiation of array types from python lists/tuples

> - ex api.MPointArray( [pt, pt] ) - provides partial alleviation of the problem without deviating too much

> 2. Get rid of them altogether in favor of python lists

> - would be most user-friendly of the two options

### Ease Up On Argument Restrictions ###

Here's a simple way to make all API methods more user-friendly: internally, the method should always attempt to cast input arguments to the expected input type instead of rejecting them outright if they're not the exact type expected.

lets take MFnTransform.setTranslation for example:: setTranslation (const MVector &vec, MSpace::Space space).

The first argument is an MVector, which can be instantiated like so:: MVector (const double d[3](3.md)).

Assuming that we fix the problems discussed under the topic Array Pointers above, we should be able to create an MVector like this:

```
>>> vec = api.MVector([1.0, 0.0, 0.0])
```

so, if a user attempts to set the translation like this:

```
>>> trnsFn.setTranslation( [1.0, 0.0, 0.0],  api.MSpace.kWorld )
```

the internal argument parsing mechanism should first attempt to cast this input to an MVector and, since it can be cast to an MVector, this shorthand syntax should succeed. Every time you pass an integer to an argument expecting a float and it works, python is doing this for you. We need to extend this philosophy to maya as well.

### Units ###

There are three unit types in Maya -- MTime, MDistance, MAngle -- and of them, only MTime is handled in a user-friendly fashion.

Check this out. You can create MTime instances with different unit types, do math with them, and the differing units will be properly handled behind the scenes ( the result is stored as the unit type of the leftmost operand ) :

```
>>> tfilm = api.MTime( 10, api.MTime.kFilm )
>>> tntsc = api.MTime( 10, api.MTime.kNTSCFrame )
>>> res = tfilm + tntsc
>>> res.value()
18.0
```

MDistance and MAngle are brutally crippled in comparison, supporting no math ops whatsoever. What's worse is that they are very rarely used as arguments or results ( MDistance is only used with MPlug, afaik). Why is this so bad? Well, when a function returns an MTime, it returns it in the current UI unit, which is handy and MEL-like. Linear values, on the other hand, are returned as doubles in centimeters. always in centimeters. And there's no formal way of knowing that the result is a distance. RTFM, i guess.

Ideally, if meters are in effect, and I get a value that is linear, it should be in meters as an MDistance, and that MDistance should have the ability to add/subtract/multiply/divide/etc with other floats and MDistances. Same for MAngle with degrees vs. radians.

Distances and angles are a bit more tricky than time, because they compose more complicated types, such as vectors, points, quaternions, eulers and matrices. So, yes, these types should have methods for properly handling units as well.

### Proper Results ( aka, NOT Passing by Reference ) ###

If you've ever used PyQt you may have noticed that in wrapping the Qt framework, Riverbank made a very intelligent decision with regard to outputs that are retrieved by passing references: they got rid of them.

if the C++ Qt function looks like this:

```
void getSomeValue( QString key, float &value )
```

then the PyQt function looks like this:

```
value getSomeValue( key )
```

if there is more than one output being passed to the Qt function as a reference, then the result of the PyQt function is a tuple of outputs. This is a great example of adopting python's idioms instead of forcing python to work like C++. I don't profess to know the details of sip vs. swig, but it would certainly save a lot of headache if a similar convention were adopted by Maya.

## Other Annoyances and Inconsitencies ##

Here's i'll be making a list of things that make me crazy as I come across them.

`MScriptUtil.createMatrixFromList` should be `MMatrix.__init__`

all math types can that can be meaningfully converted into each other, should be capable of doing so via `__init__` :

  * `MMatrix(MFloatMatrix())`
  * `MFloatMatrix(MMatrix())`
  * `MEulerRotation(MMatrix())`
  * etc

`MMatrix()(0,1)` should be `MMatrix()[0,1]`

`MPlug() == MObject()` works but `MObject() == MPlug()` raises an error





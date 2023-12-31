# CVE-2023-41993

PoC exploit for CVE-2023-41993.
It's written only up to addrof/fakeobj.
Reliability is not great.
If you want to make it better, try to spray structure IDs.

## Brief Explanation

You may want a detailed writeup for this.
but unfortunately I'm not afford the time to write the thing.
So I write some note here so you can understand how this works.

If you see the commit, it's about the change for HeapLocation.
New factor has been added to know whether the nodes are same or not.
It says us that the nodes like GetByOffset, MultiGetByOffset could be confused.
But actualy it's only about the offset.
Let's say if it has two GetByOffset offset that has different offset.
one of them are going to be CSEed and the leftover will be used instead.
So it's basically offset confusion but it doesn't give you the access to an arbitrary offset.
because to CSE such types of nodes they need to be hoisted by LICMPhase.
For the kinds of node which does write operation is not allowed to be hoisted in this phase.
Same confusion doesn't happens to PutByOffset, MultiPutByOffset nodes.
Also when GetByOffset is hoisted, safeToExecute function is called to see the node is legit to execute and allows to access only the offset less than storage capacity(inline/ool).
So the idea to exploit this was GetterSetter.
If you call `Object.__defineGetter__` to define a property, GetterSetter object is created but stored in the property storage and this not accessible in normally.
But you can with this offset manipulation you have.
Then you call `Object` function to trigger an type confusion.

```
JSObject* JSCell::toObjectSlow(JSGlobalObject* globalObject) const
{
    Integrity::auditStructureID(structureID());
    ASSERT(!isObject());
    if (isString())
        return static_cast<const JSString*>(this)->toObject(globalObject);
    if (isHeapBigInt())
        return static_cast<const JSBigInt*>(this)->toObject(globalObject);
    ASSERT(isSymbol());
    return static_cast<const Symbol*>(this)->toObject(globalObject);
}
```

It will create an SymbolObject which the GetterSetter as an internal value.
And this is incorrect, because this internal value of SymbolObject is supposed to be an Symbol instance not GetterSetter.

```
let getterSetter = jitme(1);
let symbolObject = Object(getterSetter);

symbolObject.description; // call the getter
```

```
String Symbol::description() const
{
    auto& uid = m_privateName.uid();
    return uid.isNullSymbol() ? String() : uid;
}
```

Then when you call the description getter, it will return an String instance.
This is a type confusion between Symbol.m_privateName and GetterSetter.m_getter.
Each time you call this getter, it will increase the reference counter field of m_privateName.m_uid which is at offset 0x0.
This is very useful, because this offset is where the structure ID of getter function is.
By calling this function some times, you can change the structure ID of JSFunction instance.
I have prepared another type which has many property.
Then if you sync the structure ID to be same with it, you can do oob write to property storage.
This directly gives the addrof/fakeobj primitive.

## Reference

- Int64 module: https://github.com/saelo/jscpwn
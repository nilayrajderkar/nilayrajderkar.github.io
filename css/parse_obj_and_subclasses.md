TIL: Pydantic's model_validate Doesn't Automatically Detect Subclasses.

## The Problem: When Inheritance Isn't Enough 
Replicating this with a simpler example. I defined a `Shape` base model with subclasses `Circle` and `Rectangle`. I assumed that if I defined a `Circle` subclass of `Shape`, passing a JSON object with a `diameter` field would return a `Circle` instance. **it doesn't work that way**. 

```
from pydantic import BaseModel  
import json

class Shape(BaseModel):  
sides: int  
is_2d: bool

class Circle(Shape):  
diameter: int

class Rectangle(Shape):  
perimeter: int

def getActualObject(objString: str) -> Shape:  
objJson = json.loads(objString)  
return Shape.model_validate(objJson)
```

Tested this with JSON like `{"sides": 0, "is_2d": true, "diameter": 10}`, I expected a `Circle` instance. Instead, the output is:

```
sides=0 is_2d=True
```


**Wait, where's the `diameter` field?**  
Turns out, `model_validate` on the base class `Shape` ignores subclass fields. It only validates against the base model's schema.  
Inheritance alone doesn't trigger polymorphic parsing.

---

## Initial Workarounds: Manual Type Checks

This can be hacked by checking for subclass-specific fields:

```
def getActualObject(objString: str) -> Shape:  
objJson = json.loads(objString)  
if 'diameter' in objJson:  
return Circle.model_validate(objJson)  
elif 'perimeter' in objJson:  
return Rectangle.model_validate(objJson)
```


This worked, but it felt brittle. Adding other `Triangle` or Shape subclasses later would  
need an update to the `if`/`elif` chain every time. Not scalable

---

## The Real Solution: Discriminated Unions

Pydantic has a built-in mechanism for this: **discriminated unions**.  
By adding a **discriminator field** (e.g., `shape_type`), you tell Pydantic which subclass to use:

```
from pydantic import TypeAdapter, Field, BaseModel  
from typing import Union, Literal, Annotated

class Circle(BaseModel):  
shape_type: Literal['circle'] # Discriminator value  
diameter: int

class Rectangle(BaseModel):  
shape_type: Literal['rectangle'] # Discriminator value  
perimeter: int

ShapeType = Annotated[  
Union[Circle, Rectangle],  
Field(discriminator='shape_type') # Use 'shape_type' to select the subclass  
]

def getActualObject(objString: str) -> Shape:  
objJson = json.loads(objString)  
return TypeAdapter(ShapeType).validate_python(objJson)

```

returns

```
test_input = '{"shape_type": "circle", "diameter": 10, "sides": 0, "is_2d": true}'  
shape = getActualObject(test_input)  
print(shape) # Output: shape_type='circle' diameter=10 sides=0 is_2d=True
```


**Key takeaways:**
1. **Discriminator Field**: Each subclass declares a `Literal` field (e.g., `shape_type: Literal['circle']`).  
2. **Annotated Union**: The `Annotated[Union[...], Field(discriminator='...')]` tells Pydantic to use the discriminator to pick the right subclass.  
3. **Validation**: `TypeAdapter` handles the parsing logic automatically.

---

## Why This Works: Behind the Scenes

1. **Performance**: Discriminated unions avoid brute-force validation against all possible subclasses.  
   Pydantic checks the discriminator field first, then validates only the relevant subclass.  
   (See [Pydantic's Union docs](https://docs.pydantic.dev/latest/concepts/unions/)).  
2. **Predictability**: Explicit discriminators eliminate ambiguity. No more guessing based on field presence.  
3. **Scalability**: Add new subclasses without modifying parsing logic. Just update the union!

---

## Common Pitfalls I Hit

### 1. Forgetting the Literal Type

```
class Circle(BaseModel):  
shape_type: str = 'circle' # ❌ Not a Literal!  
diameter: int
```

**Fix**: Use `Literal` to enforce exact discriminator values
`shape_type: Literal['circle'] = 'circle' # ✅`

### 2. Missing Discriminator in JSON
If your JSON lacks `shape_type`, Pydantic raises a validation error.  
**Solution**: Ensure the discriminator field is always present.

### 3. Using BaseModel.model_validate Directly

 Still returns a Shape instance
 **Fix**: Always parse via the `Union` (e.g., using `TypeAdapter`)

Eliminate Pass Through Values.

Passing real data to purposely enforce execution order is an anti pattern.
The language shouldn’t have non modified pass through values.

Discussions here:
- [Just Passing Through](https://dqmh.org/just-passing-through/)
- [Your LabVIEW Code Is a Work of Art... But I Can't Read It by Darren Nattinger. GDevCon N.A. 2024](https://www.youtube.com/watch?v=AHOZ7fiuWCA) @
- [An End to Brainless Programming - Darren Nattinger](https://www.youtube.com/watch?v=pS1UBZzKl9k) @

The Flat Sequence Structure:

![FSS_Error](Images/FSS_Error.png)
*Flat Sequence Structure usage.*

![Bounds_Of_Always_Copy](Images/Bounds_Of_Always_Copy.png)
*Bounds of where Always Copy can be placed. If the Always Copy is beyond these bounds, then get its position, delete it, and insert position that is within bounds by adjusting the positions that are out of bounds by using the **Insert Point** input on **Wire:Insert Node**. Also for **Alway Copy** within structure, prevents this below.*

![Always_Copy_Within](Images/Always_Copy_Within.png)
*Prevent this action by always placing at the vertices of where the fss should go on the **first** fss placement (which will be deleted later in script).*
# Enclose-In-Structure

A [Quick Drop](https://forums.ni.com/t5/Quick-Drop-Enthusiasts/bd-p/grp-1251) plugin for LabVIEW that encloses selected block diagram elements in a structure, automatically wiring pass-through values. Two shortcuts — `ctrl+s` to enclose, `ctrl+shift+s` to replace / cleanup — bound to **S** like **S**tructure. The tool creates a maximally compact structure around the selection, so developers can select liberally and let the tool handle cleanup.

Back-saved to LabVIEW 2020.

Repository: [github.com/natev51/enclose-in-structure](https://github.com/natev51/enclose-in-structure)


## Motivation

Passing unmodified values through a VI solely to enforce execution order is an anti-pattern — it degrades readability and obscures intent. If a wire exists only for serialization, the VI should instead be enclosed in a Flat Sequence Structure.

![FSS_Error](img/FSS_Error.png)
*Preferred: Flat Sequence Structure serializing a VI with the error wire (left). Avoided: passing the error wire through a VI that does not mutate it (right).*

LabVIEW's built-in Quick Drop and right-click options for enclosing selections in structures do not pass selected wires through the structure. This plugin fills that gap: enclose a selection, and wires pass through automatically based on the structure type.

![Structure_Placement](img/Structure_Placement.png)
*Structure placement around selected elements.*


## Usage

### Enclose In Structure — `ctrl+s`

1. Select object(s) on the block diagram.
2. Press `ctrl+space` to open Quick Drop.
3. Type the structure abbreviation (e.g. `fss`, `ws`, `fr`).
4. Press `ctrl+s`.

The structure is placed compactly around the selection. The final action of the shortcut selects the structure, enabling immediate nesting — press `ctrl+space` again and repeat to enclose the structure in another structure.

Pressing `ctrl+s` without a structure abbreviation defaults to a **Flat Sequence Structure**.

### Replace With Structure — `ctrl+shift+s`

1. Select a structure on the block diagram.
2. Press `ctrl+space` to open Quick Drop.
3. Type the target structure abbreviation.
4. Press `ctrl+shift+s`.

This replaces the selected structure with the specified one. LabVIEW natively only supports limited structure-to-structure replacements (e.g. Flat Sequence to Stacked Sequence). Attempting unsupported replacements via the `GObject.Replace` method returns error `0x482: Cannot replace this object with the type specified`. This shortcut works around that limitation by enclosing the contents in the new structure and deleting the original.

The `ctrl+s` shortcut is the primary feature (hence the plugin name), and `ctrl+shift+s` is a derivative. To replace a non-structure selection with a structure, first enclose with `ctrl+s`, then `ctrl+r` the internal objects.

At the end, the new structure is selected — same as `ctrl+s`.

If a structure is selected and is replaced by the same structure, the action is effectively a cleanup where the structure is resized to be maximally compact.

### Structure Abbreviations

The Quick Drop text field determines which structure to use. Abbreviations match existing Quick Drop conventions:

| Abbreviation    | Structure                           |
| --------------- | ----------------------------------- |
| `fss`           | Flat Sequence Structure *(default)* |
| `fs`            | For Loop                            |
| `ws`            | While Loop                          |
| `ts`            | Timed Loop                          |
| `tsq`           | Timed Sequence                      |
| `es` or `evstr` | Event Structure                     |
| `nes`           | In Place Element Structure          |
| `dds`           | Diagram Disable Structure           |
| `cds`           | Conditional Disable Structure       |
| `tss`           | Type Specialization Structure       |
| `cs`            | Case Structure                      |


## Supported Structures and Pass-Through Behavior

For any selected wire identified as a pass-through wire, the plugin wires it through the structure according to the structure type.

**Single-Frame Structures**

| Structure        | Pass-Through Method                                                                                            |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| While Loop       | Shift Registers                                                                                                |
| For Loop         | Shift Registers                                                                                                |
| Flat Sequence    | Tunnel (pass through)                                                                                          |
| Timed Loop       | Shift Registers                                                                                                |
| Timed Sequence   | Tunnel (pass through)                                                                                          |
| In Place Element | Tunnel (pass through). For clusters or containing classes: unbundle/bundle with pass through of first element. |

**Multi-Frame Structures**

| Structure | Pass-Through Method |
|---|---|
| Case Structure | Linked Input Tunnel, then create and wire unwired cases |
| Event Structure | Linked Input Tunnel, then create and wire unwired cases |
| Diagram Disable | Linked Input Tunnel, then create and wire unwired cases |
| Conditional Disable | Linked Input Tunnel, then create and wire unwired cases |
| Type Specialization | Linked Input Tunnel, then create and wire unwired cases |

### Excluded Structures

The following are intentionally unsupported:

- **Stacked Sequence** — Does not allow pass-through wires (they're stacked). Also considered poor practice.


## Structure Properties

All structures created by this plugin are marked with:
- **Auto Grow**: False
- **Exclude From Diagram Cleanup**: False

Future releases may leverage a some kind of user-customizable defaults, provided loading it does not introduce noticeable latency.


## Installation

> **Note:** This plugin is not yet available on VIPM. Installation is currently manual.

Copy the Quick Drop VI (`Enclose in Structure.vi`) and its private subfolder (`_Enclose in Structure`) to:

```
C:\Program Files\National Instruments\LabVIEW 20XX\resource\dialog\QuickDrop\plugins
```

To uninstall, delete the VI and private folder from that location.

> VIPM will automate this (and other necessary areas) once the package is released.


## Technical Implementation Details

### Process Overview

1. Insert 4 `Always Copy.vi` nodes at their respective corners of the bounds  
2. Insert `Always Copy.vi` on selected pass through wires.
3. Enclose the selection (including the `Always Copy.vi` instances) in the target structure using `EncloseSelection2`.
4. Delete all `Always Copy.vi` instances.

The `Always Copy.vi` placements at the corners of the bounds are pixel-perfect with the selected elements, ensuring the most compact structure is created.

![Bounds_Of_Always_Copy](img/Bounds_Of_Always_Copy.png)
*Bounds of `Always Copy.vi` placement. Each corner of `Always Copy.vi` matches the corresponding corner of the selected elements' bounds. For wires, if the `Always Copy.vi` would be placed beyond these bounds, its position is adjusted to remain within them.*

### EncloseSelection2 and Type Casting

The plugin uses `TopLevelDiagram:EncloseSelection2` to create structures. Two constraints apply:

1. A reference to an actual structure instance must be wired — a `Class Specifier Constant` alone is not sufficient.
2. The method only accepts classes inheriting from `Structure`. Since `Flat Sequence Structure` and `Timed Sequence Structure` do not inherit from `Structure`, a type cast workaround is required.

![Type_Cast_Structure](img/Type_Cast_Structure.png)
*`EncloseSelection2` accepts a Flat Sequence directly when type-cast to `Structure`.*

Alternatively, one could wrap with a Stacked Sequence first, then convert to a Flat Sequence:

![Convert_To_FlatSeq](img/Convert_To_FlatSeq.png)
*Convert To Flat Sequence.*

Other class hierarchies (e.g. XML Parser classes) may require `Coerce To Type` since `To More Specific` / `To More Generic` are not supported. See [Everything You Need to Know about VI Scripting in LabVIEW](https://forums.ni.com/t5/Community-Documents/Everything-You-Need-to-Know-about-VI-Scripting-in-LabVIEW/ta-p/4428599) (Flat Sequences: slides 44-45, Type Casting refnums: slide 48).

There are additional cases where type casting is necessary — for example, using the VI method `Create from Reference` to create a constant. Both `Source Object Ref` (input) and `New Object Ref` (output) are typed as `Control`, but work with type-cast constants. Ideally, these terminals would use the common parent class (`GObject`).

![Type_Cast_Misc](img/Type_Cast_Misc.png)
*Type Cast: class name at outputs preserved.*

> **Caution:** Type casting is not foolproof. Casting a `ComboBoxConstant` to `ComboBox` to set a property on the constant caused LabVIEW to crash. Similarly, casting `LabVIEWClassConstant` to `LabVIEWClassControl` to access `LabVIEW Class Name` / `Qualified Name` properties — see [Scripting: Obtaining name of LVClass wrapped in DVR](https://forums.ni.com/t5/LabVIEW/Scripting-Obtaining-name-of-LVClass-wrapped-in-DVR/m-p/4473917).

### Structure Positioning and Resizing

`Structure` properties `Position` and `Frame Size` allow repositioning and resizing, but only from the bottom and right sides.

> Context help for the private property `Frame Rectangle` claims it can resize from all sides, but it always returns error 1058 (property not found).

### Structure-Specific Offsets

Different structures require additional space for their UI elements (e.g. the For Loop's count terminal, the Case Structure's selector). Offsets are applied per structure type to ensure the bounds accommodate these elements. The Timed Sequence structure type is particularly sensitive when there is insufficient room to create it.

### SuppressTypeProp

When creating a SubVI from the selection, the `Always Copy.vi` instances may briefly appear on the diagram if it is "large." `SuppressTypeProp` may allow faster execution and prevent this visual artifact. See [this discussion at 25:15](https://youtu.be/P0IvWVIkq4g?si=lbVlY2dQu6qODy5X).

An alternative optimization is using the private Application property `App.DeferDrawing`. The better choice between the two is not yet determined.

### Set and Release Busy

The plugin sets the application busy during execution as a precaution. Since type propagation is suppressed, clicking elsewhere on the block diagram during execution could potentially cause issues.

### Snakey Wires

When determining the bounds of the structure, the plugin only considers the wire portion immediately around the selected non-wire objects. If a wire extends horizontally far beyond the selected items (e.g. a SubVI input that snakes to the top-left of the diagram), only the local segment is enclosed. This prevents the structure from growing to encompass distant wire bends.

### QD VI Organization

Following the convention of other Quick Drop plugins, the VIs are **not** placed in an `.lvlib`. Other QD plugins use no libraries, with support VIs stored in underscore-prefixed subfolders.


## Known Issues

- **For Loop sizing** — The count terminal can cause the structure to be slightly larger than expected.
- **Timed Sequence placement** — This structure type is more sensitive to space constraints during creation.
- **`GObject.Replace` limitations** — Only specific structure-to-structure replacements are natively supported by LabVIEW. The `ctrl+shift+s` shortcut works around this, but the underlying limitation remains.


## Future Ideas

### SubVI Approach

An alternative implementation path using SubVIs:

1. Create SubVI from selection.
2. Replace with SubVI contents.

This would work well for Sequence Structures since `WrapInSeq` supports `SubVI:Inline`.

![Create_SubVI_Icon](img/Create_SubVI_Icon.png)
*Create SubVI icon.*

![WrapInSeq](img/WrapInSeq.png)
*`WrapInSeq`.*

The relevant code is at:
`[LabVIEW 20xx]\resource\plugins\PopupMenus\edit time panel and diagram\Create SubVI from Selected Wires.llb`

That code's process:
1. Inserts `Always Copy.vi` on selected wires.
2. Selects the various `Always Copy.vi`.
3. Makes a SubVI out of them.
4. Removes the various `Always Copy.vi`.

**Consideration:** Inlining an unsaved SubVI may trigger a save prompt.

### NES for Classes and Clusters

When dropping an In Place Element Structure on a pass through wire whose type is a class that the current VI belongs to, the structure could automatically unbundle the class instead of a plain pass-through. Similarly, for cluster-typed wires, it could unbundle the cluster elements. This would match a very common developer workflow.

Reference: Look into the right-click menu `Insert -> In Place Element Structure` for its internals.

### Development Install Script

A script to automate installation: detect the LabVIEW version, copy the VI and private folder to the plugins directory (or remove them for uninstall).


## Resources

- [Quick Drop Enthusiasts](https://forums.ni.com/t5/Quick-Drop-Enthusiasts/bd-p/grp-1251)
- [List of Community Quick Drop Keyboard Shortcuts](https://forums.ni.com/t5/Quick-Drop-Enthusiasts/List-of-Community-Quick-Drop-Keyboard-Shortcuts/td-p/3527206)
- [Just Passing Through](https://dqmh.org/just-passing-through/)
- [Your LabVIEW Code Is a Work of Art... But I Can't Read It — Darren Nattinger, GDevCon N.A. 2024](https://www.youtube.com/watch?v=AHOZ7fiuWCA)
- [An End to Brainless Programming — Darren Nattinger](https://www.youtube.com/watch?v=pS1UBZzKl9k)
- [Add structure support to Insert QD shortcut](https://forums.ni.com/t5/Quick-Drop-Enthusiasts/Add-structure-support-to-Insert-QD-shortcut/m-p/4242321)
- [LabVIEW Idea Exchange: Option to connect wires through a dropped structure](https://forums.ni.com/t5/LabVIEW-Idea-Exchange/Option-to-connect-wires-through-a-dropped-structure-on-the/idi-p/1483814)
- [LabVIEW Idea Exchange: Add structure on bare wires](https://forums.ni.com/t5/LabVIEW-Idea-Exchange/Add-structure-on-bare-wires/idi-p/4467734)
- [How to configure Package Build Spec for Quick Drops](https://forums.ni.com/t5/Quick-Drop-Enthusiasts/How-to-configure-Package-Build-Spec-for-Quick-Drops/td-p/3806926)
- [Everything You Need to Know about VI Scripting in LabVIEW](https://forums.ni.com/t5/Community-Documents/Everything-You-Need-to-Know-about-VI-Scripting-in-LabVIEW/ta-p/4428599)
